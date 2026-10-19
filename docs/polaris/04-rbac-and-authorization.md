# 権限モデルと認可

Polaris の中核機能。Iceberg REST 仕様には認可の規定がないため、この層は完全に Polaris 独自である。

## 全体像

```
リクエスト
   │
   ▼ ① 名前 → 実体の解決 (grant record も同梱で取得)
Resolver / PolarisResolutionManifest
   │
   ▼ ② 「この操作にはどの privilege が要るか」
RbacOperationSemantics: PolarisAuthorizableOperation → {target 権限, secondary 権限}
   │
   ▼ ③ 「principal が activate したロール群は、その権限を持つか」
PolarisAuthorizerImpl.hasTransitivePrivilege
   │   - 解決済みパスを親→子の順に走査
   │   - 各エンティティの grant record を見る
   │   - SUPER_PRIVILEGES で包含関係を判定
   ▼
allow / ForbiddenException
```

## 二段のロール

Polaris の RBAC は 2 段階のロールを持つ。これが「全体管理者」と「カタログ管理者」を分離する仕組みになっている。

```
PRINCIPAL ──[PRINCIPAL_ROLE_USAGE]──▶ PRINCIPAL_ROLE ──[CATALOG_ROLE_USAGE]──▶ CATALOG_ROLE
                                            │                                        │
                                            │ (ROOT 直下の権限)                        │ (カタログ配下の権限)
                                            ▼                                        ▼
                                    CATALOG_CREATE 等                    TABLE_READ_DATA 等
```

- **`PRINCIPAL_ROLE`** はサービス全体のスコープ。`ROOT` エンティティ (`root_container`) や `CATALOG` そのものへの権限を受け取る
- **`CATALOG_ROLE`** はカタログ内のスコープ。カタログ配下の namespace / table / view / policy への権限を受け取る
- 両者は `CATALOG_ROLE_USAGE` という privilege によって接続される。
  つまり「principal role にカタログロールを付与する」も grant record 1 本で表現される

principal が「どのロールを activate するか」は認証時に決まる ([05](05-authentication.md))。
リクエストごとに一部のロールだけを有効化することもできる。

## Privilege (`polaris-core/.../core/entity/PolarisPrivilege.java`)

102 種類。各エントリは `(code, securableType, securableSubTypes, granteeType)` を持つ。

```java
TABLE_READ_DATA(
    20,
    PolarisEntityType.TABLE_LIKE,                                   // securable の型
    List.of(PolarisEntitySubType.ICEBERG_TABLE,                     // securable のサブタイプ
            PolarisEntitySubType.GENERIC_TABLE),
    PolarisEntityType.CATALOG_ROLE),                                // grantee の型
```

### カテゴリ別の概観

| カテゴリ | 例 | securable |
|---------|---|-----------|
| サービス管理 | `SERVICE_MANAGE_ACCESS` | `ROOT` |
| カタログ CRUD | `CATALOG_CREATE`, `CATALOG_LIST` | `ROOT` |
| カタログ操作 | `CATALOG_DROP`, `CATALOG_MANAGE_ACCESS`, `CATALOG_MANAGE_CONTENT`, `CATALOG_MANAGE_METADATA` | `CATALOG` |
| ロール利用 | `PRINCIPAL_ROLE_USAGE`, `CATALOG_ROLE_USAGE` | `PRINCIPAL_ROLE` / `CATALOG_ROLE` |
| Namespace | `NAMESPACE_{CREATE,DROP,LIST,READ_PROPERTIES,WRITE_PROPERTIES,FULL_METADATA}` | `NAMESPACE` |
| Table | `TABLE_{CREATE,DROP,LIST,READ_PROPERTIES,WRITE_PROPERTIES,READ_DATA,WRITE_DATA,FULL_METADATA}` | `NAMESPACE` (CREATE のみ) / `TABLE_LIKE` |
| View | `VIEW_*` (Table と対称) | 同上 |
| Principal / Role 管理 | `PRINCIPAL_{CREATE,DROP,LIST,...}`, `PRINCIPAL_ROTATE_CREDENTIALS` | `ROOT` / `PRINCIPAL` |
| Grant 管理 | `*_MANAGE_GRANTS_ON_SECURABLE`, `*_MANAGE_GRANTS_FOR_GRANTEE`, `*_LIST_GRANTS` | 各種 |
| Policy | `POLICY_{CREATE,READ,WRITE,DROP,LIST,ATTACH,DETACH}`, `{CATALOG,NAMESPACE,TABLE}_{ATTACH,DETACH}_POLICY` | `POLICY` / `NAMESPACE` 等 |
| **細粒度テーブル更新** (code 85-102) | `TABLE_ADD_SCHEMA`, `TABLE_ADD_SNAPSHOT`, `TABLE_SET_SNAPSHOT_REF`, `TABLE_REMOVE_SNAPSHOTS`, `TABLE_SET_LOCATION`, `TABLE_MANAGE_STRUCTURE`, ... | `TABLE_LIKE` |

最後のカテゴリが特徴的。Iceberg の `UpdateTableRequest` に含まれる `MetadataUpdate` の種類ごとに privilege が定義されており、
「スナップショットの追加は許すがスキーマ変更は許さない」といった制御ができる。
feature config `ENABLE_FINE_GRAINED_UPDATE_TABLE_PRIVILEGES` で有効化する。
無効時は `UPDATE_TABLE` が一律 `TABLE_WRITE_PROPERTIES` を要求する。

### `TABLE_READ_DATA` / `TABLE_WRITE_DATA` の意味

これらは「メタデータを読む」ではなく「**データファイルへアクセスするための認証情報を払い出してよい**」を意味する。
`LOAD_TABLE_WITH_READ_DELEGATION` が `TABLE_READ_DATA` を要求するのはそのため ([07](07-storage-and-credential-vending.md))。

`TABLE_READ_PROPERTIES` (メタデータのみ) と `TABLE_READ_DATA` (+ 認証情報) が分かれている点が、
Polaris の credential vending 設計の要になっている。

## 操作 → 必要権限のマッピング (`RbacOperationSemantics.java`)

`PolarisAuthorizableOperation` (認可対象の操作、REST エンドポイントより粒度が細かい) から、
必要な privilege 集合を引く `EnumMap`。

```java
record RbacOperationSemantics(
    Set<PolarisPrivilege> targetPrivileges,      // 主対象に必要な権限 (必須・非空)
    Set<PolarisPrivilege> secondaryPrivileges,   // 副対象に必要な権限 (省略可)
    ResolvedPathRooting rooting)                 // ROOT を解決パスの先頭に足すか
```

登録例:

```java
register(LOAD_TABLE,                       TABLE_READ_PROPERTIES);
register(LOAD_TABLE_WITH_READ_DELEGATION,  TABLE_READ_DATA);
register(LOAD_TABLE_WITH_WRITE_DELEGATION, TABLE_WRITE_DATA);
register(DROP_TABLE_WITH_PURGE,            EnumSet.of(TABLE_DROP, TABLE_WRITE_DATA));
register(RENAME_TABLE,                     EnumSet.of(TABLE_DROP),          // target = 移動元
                                           EnumSet.of(TABLE_LIST, TABLE_CREATE));  // secondary = 移動先
register(COMMIT_TRANSACTION,               EnumSet.of(TABLE_WRITE_PROPERTIES, TABLE_CREATE));
```

**target と secondary の分離**が rename / policy attach / grant 操作を素直に表現している。
rename は「移動元に対する DROP」と「移動先に対する CREATE + LIST」の両方を要求する。

`ResolvedPathRooting` は `ROOT` (root container を解決パスの先頭に含める = ROOT 上の grant が効く) と
`CATALOG` (含めない) の 2 値。既定は `ROOT`。

## 認可の意図 (`AuthorizationIntent`)

操作の形によって「何を target とし何を secondary とするか」が違うため、意図をクラスで表現する:

| Intent | target | secondary |
|--------|--------|-----------|
| `TargetlessAuthorizationIntent` | (root container) | - |
| `SingleTargetAuthorizationIntent` | 対象 1 個 | - |
| `RenameAuthorizationIntent` | `from` | `to` |
| `PolicyAttachmentAuthorizationIntent` | policy | attachedTo |
| `RoleAssignmentAuthorizationIntent` | role | assignee |
| `PrivilegeGrantAuthorizationIntent` | grantTarget | grantee |
| `RootPrivilegeGrantAuthorizationIntent` | root container | grantee |

`AuthorizationRequest` は複数の intent を持てる。`PolarisAuthorizer.authorize()` の契約は
「**1 リクエスト内の intent は AND 結合し、最初の deny で短絡してよい**」。

## Resolver — 名前から実体へ

`polaris-core/.../core/persistence/resolver/Resolver.java` (1,175 行)。認可の前提となる「解決」を行う。

REST リクエストは `ns1.ns2.table` のような**名前のパス**で来るが、認可判定には
「パス上の各エンティティの実体 + それぞれの grant record」が必要になる。それを一括で取ってくるのが Resolver。

### 解決するもの

| 対象 | 用途 |
|------|------|
| 呼び出し元 principal | 誰か |
| principal role 群 | activate されたロール |
| 参照カタログ | カタログスコープの起点 |
| **activate された catalog role 群** | principal role → catalog role の `CATALOG_ROLE_USAGE` grant を辿って求める |
| 名前パス群 (`ResolverPath`) | namespace / table / view / policy への実体パス |
| 名前指定エンティティ群 | トップレベル (principal, catalog 等) |

結果は `ResolvedPolarisEntity` (エンティティ + `grantRecordsAsSecurable` + `grantRecordsAsGrantee`) のリストとして、
`PolarisResolvedPathWrapper` (パス全体) にまとめられる。

### 多段パス (multi-pass) 解決

エンティティキャッシュ ([03](03-persistence.md)) を使うため、「キャッシュから読んだ値がバックエンドで既に変わっている」可能性がある。
Resolver は 1 回の解決を複数パスで行う (`resolveWithPlan` → `runResolvePass` のループ):

各パスで、対象エンティティを 3 群に分ける:

1. **drop / purge 済み** → エラーを返す
2. **キャッシュヒットしたが未検証** → entity id / 名前 / 親 id が変わっていないか確認。変わっていれば**もう 1 パス走らせる**
3. **バックエンドからリロード済み** → 検証済み。以降のパスでは再検証しない

パス数は通常数回で収束する。無限ループ防止のため上限 1,000 回のガードがある
(コメントいわく「バグでスピンし続けないための保険」)。

### `PolarisResolutionManifest`

Resolver を包む、リクエスト単位のファサード。`CatalogHandler` が使う。

| メソッド | 説明 |
|---------|------|
| `addPath(ResolverPath)` | 解決したいパスを登録 |
| `addPassthroughPath(ResolverPath)` | **passthrough パス**を登録 (後述) |
| `addTopLevelName(name, type, isOptional)` | トップレベルエンティティを登録 |
| `resolveAll()` / `resolveSelections(Set<Resolvable>)` | 解決の実行 (全部 / 一部) |
| `getResolvedPath(key)` / `getPassthroughResolvedPath(key)` | 結果の取得 |
| `getAllActivatedCatalogRoleAndPrincipalRoles()` | 認可判定に使う grantee id 集合 |
| `getResolvedRootContainerEntityAsPath()` | root container |

**passthrough パス**は「解決に失敗しても全体を失敗させない」パス。
たとえばテーブル作成時、まだ存在しないテーブル自体のパスは passthrough として登録され、
存在すれば取得、なければ null になる。認可対象の親 namespace は通常パスとして必須解決される。

`getIsPassthroughFacade()` は参照カタログが「passthrough facade」(フェデレーション先へ素通しするカタログ) かどうかを返し、
その場合はカタログ内パス解決の厳密さを緩める ([09](09-federation-and-extensions.md))。

## 認可判定 (`PolarisAuthorizerImpl.java`, 1,130 行)

### super privilege

`SUPER_PRIVILEGES` は「desired privilege → それを包含する granted privilege 群」の `SetMultimap`。
`matchesOrIsSubsumedBy(desired, granted)` が完全一致か包含関係を判定する。

```java
SUPER_PRIVILEGES.putAll(TABLE_CREATE,
    List.of(CATALOG_MANAGE_CONTENT, CATALOG_MANAGE_METADATA, TABLE_CREATE, TABLE_FULL_METADATA));

SUPER_PRIVILEGES.putAll(TABLE_READ_PROPERTIES,
    List.of(CATALOG_MANAGE_CONTENT, CATALOG_MANAGE_METADATA, TABLE_FULL_METADATA,
            TABLE_READ_DATA, TABLE_READ_PROPERTIES, TABLE_WRITE_DATA, TABLE_WRITE_PROPERTIES));
```

包含関係は静的なテーブルで、階層構造から自動導出しているわけではない。
`TABLE_READ_DATA` が `TABLE_READ_PROPERTIES` を包含する (データが読めるならプロパティも読める) といった
意味的な関係が手で書き下されている。

`SERVICE_MANAGE_ACCESS` は自分自身のみを包含するように見えるが、実際にはほぼ全 privilege のエントリに含まれるため
実質的な全権となる。

### 推移的権限チェック (`hasTransitivePrivilege`)

```java
for (ResolvedPolarisEntity e : resolvedPath.getResolvedFullPath()) {   // 親 → 子の順
  for (PolarisGrantRecord g : e.getGrantRecordsAsSecurable()) {
    if (matchesOrIsSubsumedBy(desiredPrivilege, PolarisPrivilege.fromCode(g.getPrivilegeCode()))
        && activatedGranteeIds.contains(g.getGranteeId())) {
      return true;
    }
  }
}
return false;
```

**解決済みパス上のどこか 1 箇所に権限があれば許可される**、というのが Polaris の「推移的」の意味。
`ROOT → CATALOG → NAMESPACE(ns1) → NAMESPACE(ns2) → TABLE` というパスに対して、
カタログレベルで `CATALOG_MANAGE_CONTENT` を持っていれば、その配下の全テーブルの操作が通る。

走査を**親から**始めるのは「権限は階層のできるだけ上で管理されるのが一般的なので、早く打ち切れる」という
コメント通りの最適化。

なお、この設計には **deny がない**。grant のみで、明示的な拒否は表現できない。

### 失敗時の情報

`findMissingPrivileges()` が「不足している privilege と、それをチェックした対象パス」のリストを返す。
サーバーログには `TABLE_CREATE on NAMESPACE 'ns1'` のように具体的に出るが、
**クライアントに返すメッセージは意図的に汎用的**にしてある (情報漏洩を避けるため)。

## 認可器の差し替え

`PolarisAuthorizer` は `PolarisAuthorizerFactory` 経由で差し替え可能。設定は `polaris.authorization.type`。

| 実装 | モジュール | 説明 |
|------|-----------|------|
| `PolarisAuthorizerImpl` | `polaris-core` | 既定。上記の RBAC |
| `OpaPolarisAuthorizer` | `extensions/auth/opa` | Open Policy Agent に判定を委譲 |
| `RangerPolarisAuthorizer` | `extensions/auth/ranger` | Apache Ranger に判定を委譲 |

インターフェースが 2 段構えになっているのがこの差し替えを支えている:

```java
void resolveAuthorizationInputs(AuthorizationState, AuthorizationRequest);  // 解決フェーズ
AuthorizationDecision authorize(AuthorizationState, AuthorizationRequest);  // 判定フェーズ
```

RBAC 実装は解決フェーズで `resolutionManifest.resolveAll()` を呼ぶが、
外部エンジンに委譲する実装は「principal と対象パスだけ」など必要最小限を解決すればよい
(インターフェースのコメントが明示的にそう述べている)。判定に必要な情報量が実装によって違うためのフェーズ分けである。

### OPA 拡張

`extensions/auth/opa/` は OPA へ送る JSON の構造を `opa-input-schema.json` + `SCHEMA.md` として明文化している。
`model/` に `ResourceEntity`, `Context` などの入力 DTO があり、
`token/` に OPA への認証トークン供給 (`FileBearerTokenProvider` / `StaticBearerTokenProvider`) がある。
`OpaProductionReadinessChecks` で「本番設定として妥当か」の起動時チェックも行う ([08](08-runtime-quarkus.md))。

### Ranger 拡張

`RangerPolarisOperationSemantics` が `RbacOperationSemantics` に相当する「操作 → Ranger のアクセス種別」写像を持つ。
Ranger のポリシー評価に GraalVM の JS エンジンが必要なため、`graalvm-polyglot-js` に依存している。
