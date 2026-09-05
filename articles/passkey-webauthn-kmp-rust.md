---
title: "KMPアプリにパスキー（WebAuthn）ログインを実装するには？"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["KotlinMultiplatform", "Rust", "WebAuthn", "Android", "iOS"]
published: false
---

# はじめに

個人で作っている猫育成ゲームは、最初はゲストで遊べて、あとからアカウントにアップグレードできる作りにしています。そのアップグレードの手段をパスキー（WebAuthn）にしました。パスワードを自分で持ちたくなかったのが理由です。

サーバーはRust（Axum）で[webauthn-rs](https://github.com/kanidm/webauthn-rs) 0.5、クライアントはKMPでAndroidとiOSそれぞれのOSのAPIを叩いています。実装するときに調べたことをめもっとく。

<!-- TODO: パスキーにする前に他の認証（メール+パスワード、ソーシャルログイン）を検討したか、なぜやめたかを本人が加筆 -->

# 全体の流れ

登録もログインも「begin でチャレンジをもらう → OSの生体認証 → complete で検証」の2往復です。

```
登録:
  POST /api/auth/passkey/register/begin      (JWT必須) → challenge_id + options
  → OSのパスキーAPIに options を渡して指紋/Face ID
  POST /api/auth/passkey/register/complete   (JWT必須) → token + refresh_token

ログイン:
  POST /api/auth/passkey/authenticate/begin      (認証不要) → challenge_id + options
  → OSのパスキーAPIに options を渡して指紋/Face ID
  POST /api/auth/passkey/authenticate/complete   (認証不要) → token + refresh_token
```

登録側だけJWTが要るのは、ゲストユーザーが自分のアカウントにパスキーを足す、という流れだからです。

# サーバー（Rust + webauthn-rs）

## チャレンジをDBに置く

`webauthn-rs`の`start_passkey_registration`が返す状態（`PasskeyRegistration`）は、`complete`のときにもう一度必要になります。Lambdaで動かしていてインメモリに持てないので、JSONにしてDBに入れることにしました。

```rust:passkey.rs
let (ccr, passkey_registration) = webauthn
    .start_passkey_registration(user_id, &email, &display_name, exclude_credentials)?;

let state_json = serde_json::to_string(&passkey_registration)?;

sqlx::query(
    "INSERT INTO webauthn_challenges (user_id, challenge_type, state_json, expires_at) \
     VALUES ($1, $2, $3, NOW() + INTERVAL '5 minutes') RETURNING id",
)
```

この状態をシリアライズするには、`webauthn-rs`のfeatureに`danger-allow-state-serialisation`を付ける必要があります。名前のとおり「危険」と言われているやつなので、有効期限を5分にして、使ったら消す運用にしています。

```toml:Cargo.toml
webauthn-rs = { version = "0.5", features = ["danger-allow-state-serialisation", "conditional-ui"] }
```

取り出すときは`DELETE ... RETURNING`にして、1回しか使えないようにしています。

```rust:passkey.rs
"DELETE FROM webauthn_challenges \
 WHERE id = $1 AND challenge_type = 'registration' AND expires_at > NOW() AND user_id = $2 \
 RETURNING user_id, state_json",
```

期限切れの掃除は別で`DELETE FROM webauthn_challenges WHERE expires_at < NOW()`を流しています。

## ログインはdiscoverable credentials

ログインのほうは「メールアドレスを入れてから指紋」ではなく、いきなり指紋でユーザーを特定したかったので、discoverable credentials（`residentKey: required`）にしています。

```rust:passkey.rs
let (rcr, passkey_authentication) = webauthn.start_discoverable_authentication()?;
```

完了側は`finish_discoverable_authentication`に、そのユーザーが持っている鍵を渡して検証します。

保存しているのは`credential_id`、シリアライズした公開鍵、署名カウンタ（クローン検出用）、`backup_eligible` / `backup_state` あたりです。

# クライアント（KMP）

共通側のインターフェースはこれだけです。サーバーが返したJSONをそのままOSに渡して、OSが返したJSONをそのままサーバーに送るので、間で構造を持つ必要がありません。

```kotlin:PasskeyManager.kt
interface PasskeyManager {
    suspend fun createPasskey(optionsJson: String): String
    suspend fun getPasskey(optionsJson: String): String
}
```

## Androidは短い

`androidx.credentials`の`CredentialManager`がJSON文字列をそのまま受け取ってくれるので、実装は数行で終わります。

```kotlin:AndroidPasskeyManager.kt
class AndroidPasskeyManager(
    private val context: Context,
) : PasskeyManager {

    private val credentialManager = CredentialManager.create(context)

    override suspend fun createPasskey(optionsJson: String): String {
        val request = CreatePublicKeyCredentialRequest(optionsJson)
        val result = credentialManager.createCredential(context, request)
        return (result as CreatePublicKeyCredentialResponse).registrationResponseJson
    }

    override suspend fun getPasskey(optionsJson: String): String {
        val option = GetPublicKeyCredentialOption(optionsJson)
        val request = GetCredentialRequest(listOf(option))
        val result = credentialManager.getCredential(context, request)
        val credential = result.credential as PublicKeyCredential
        return credential.authenticationResponseJson
    }
}
```

## iOSは自分でJSONをほどく

iOSの`AuthenticationServices`はJSONを受け取ってくれないので、`NSJSONSerialization`でパースして、必要な値を取り出して、base64urlをデコードして`NSData`にして…というのを自分で書くことになりました。

```kotlin:IosPasskeyManager.kt
// Server wraps options in {"publicKey": {...}} — unwrap if present
val publicKey = json.objectForKey("publicKey") as? NSDictionary ?: json

val rpDict = publicKey.objectForKey("rp") as? NSDictionary
val rpId = rpDict?.objectForKey("id") as? String ?: ""
val userDict = publicKey.objectForKey("user") as? NSDictionary
val userIdBase64 = userDict?.objectForKey("id") as? String ?: ""
val userName = userDict?.objectForKey("name") as? String ?: ""
val challengeBase64 = publicKey.objectForKey("challenge") as? String ?: ""

val challengeData = base64UrlDecode(challengeBase64) ?: ...
val userIdData = base64UrlDecode(userIdBase64) ?: ...

val provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
    relyingPartyIdentifier = rpId,
)
val registrationRequest = provider.createCredentialRegistrationRequestWithChallenge(
    challenge = challengeData,
    name = userName,
    userID = userIdData,
)
```

`ASAuthorizationController`はdelegateで結果が返ってくるので、`suspendCancellableCoroutine`で`suspend`関数にします。

ここで1つハマりどころがあって、`ASAuthorizationController.delegate`は弱参照なので、delegateをローカル変数で持っているとGCされて何も返ってこなくなります。インスタンスのフィールドで強参照を持つようにしました。

```kotlin:IosPasskeyManager.kt
// ASAuthorizationController.delegate は弱参照のため、
// GC されないようインスタンスで強参照を保持する
private var activeDelegate: NSObject? = null
private var activeController: ASAuthorizationController? = null
```

<!-- TODO: この弱参照の問題にどうやって気づいたか（何も起きなくて調べた？）を本人が加筆 -->

# ドメインの関連付けを忘れずに

パスキーはRP ID（ドメイン）とアプリを紐付けないと動きません。`.well-known`の2ファイルをドメイン側で配信します。

- Android: `https://example.com/.well-known/assetlinks.json` に、パッケージ名と署名証明書のSHA-256フィンガープリントを書く
- iOS: `https://example.com/.well-known/apple-app-site-association` に、`<TeamID>.<BundleID>` を書く

フィンガープリントはこれで取れます。

```bash
# release 用
keytool -list -v -keystore <your-release>.jks | grep SHA256

# debug 用
keytool -list -v -keystore ~/.android/debug.keystore -storepass android | grep SHA256
```

Android は Production / Staging / Dev でapplicationIdを分けているので、3つとも書いておく必要がありました。iOS側は Xcode の Associated Domains に `webcredentials:example.com` を追加します。

# おわりに

サーバーはwebauthn-rsが検証をやってくれるので、自分が書くのは「チャレンジをどこに置くか」だけでした。クライアントはAndroidが拍子抜けするほど短くて、iOSはJSONをほどくところを全部自分で書く、という差がおもしろかったです。

パスキーだと端末をなくしたときに詰むので、メールで認証コードを送る復旧フローも別で作りました。そっちも書けたら書こうと思います。

<!-- TODO: 実際に使われているか（登録率など言える範囲で）を本人が加筆 -->

# 参考

- https://github.com/kanidm/webauthn-rs
- https://developer.android.com/identity/sign-in/credential-manager
- https://developer.apple.com/documentation/authenticationservices/public-private-key-authentication
- https://www.w3.org/TR/webauthn-3/
