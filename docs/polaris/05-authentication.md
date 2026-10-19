# 認証と Realm 解決

認可 ([04](04-rbac-and-authorization.md)) が「何ができるか」なら、こちらは「誰か」と「どのテナントか」を決める層。
Quarkus Security の仕組み (`HttpAuthenticationMechanism` / `SecurityIdentityAugmentor`) の上に構築されている。

## リクエストのフィルタ順序 (`runtime/service/.../service/config/FilterPriorities.java`)

```java
REQUEST_ID_FILTER     = Priorities.AUTHENTICATION - 101   // X-Request-ID の採番
REALM_CONTEXT_FILTER  = REQUEST_ID_FILTER + 1             // Realm の解決 ← 認証より前
MDC_FILTER            = REALM_CONTEXT_FILTER + 1          // ログ MDC への注入
TRACING_FILTER        = REALM_CONTEXT_FILTER + 2          // OpenTelemetry
IDEMPOTENCY_KEY_FILTER= REALM_CONTEXT_FILTER + 3          // 冪等キー
RATE_LIMITER_FILTER   = Priorities.USER                   // レート制限 (認証の後)
```

**Realm 解決が認証より前**なのが要点。認証設定 (どの IdP を使うか、内部トークンか外部トークンか) 自体が
realm ごとに違いうるため、先に realm が決まっていないと認証方式を選べない。

## Realm 解決 (`runtime/service/.../service/context/`)

`RealmContextResolver` が `(requestURL, method, path, headers)` から realm 識別子を返す。
既定実装 `DefaultRealmContextResolver`:

```java
String realm = headers.apply(configuration.headerName());   // 既定: "Polaris-Realm"
if (realm != null) {
  if (!configuration.realms().contains(realm)) throw ...;   // 許可リスト外は拒否
} else {
  if (configuration.requireHeader()) throw ...;
  realm = configuration.defaultRealm();                     // 既定 realm にフォールバック
}
```

設定 (`polaris.realm-context.*`):

| キー | 既定 | 説明 |
|-----|------|------|
| `type` | `default` | リゾルバ実装の選択 (`@Identifier`) |
| `realms` | `POLARIS` | **許可される realm の列挙**。最初の要素が既定 realm |
| `header-name` | `Polaris-Realm` | realm を運ぶヘッダ名 |
| `require-header` | `false` | ヘッダ必須にするか |

realm が事前に列挙された許可リストで管理されている点に注意。任意の文字列で新しい realm が
勝手に生えることはなく、realm の追加は設定変更 + `bootstrap` の実行を伴う ([08](08-runtime-quarkus.md))。

戻り値が `CompletionStage<RealmContext>` なのは、realm 解決に外部問い合わせを伴う実装 (テナント DB を引く等) を
想定した非同期インターフェースだから。既定実装は即時完了する。

## 3 つの認証モード (`AuthenticationType`)

| モード | 内部トークンエンドポイント | 説明 |
|-------|------------------------|------|
| `INTERNAL` | 有効 | Polaris が唯一の IdP |
| `EXTERNAL` | **無効** | 外部 IdP (Keycloak 等) に委譲 |
| `MIXED` | 有効 | 内部トークンを**先に**試し、Polaris 発行でなければ外部 IdP に渡す |

設定は `polaris.authentication.type`。`AuthenticationConfiguration` は
`Map<String, AuthenticationRealmConfiguration> realms()` の形で **realm ごとに上書き可能**。
キー `<default>` が全 realm 共通の既定値になる。

## 内部認証 (`service/auth/internal/`)

### 認証メカニズム

`InternalAuthenticationMechanism` は Quarkus の `HttpAuthenticationMechanism` 実装。
優先度を `DEFAULT_PRIORITY + 100` にして、OIDC メカニズム (`DEFAULT_PRIORITY + 1`) より**先に**試されるようにしている。

```
type == EXTERNAL          → 何もしない (null を返し、OIDC に譲る)
Authorization ヘッダ無し   → null
Bearer 以外               → null
tokenBroker.verify() 失敗 → MIXED なら null (OIDC に譲る) / INTERNAL なら AuthenticationFailedException
```

`MIXED` の「先に内部を試し、失敗したら外部へ」という挙動がこの分岐に表れている。

### `TokenBroker`

```java
boolean supportsGrantType(String grantType);
boolean supportsRequestedTokenType(TokenType tokenType);
TokenResponse generateFromClientSecrets(...);   // client_credentials フロー
TokenResponse generateFromToken(...);           // token-exchange フロー
PolarisCredential verify(String token);         // 検証
```

実装は `JWTBroker` (Auth0 java-jwt)。署名方式は 2 つの `TokenBrokerFactory` で選ぶ:

| ファクトリ | 設定値 | 鍵 |
|-----------|-------|---|
| `RSAKeyPairJWTBrokerFactory` | `rsa-key-pair` (既定) | RSA 鍵ペア。`LocalRSAKeyProvider` / `PemUtils` で PEM を読む |
| `SymmetricKeyJWTBrokerFactory` | `symmetric-key` | 共有秘密鍵 (HMAC) |

JWT のクレーム:

| クレーム | 内容 |
|---------|------|
| `iss` | `ISSUER_KEY` (固定) |
| `sub` | principal 名 |
| `active` | `true` (検証時に必須条件として要求される) |
| `principalId` | エンティティ id |
| `client_id` | クライアント ID |
| `scope` | `PRINCIPAL_ROLE:...` の空白区切り |

`AlgorithmAndVerifier` が検証器をファクトリ側で 1 度だけ構築し、リクエストスコープのブローカー間で共有する
(検証器の構築コストを毎リクエスト払わないため)。

### OAuth2 トークンエンドポイント

`DefaultOAuth2ApiService` が `/v1/oauth/tokens` を実装する (`EXTERNAL` モードでは `DisabledOAuth2ApiService` に差し替わる)。

`TokenRequestValidator` が受け付ける grant type は 2 つだけ:

- `client_credentials`
- `urn:ietf:params:oauth:grant-type:token-exchange`

**`scope` は Iceberg REST 仕様上は任意だが Polaris は必須にしている**。
形式は `PRINCIPAL_ROLE:NAME PRINCIPAL_ROLE:NAME2 ...` で、ここで指定したロールだけが activate される。

### 資格情報の保存

`principal_authentication_data` テーブル ([03](03-persistence.md)) に
`main_secret_hash` / `secondary_secret_hash` / `secret_salt` として保存。平文は保持しない。
secondary があるのはローテーション中に新旧両方を有効にするため。

関連 privilege: `PRINCIPAL_ROTATE_CREDENTIALS` (本人によるローテーション) と
`PRINCIPAL_RESET_CREDENTIALS` (管理者によるリセット、feature config `ENABLE_CREDENTIAL_RESET` で制御)。

## 外部認証 (`service/auth/external/`)

Quarkus OIDC 拡張を使う。Polaris 側が足すのは「OIDC の JWT を Polaris のエンティティに対応づける」部分。

### マルチテナント OIDC

`OidcConfiguration` (`polaris.oidc.*`) は `Map<String, OidcTenantConfiguration> tenants()` を持ち、
複数の OIDC テナント (= 複数の IdP) を同時に構成できる。キー `<default>` が既定。

`OidcTenantResolver` (既定 `DefaultOidcTenantResolver`) が、受け取った identity からどのテナント設定を使うか決める。
`OidcTenantResolvingAugmentor` が `SecurityIdentity` にその結果を付与する。

### クレームからのマッピング

`OidcPolarisCredentialAugmentor` が OIDC の `JsonWebToken` から `PolarisCredential` を作る。実際の写像は 2 つの SPI:

| SPI | 既定実装 | 役割 |
|-----|---------|------|
| `PrincipalMapper` | `DefaultPrincipalMapper` | JWT のクレームから principal **id** または **名前**を取り出す |
| `PrincipalRolesMapper` | `DefaultPrincipalRolesMapper` | JWT のクレームから principal role 群を取り出す |

`ClaimsLocator` が「クレームパス」(`realm_access/roles` のようなネストしたパス) を解決するヘルパ。
設定でパスを指定できるので、IdP 側のクレーム構造に合わせられる。

設定は `polaris.oidc.principal-mapper.type` / `polaris.oidc.principal-roles-mapper.type` で
`@Identifier` による差し替えが可能。

## 認証の共通出口: `Authenticator`

内部・外部どちらの経路も、最終的には Quarkus の `SecurityIdentity` になる。
そこから Polaris のドメインオブジェクトへ変換するのが `Authenticator`:

```java
PolarisPrincipal authenticate(SecurityIdentity identity) throws AuthenticationFailedException;
```

`AuthenticatingAugmentor` (`SecurityIdentityAugmentor`, priority 1000) が
匿名でない identity に対してこれを呼び、結果の `PolarisPrincipal` とロール・属性を
`QuarkusSecurityIdentity` に詰め直す。

### `DefaultAuthenticator`

```
PolarisCredential を取り出す
  ↓
principal エンティティを解決 (id または名前で永続化層を引く)
  ↓
principal role 群を解決
  ↓
PolarisPrincipal.of(name, attributes, roles)
```

ロール解決には 2 つの特殊な文字列がある:

| 値 | 意味 |
|---|------|
| `PRINCIPAL_ROLE:ALL` | **擬似ロール**。DB には存在しない。「この principal に付与された全ロールを activate する」 |
| `PRINCIPAL_ROLE:<name>` | 指定ロールのみを activate。principal の grant から解決される |

このプレフィックスに一致しないロールは**無視される**。
外部 IdP のロール名をそのまま使いたい場合は `PrincipalRolesMapper` で変換しておく必要がある、という設計。

「リクエストごとに一部のロールだけを有効化する」ことができる点は、
最小権限で操作するための仕組みになっている。activate されたロールの集合が
そのまま認可判定の `activatedGranteeIds` になる ([04](04-rbac-and-authorization.md))。

## `PolarisPrincipal` (`polaris-core/.../core/auth/PolarisPrincipal.java`)

`java.security.Principal` を実装する不変オブジェクト。名前、属性 (`Map<String, Object>`)、
activate されたロール名の集合を持つ。`polaris-core` 側にあるので、認可ロジックは Quarkus を知らずに済む。

## Realm と認証の関係のまとめ

```
HTTP リクエスト
  │  Polaris-Realm: prod
  ▼
RealmContextFilter → RealmContext("prod")
  │
  ▼ realm ごとの認証設定を引く
AuthenticationConfiguration.forRealm("prod") → type=MIXED, token-broker=rsa-key-pair
  │
  ├─▶ InternalAuthenticationMechanism (priority +100): JWTBroker.verify()
  │       成功 → SecurityIdentity (PolarisCredential 付き)
  │       失敗 (MIXED) → 次へ
  └─▶ Quarkus OIDC (priority +1): 外部 IdP へ検証
          → OidcPolarisCredentialAugmentor → PrincipalMapper / PrincipalRolesMapper
  │
  ▼ AuthenticatingAugmentor (priority 1000)
Authenticator.authenticate() → PolarisPrincipal(name, attributes, activatedRoles)
  │
  ▼
CatalogHandler / PolarisAuthorizer  ([04])
```
