# 型システムとスキーマ

## 型階層

`api/src/main/java/org/apache/iceberg/types/`

```
Type
 +-- PrimitiveType
 |    +-- BooleanType, IntegerType, LongType, FloatType, DoubleType
 |    +-- DateType, TimeType, TimestampType, TimestampNanoType
 |    +-- StringType, UUIDType, FixedType(length), BinaryType
 |    +-- DecimalType(precision, scale)
 |    +-- GeometryType(CRS), GeographyType(CRS, algorithm)
 |    +-- UnknownType
 +-- NestedType
 |    +-- StructType (ordered list of NestedField)
 |    +-- ListType (element field with unique ID)
 |    +-- MapType (key field + value field, each with unique ID)
 +-- VariantType
```

## NestedField

スキーマの基本構成要素。各フィールドが持つ属性:
- `id` (int): **グローバルに一意**。スキーマ進化の鍵
- `name` (String): フィールド名 (リネーム可能)
- `type` (Type): データ型
- `isOptional` (boolean): nullable かどうか
- `doc` (String): ドキュメント
- `initialDefault` / `writeDefault` (Literal): デフォルト値 (v3+)

## TypeID → Java マッピング

| TypeID | Java Class |
|--------|-----------|
| BOOLEAN | Boolean |
| INTEGER | Integer |
| LONG | Long |
| FLOAT | Float |
| DOUBLE | Double |
| DATE | Integer (epoch days) |
| TIME | Long (micros) |
| TIMESTAMP | Long (micros) |
| TIMESTAMP_NANO | Long (nanos) |
| STRING | CharSequence |
| UUID | java.util.UUID |
| FIXED / BINARY | ByteBuffer |
| DECIMAL | BigDecimal |
| GEOMETRY / GEOGRAPHY | ByteBuffer |
| STRUCT | StructLike |
| LIST | List |
| MAP | Map |
| VARIANT | Variant |

## Schema クラス (`api/.../Schema.java`)

`StructType` をラップし、以下を追加:
- **Schema ID**: テーブルメタデータからの読込時に付与
- **Identifier field IDs**: プライマリキーに似た概念 (required、primitive、float/double 不可)
- **遅延インデックス**: `idToField`, `nameToId`, `idToAccessor` 等をオンデマンド構築
- **カラム射影**: `select(names)` で射影スキーマ作成
- **フォーマットバージョン互換性チェック**: TIMESTAMP_NANO, VARIANT 等は v3+ でのみ使用可

## スキーマ進化 (`UpdateSchema` / `SchemaUpdate`)

| 操作 | 説明 |
|------|------|
| `addColumn(name, type)` | 新カラム追加 (新規 ID 割り当て) |
| `deleteColumn(name)` | カラム削除 |
| `renameColumn(old, new)` | リネーム (ID 保持) |
| `updateColumn(name, newType)` | 型変更 (安全な昇格のみ) |
| `moveFirst/Before/After` | カラム順序変更 |
| `makeColumnOptional/requireColumn` | nullable 変更 |
| `unionByNameWith(newSchema)` | 名前ベースのマージ |
| `setIdentifierFields(names)` | 識別子フィールド設定 |
| `allowIncompatibleChanges()` | 破壊的変更の許可 |

## 型昇格ルール

| ソース型 | 許可される昇格 (v1/v2) | v3 追加 |
|----------|----------------------|---------|
| `unknown` | - | 任意の型 |
| `int` | `long` | - |
| `float` | `double` | - |
| `decimal(P,S)` | `decimal(P',S)` (P' > P) | - |
| `date` | - | `timestamp`, `timestamp_ns` |

パーティションソースフィールドの型昇格は、Transform が昇格後に異なる値を生成する場合ブロックされる。

## カラム ID の設計

- カラム ID は**再利用されない**
- `last-column-id` で最大 ID を追跡、新カラムは常に +1
- ネストフィールド (struct 内、list 要素、map key/value) もすべて一意 ID を持つ
- データファイル内のカラム射影は ID ベース → リネーム・並べ替え後も古いデータを正しく読める
