# ストレージ設定と認証情報の払い出し

Polaris が「単なるメタデータサーバー」以上のものになっている理由がこの層。
クライアント (Spark 等) にオブジェクトストレージの長期鍵を配らず、
**そのテーブルのパスに絞り込んだ短期認証情報**をリクエストのたびに発行する。

これにより、ストレージ側の ACL を触らずに Polaris の RBAC ([04](04-rbac-and-authorization.md)) だけで
データアクセスを制御できる。認可の判断結果が、そのまま IAM ポリシーとして具体化される。

## 全体の流れ

```
クライアント: GET /v1/{prefix}/namespaces/ns/tables/t
              X-Iceberg-Access-Delegation: vended-credentials
  │
  ▼ ① 認可: TABLE_READ_DATA を持つか?  ([04])
IcebergCatalogHandler
  │
  ▼ ② ストレージ設定をエンティティ階層から探す
PolarisStorageConfigurationInfo.findStorageConfigFromHierarchy()
      table → namespace → catalog と遡り、最初に見つかったものを使う
  │
  ▼ ③ どのロケーションにどの操作を許すかを組み立てる
List<LocationGrant>  = [(loc, {READ, LIST}), ...]
  │
  ▼ ④ キャッシュを引く / 無ければ発行
StorageCredentialCache.getOrLoad(key)
  │
  ▼ ⑤ クラウド固有の発行
PolarisStorageIntegration.getStorageAccessConfig(grants, refreshEndpoint, context)
      AWS  : STS AssumeRole + 動的生成した IAM ポリシー
      GCP  : IAM Credentials / Workload Identity Federation
      Azure: SAS トークン
  │
  ▼ ⑥ レスポンスの config に詰めて返す
LoadTableResponse.config = { "s3.access-key-id": ..., "s3.session-token": ..., ... }
```

## ストレージ設定 (`polaris-core/.../core/storage/PolarisStorageConfigurationInfo.java`)

抽象クラス。カタログ (または namespace / table) の `internalProperties` に JSON でシリアライズされて保存される。

```java
public abstract List<String> getAllowedLocations();
public abstract String getStorageName();
public abstract StorageType getStorageType();
public abstract String getFileIoImplClassName();
```

### `StorageType`

| 種別 | URI プレフィックス |
|-----|------------------|
| `S3` | `s3://`, `s3a://` |
| `AZURE` | `abfs://`, `abfss://`, `wasb://`, `wasbs://` |
| `GCS` | `gs://` |
| `FILE` | `file://` |

`validatePrefixForStorageType()` が「許可ロケーションの URI スキームが種別と整合しているか」を検証する。
`FILE` は `ALLOW_INSECURE_STORAGE_TYPES` を有効にしない限り使えない。

有効化されている種別は feature config `SUPPORTED_CATALOG_STORAGE_TYPES` (既定 `["S3","GCS","AZURE"]`) で制御する。

### 階層からの探索

```java
static Optional<PolarisEntity> findEntityWithStorageConfigFromHierarchy(...);
static Optional<PolarisStorageConfigurationInfo> findStorageConfigFromHierarchy(...);
static Optional<LocationRestrictions> forEntityPath(realmConfig, resolvedFullPath);
```

解決済みパス ([04](04-rbac-and-authorization.md)) を末端から遡り、
最初にストレージ設定を持つエンティティのものを採用する。
通常はカタログに 1 つ設定するが、namespace やテーブル単位で上書きもできる。

### AWS の設定 (`AwsStorageConfigurationInfo`)

| フィールド | 用途 |
|-----------|------|
| `roleARN` | AssumeRole 先のロール |
| `externalId` | confused deputy 対策の external id |
| `userARN` | Polaris 側の実行主体 ARN |
| `region` / `endpoint` / `endpointInternal` / `stsEndpoint` | エンドポイント (S3 互換ストレージ対応) |
| `pathStyleAccess` | パススタイルアクセス (MinIO 等) |
| `currentKmsKey` / `allowedKmsKeys` | SSE-KMS 用の鍵 |
| `stsUnavailable` / `kmsUnavailable` | STS / KMS が使えない環境向けのフラグ |

`endpoint` と `endpointInternal` が分かれているのは、Polaris 自身がアクセスする際のエンドポイントと
クライアントに教えるエンドポイントが異なる構成 (コンテナ内 DNS など) を扱うため。
`stsUnavailable` は credential vending 自体ができない環境 (S3 互換ストレージ) を想定した逃げ道。

## `LocationGrant` と `PolarisStorageIntegration`

```java
StorageAccessConfig getStorageAccessConfig(
    @NonNull List<LocationGrant> grants,        // (ロケーション集合, アクション集合) の組
    @NonNull Optional<String> refreshEndpoint,  // クライアントが更新に使うエンドポイント
    @NonNull CredentialVendingContext context); // 監査タグ・キャッシュキー用のメタデータ
```

アクションは `PolarisStorageActions` (`READ` / `WRITE` / `LIST` / `DELETE` / `ALL`)。

インターフェースの javadoc が**実装ごとの粒度の差を明記している**点が誠実:

- **AWS / GCP**: grant ごとのアクション分離を守る。`(loc, {WRITE})` は `loc` に read/list を与えない
- **Azure**: SAS トークンがコンテナ (または階層型 ADLS では単一パス) 単位で monolithic なため、
  **prefix ごとのアクション分離を完全には守れない**。アクションフラグは全 grant の和集合になる

### AWS の実装 (`AwsCredentialsStorageIntegration`)

`policyString()` が `LocationGrant` から IAM ポリシーを動的に生成する:

| grant のアクション | 生成される statement |
|------------------|-------------------|
| read | `s3:GetObject`, `s3:GetObjectVersion` — リソースは `arn:aws:s3:::bucket/prefix/*` |
| list | `s3:ListBucket` — リソースはバケット、`s3:prefix` の `StringLike` 条件付き |
| write | `s3:PutObject`, `s3:DeleteObject` |
| (いずれか) | `s3:GetBucketLocation` — バケット単位 |

そのポリシーを `AssumeRole` の `policy` パラメータ (session policy) として渡し、
`durationSeconds` に `STORAGE_CREDENTIAL_DURATION_SECONDS` を指定する。
AssumeRole の結果は元ロールの権限と session policy の**積**になるので、
ロールが広い権限を持っていても、払い出される認証情報はこのポリシーの範囲に閉じる。

**空ポリシーのガード**が入っているのが重要な安全策:

> If the resulting policy would have no statements ..., a single explicit `Deny *` statement is
> emitted so we don't send an empty policy to AWS and inadvertently assume the role with full privileges.

空の session policy を渡すと「制限なし」= 元ロールの全権になってしまうため、
statement が 0 個になる場合は明示的な `Deny *` を出す。

`escapeIamGlobLiteral()` でロケーション中の IAM グロブ特殊文字 (`*`, `?`) をエスケープしている。
これがないと、`*` を含むテーブルパスで意図より広いリソースにマッチしてしまう。

### 監査のためのセッションタグ (`CredentialVendingContext`)

STS のセッションタグとして以下を付与できる。CloudTrail に記録され、
「どの realm のどの principal がどのテーブルのためにこの認証情報を使ったか」を追跡できる:

| タグキー | 内容 |
|---------|------|
| `polaris:realm` | realm 識別子 |
| `polaris:catalog` | カタログ名 |
| `polaris:namespace` | namespace (ドット区切り) |
| `polaris:table` | テーブル名 |
| `polaris:principal` | プリンシパル名 |
| `polaris:roles` | activate されたロール (カンマ区切り・ソート済み) |
| `polaris:trace_id` | OpenTelemetry トレース ID |

`activatedRoles` が `PolarisPrincipal` から都度取り出されず context のフィールドになっているのは、
**キャッシュキーの一部にするため** (javadoc に明記)。
ロールが違えば違う認証情報になるので、キャッシュを共有してはいけない。

`trace_id` は feature config `INCLUDE_TRACE_ID_IN_SESSION_TAGS` で制御される。
これが入ると CloudTrail (認証情報の払い出し) / Polaris のイベント (カタログ操作) /
コンピュートエンジンからのメトリクスレポート ([03](03-persistence.md)) を 1 本のトレースで突き合わせられる。
一方でトレース ID がキャッシュキーに入るのでキャッシュヒット率は落ちる — 既定は無効。

セッション名とセッションタグの構成は `AwsSessionNameBuilder` / `AwsSessionTagsBuilder` /
`SessionNameField` / `SessionTagField` で設定可能。
`AwsRoleSessionNameSanitizer` が AWS の文字種制限に合わせて名前を整形する。

### GCP / Azure

| 実装 | 方式 |
|------|------|
| `GcpCredentialsStorageIntegration` | IAM Credentials API でダウンスコープしたアクセストークン。`GcpFederatedCredentialsExchanger` で Workload Identity Federation にも対応 |
| `AzureCredentialsStorageIntegration` | SAS (Shared Access Signature) トークンを生成。`AzureLocation` がコンテナ/パスを解析 |

GCP には **principal attribution** という仕組みがある
(`GcpAttributionParams`, `GcpAttributionSubjectBuilder`, feature config `GCS_PRINCIPAL_ATTRIBUTION_*`)。
Polaris が署名鍵を持ち、principal を subject とする WIF トークンを作って交換することで、
GCP 側の監査ログに Polaris の principal を残す。

Azure の設定には `AZURE_TIMEOUT_MILLIS` / `AZURE_RETRY_COUNT` / `AZURE_RETRY_DELAY_MILLIS` /
`AZURE_RETRY_JITTER_FACTOR` という専用のリトライ設定がある。

## 認証情報キャッシュ (`storage/cache/StorageCredentialCache.java`)

Caffeine ベース。**expiry がやや凝っている**:

```java
.expireAfter(Expiry.creating((key, entry) ->
    Duration.ofMillis(Math.max(0, Math.min(
        (entry.getExpirationTime() - System.currentTimeMillis()) / 2,   // 残存期間の半分
        entry.maxCacheDurationMs())))))                                 // 設定上限
```

「認証情報の残り有効期間の**半分**」と「設定上限」の小さい方。
残り期間の半分にすることで、キャッシュから取り出した認証情報が必ず有効期間の後半分を残している状態になる。

`maxCacheDurationMs()` は起動時に整合性チェックを行う:

```java
if (STORAGE_CREDENTIAL_CACHE_DURATION_SECONDS >= STORAGE_CREDENTIAL_DURATION_SECONDS)
    throw new IllegalArgumentException(...);
```

キャッシュ期間が認証情報の有効期間以上だと、期限切れの認証情報を配ってしまうので設定エラーにする。

キャッシュキー (`StorageCredentialCacheKey` → `AwsStorageCredentialCacheKey` 等) には
ストレージ設定、grant のロケーションとアクション、`CredentialVendingContext` の各項目が含まれる。

## FileIO

Polaris 自身も `metadata.json` を読み書きするのでストレージにアクセスする。その経路も同じ払い出し機構を通る。

```java
public interface FileIOFactory {
  FileIO loadFileIO(StorageAccessConfig storageAccessConfig,
                    String ioImplClassName,
                    Map<String, String> properties);
}
```

| 実装 | 説明 |
|------|------|
| `DefaultFileIOFactory` | 既定。`StorageAccessConfig` の認証情報を properties に混ぜて Iceberg の FileIO をロード |
| `WasbTranslatingFileIOFactory` | `wasb://` を `abfs://` に書き換えるラッパー (レガシー Azure URI 対応) |

`ExceptionMappingFileIO` がストレージ例外を Polaris の例外にマップするデコレータ。

`StorageAccessConfigProvider` が「テーブル → ストレージ設定 → grant → キャッシュ → `StorageAccessConfig`」の
一連の流れを担う。`LocalIcebergCatalog` はここから FileIO を得るとき、必要最小のアクションだけを要求する
([06](06-iceberg-rest-service.md)):

- `doRefresh` → `{READ, LIST}`
- `doCommit` → `{READ, WRITE, LIST}`

`ALLOW_SPECIFYING_FILE_IO_IMPL` が無効なら、クライアントが `io-impl` を指定して
FileIO 実装を差し替えることはできない (任意クラスのロードを塞ぐ)。

## クライアントへの受け渡し (`StorageAccessProperty`)

払い出した認証情報を Iceberg クライアントが理解するプロパティ名に写す enum:

| クラウド | プロパティ |
|---------|----------|
| AWS | `AWS_KEY_ID`, `AWS_SECRET_KEY`, `AWS_TOKEN`, `AWS_SESSION_TOKEN_EXPIRES_AT_MS`, `AWS_ENDPOINT`, `AWS_PATH_STYLE_ACCESS`, `CLIENT_REGION`, `AWS_REFRESH_CREDENTIALS_ENDPOINT` |
| GCP | `GCS_ACCESS_TOKEN`, `GCS_ACCESS_TOKEN_EXPIRES_AT_MS`, `GCS_REFRESH_CREDENTIALS_ENDPOINT` |
| Azure | `AZURE_SAS_TOKEN_BARE`, `AZURE_SAS_TOKEN_ACCOUNT_HOST`, `AZURE_SAS_TOKEN_ACCOUNT_NAME`, `AZURE_ACCOUNT_NAME`, `AZURE_SAS_TOKEN_EXPIRES_AT_MS`, `AZURE_REFRESH_CREDENTIALS_ENDPOINT` |

`*_EXPIRES_AT_MS` と `*_REFRESH_CREDENTIALS_ENDPOINT` の組がクライアント側の自動更新を可能にする。
クライアントは有効期限が近づいたら `/v1/{prefix}/namespaces/{ns}/tables/{t}/credentials` を叩き直す。

## 関連する feature config

| 設定 | 既定 | 効果 |
|-----|------|------|
| `STORAGE_CREDENTIAL_DURATION_SECONDS` | - | 払い出す認証情報の有効期間 |
| `STORAGE_CREDENTIAL_CACHE_DURATION_SECONDS` | - | キャッシュ上限 (上記より小さいこと) |
| `SKIP_CREDENTIAL_SUBSCOPING_INDIRECTION` | false | サブスコープを省略 (テスト用途) |
| `INCLUDE_PRINCIPAL_NAME_IN_SUBSCOPED_CREDENTIAL` | - | セッション名に principal 名を含める |
| `ALLOW_EXTERNAL_CATALOG_CREDENTIAL_VENDING` | - | 外部カタログでの払い出しを許すか |
| `ALLOW_FEDERATED_CATALOGS_CREDENTIAL_VENDING` | - | フェデレーションカタログでの払い出しを許すか ([09](09-federation-and-extensions.md)) |
| `ALLOW_SETTING_S3_ENDPOINTS` | - | カタログ設定での S3 エンドポイント指定を許すか |
| `RESOLVE_CREDENTIALS_BY_STORAGE_NAME` | - | ストレージ名で認証情報を解決する |
| `ALLOW_UNRESTRICTED_STORAGE_CONFIG_ROLE_CHANGES` | - | ストレージ設定のロール変更を無制限に許すか |

最後の 2 つは特権昇格の経路になりうる設定で、既定は保守的。
`ALLOW_UNRESTRICTED_STORAGE_CONFIG_ROLE_CHANGES` を有効にすると、カタログ設定を更新できる者が
`roleARN` を任意のロールに書き換えて、そのロールの権限で認証情報を払い出せるようになる。
