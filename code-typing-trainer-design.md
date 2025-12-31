# Code Typing Trainer - 設計ドキュメント

## 概要

リポジトリのコードベースから単語を抽出し、タイピング練習の問題セットを生成するツール。
開発者が普段書くコードの語彙を効率的に練習できるようにする。

---

## 要件サマリー

### 機能要件

| 機能 | 説明 |
|------|------|
| リポジトリ解析 | Git リポジトリからソースコードを解析し、識別子を抽出 |
| 正規化 | `camelCase`, `snake_case`, `PascalCase` を同一概念として認識 |
| 差分検知 | コミットハッシュの比較で更新の必要性を検知 |
| 差分更新 | 問題セットを最新のコードベースに追従して更新 |
| 問題セット出力 | Monkeytype 互換形式で出力 |
| 統計管理 | 正規化キーベースで練習統計を集計 |

### 非機能要件

- TypeScript で実装
- CLI フレームワークに gunshi を採用
- SQLite でローカルファーストなデータ永続化
- 言語パーサーはプラグイン形式で拡張可能

---

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          code-typing-trainer                                │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLI Layer (gunshi)                             │
├──────────────┬──────────────┬──────────────┬──────────────┬────────────────┤
│    init      │    scan      │    status    │   export     │    stats       │
│  リポジトリ   │   解析実行    │   更新検知    │  問題出力     │   統計表示     │
│  登録        │              │              │              │               │
└──────────────┴──────────────┴──────────────┴──────────────┴────────────────┘
                                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Core Domain Layer                                │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│     Analyzer        │     Normalizer      │         Exporter                │
│   トークン抽出       │   正規化処理         │      フォーマット変換            │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
                                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Plugin System                                     │
├──────────────┬──────────────┬──────────────┬────────────────────────────────┤
│  TypeScript  │    Python    │     Rust     │           ...                  │
│   Parser     │    Parser    │    Parser    │                                │
└──────────────┴──────────────┴──────────────┴────────────────────────────────┘
                                      │
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Infrastructure Layer                               │
├─────────────────────────────────┬───────────────────────────────────────────┤
│           SQLite                │              Git Integration              │
│     (better-sqlite3)            │              (simple-git)                 │
└─────────────────────────────────┴───────────────────────────────────────────┘
```

---

## データモデル

### ER 図

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐    │
│  │   Repository    │       │     Token       │       │   TypingStats   │    │
│  ├─────────────────┤       ├─────────────────┤       ├─────────────────┤    │
│  │ id (PK)         │──┐    │ id (PK)         │       │ id (PK)         │    │
│  │ name            │  │    │ repo_id (FK)    │──┐    │ repo_id (FK)    │────┤
│  │ path            │  └───▶│ raw             │  │    │ canonical       │    │
│  │ last_commit     │       │ canonical       │◀─┼────│ attempts        │    │
│  │ last_scanned_at │       │ segments (JSON) │  │    │ errors          │    │
│  │ created_at      │       │ casing          │  │    │ avg_time_ms     │    │
│  └─────────────────┘       │ kind            │  │    │ last_practiced  │    │
│                            │ source_file     │  │    └─────────────────┘    │
│                            │ frequency       │  │                           │
│                            │ created_at      │  │    ┌─────────────────┐    │
│                            │ updated_at      │  │    │  VariantStats   │    │
│                            └─────────────────┘  │    ├─────────────────┤    │
│                                                 │    │ id (PK)         │    │
│                                                 │    │ repo_id (FK)    │────┘
│                                                 │    │ raw             │
│                                                 │    │ canonical       │
│                                                 │    │ attempts        │
│                                                 │    │ errors          │
│                                                 │    │ avg_time_ms     │
│                                                 │    └─────────────────┘
└──────────────────────────────────────────────────────────────────────────────┘
```

### テーブル定義

#### Repository

リポジトリの登録情報を管理する。

| カラム | 型 | 説明 |
|--------|------|------|
| id | TEXT (UUID) | 主キー |
| name | TEXT | リポジトリ名（表示用） |
| path | TEXT | ローカルパス |
| last_commit | TEXT | 最後にスキャンしたコミットハッシュ |
| last_scanned_at | TEXT (ISO8601) | 最後にスキャンした日時 |
| created_at | TEXT (ISO8601) | 登録日時 |

#### Token

抽出されたトークン（識別子）を管理する。

| カラム | 型 | 説明 |
|--------|------|------|
| id | TEXT (UUID) | 主キー |
| repo_id | TEXT (FK) | リポジトリID |
| raw | TEXT | 元の表記（例: `getUserProfile`） |
| canonical | TEXT | 正規化キー（例: `get_user_profile`） |
| segments | TEXT (JSON) | 分割された単語（例: `["get", "user", "profile"]`） |
| casing | TEXT | 表記スタイル（camel, snake, pascal, kebab, etc.） |
| kind | TEXT | トークン種別（function, variable, class, etc.） |
| source_file | TEXT | 検出されたファイルパス |
| frequency | INTEGER | 出現回数 |
| created_at | TEXT (ISO8601) | 作成日時 |
| updated_at | TEXT (ISO8601) | 更新日時 |

**インデックス:**
- `(repo_id, canonical)` - 正規化キーでの検索・集計用
- `(repo_id, raw)` - 重複チェック用
- `(repo_id, frequency DESC)` - 頻度順の取得用

#### TypingStats

正規化キー単位での練習統計を管理する。

| カラム | 型 | 説明 |
|--------|------|------|
| id | TEXT (UUID) | 主キー |
| repo_id | TEXT (FK) | リポジトリID |
| canonical | TEXT | 正規化キー |
| attempts | INTEGER | 練習回数 |
| errors | INTEGER | エラー回数 |
| avg_time_ms | REAL | 平均入力時間（ミリ秒） |
| last_practiced | TEXT (ISO8601) | 最後に練習した日時 |

#### VariantStats

表記バリアント単位での練習統計を管理する。

| カラム | 型 | 説明 |
|--------|------|------|
| id | TEXT (UUID) | 主キー |
| repo_id | TEXT (FK) | リポジトリID |
| raw | TEXT | 元の表記 |
| canonical | TEXT | 正規化キー |
| attempts | INTEGER | 練習回数 |
| errors | INTEGER | エラー回数 |
| avg_time_ms | REAL | 平均入力時間（ミリ秒） |

---

## 正規化ロジック

### 入力と出力の例

| 入力 (raw) | 出力 (canonical) | segments | casing |
|------------|------------------|----------|--------|
| `getUserProfile` | `get_user_profile` | `["get", "user", "profile"]` | `camel` |
| `get_user_profile` | `get_user_profile` | `["get", "user", "profile"]` | `snake` |
| `GetUserProfile` | `get_user_profile` | `["get", "user", "profile"]` | `pascal` |
| `get-user-profile` | `get_user_profile` | `["get", "user", "profile"]` | `kebab` |
| `GETUSERPROFILE` | `getuserprofile` | `["getuserprofile"]` | `upper` |
| `MAX_RETRY_COUNT` | `max_retry_count` | `["max", "retry", "count"]` | `screaming_snake` |

### 正規化アルゴリズム

```typescript
interface NormalizeResult {
  canonical: string;
  segments: string[];
  casing: CasingStyle;
}

type CasingStyle = 
  | 'camel'           // getUserProfile
  | 'pascal'          // GetUserProfile
  | 'snake'           // get_user_profile
  | 'screaming_snake' // GET_USER_PROFILE
  | 'kebab'           // get-user-profile
  | 'flat'            // getuserprofile
  | 'unknown';

function normalize(raw: string): NormalizeResult {
  // 1. ケーシングスタイルを検出
  const casing = detectCasing(raw);
  
  // 2. セグメントに分割
  const segments = splitIntoSegments(raw);
  
  // 3. 正規化キーを生成（snake_case に統一）
  const canonical = segments.map(s => s.toLowerCase()).join('_');
  
  return { canonical, segments, casing };
}

function splitIntoSegments(raw: string): string[] {
  // ケースバウンダリで分割
  // getUserProfile -> ['get', 'User', 'Profile']
  // get_user_profile -> ['get', 'user', 'profile']
  // get-user-profile -> ['get', 'user', 'profile']
  
  return raw
    // snake_case, kebab-case の区切り
    .split(/[-_]/)
    // camelCase, PascalCase の区切り
    .flatMap(part => part.split(/(?<=[a-z])(?=[A-Z])/))
    // 連続大文字の区切り（例: XMLParser -> XML, Parser）
    .flatMap(part => part.split(/(?<=[A-Z])(?=[A-Z][a-z])/))
    .filter(s => s.length > 0);
}
```

---

## CLI コマンド設計

### コマンド一覧

```
code-typing-trainer
├── init <path>          # リポジトリを登録
├── scan [repo]          # トークンを抽出・更新
├── status [repo]        # 更新状況を確認
├── export [repo]        # 問題セットを出力
├── stats [repo]         # 練習統計を表示
├── repos                # 登録リポジトリ一覧
│   ├── list             # 一覧表示
│   └── remove <repo>    # 登録解除
└── plugins              # プラグイン管理
    └── list             # 利用可能なパーサー一覧
```

### 各コマンドの詳細

#### `init <path>`

リポジトリを登録する。

```bash
# 基本
ctt init ./my-project

# 名前を指定
ctt init ./my-project --name "My Project"

# モノレポのサブディレクトリを指定（将来拡張）
ctt init ./monorepo --scope "packages/core"
```

**処理フロー:**
1. パスの存在確認
2. Git リポジトリかどうかの確認
3. リポジトリ情報を DB に登録
4. 現在のコミットハッシュを記録

#### `scan [repo]`

リポジトリを解析してトークンを抽出する。

```bash
# 全リポジトリをスキャン
ctt scan

# 特定のリポジトリをスキャン
ctt scan my-project

# 強制再スキャン（差分ではなく全件）
ctt scan --full
```

**処理フロー:**
1. 対象リポジトリの特定
2. 各ファイルに対応するパーサープラグインを選択
3. トークンを抽出
4. 正規化して DB に保存
5. コミットハッシュとスキャン日時を更新

#### `status [repo]`

リポジトリの更新状況を確認する。

```bash
ctt status

# 出力例:
# Repository         Last Scanned    Status
# ─────────────────────────────────────────────
# my-project         2 hours ago     ✓ Up to date
# another-project    3 days ago      ⚠ 5 commits behind
```

**処理フロー:**
1. 各リポジトリの現在のコミットハッシュを取得
2. DB に保存されているハッシュと比較
3. 差分があれば警告表示

#### `export [repo]`

問題セットをエクスポートする。

```bash
# JSON 形式（Monkeytype 言語ファイル互換）
ctt export my-project --format json --output ./wordlist.json

# テキスト形式（Monkeytype カスタムテキスト用）
ctt export my-project --format text --output ./wordlist.txt

# 頻度上位 N 件
ctt export my-project --top 500

# 特定の casing のみ
ctt export my-project --casing camel,snake

# 苦手な単語を重点的に（将来拡張）
ctt export my-project --weak --weight
```

**出力形式:**

JSON (Monkeytype 言語ファイル互換):
```json
{
  "name": "my-project",
  "words": [
    "getUserProfile",
    "fetchData",
    "handleError",
    ...
  ]
}
```

テキスト (カスタムテキスト用):
```
getUserProfile fetchData handleError setConfig validateInput ...
```

#### `stats [repo]`

練習統計を表示する。

```bash
ctt stats my-project

# 出力例:
# ═══════════════════════════════════════════════════════════════
#                    my-project Statistics
# ═══════════════════════════════════════════════════════════════
# 
# Overall:
#   Total tokens: 1,234
#   Practiced: 456 (37%)
#   Accuracy: 94.2%
# 
# Weakest (by error rate):
#   1. authentication  (canonical)  - 78% accuracy
#      └─ authenticateUser (camel)  - 75% accuracy
#      └─ authenticate_user (snake) - 82% accuracy
#   2. initialization  (canonical)  - 81% accuracy
#   ...
```

---

## プラグインシステム

### パーサープラグインのインターフェース

```typescript
// types/plugin.ts

export interface LanguageParserPlugin {
  /** プラグイン識別子 */
  readonly id: string;
  
  /** 表示名 */
  readonly name: string;
  
  /** 対応する拡張子 */
  readonly extensions: readonly string[];
  
  /** 
   * ソースコードからトークンを抽出する
   * @param source ソースコードの文字列
   * @param filepath ファイルパス（コンテキスト用）
   * @returns 抽出されたトークンの配列
   */
  extractTokens(source: string, filepath: string): Promise<RawToken[]>;
}

export interface RawToken {
  /** 元の表記 */
  raw: string;
  
  /** トークン種別 */
  kind: TokenKind;
  
  /** 行番号 (1-indexed) */
  line: number;
  
  /** 列番号 (1-indexed) */
  column: number;
}

export type TokenKind =
  | 'function'
  | 'method'
  | 'variable'
  | 'constant'
  | 'class'
  | 'interface'
  | 'type'
  | 'property'
  | 'parameter'
  | 'enum'
  | 'enum_member'
  | 'namespace'
  | 'unknown';
```

### プラグイン登録

```typescript
// plugins/registry.ts

export class PluginRegistry {
  private plugins: Map<string, LanguageParserPlugin> = new Map();
  
  register(plugin: LanguageParserPlugin): void {
    this.plugins.set(plugin.id, plugin);
  }
  
  getParserForFile(filepath: string): LanguageParserPlugin | undefined {
    const ext = path.extname(filepath);
    for (const plugin of this.plugins.values()) {
      if (plugin.extensions.includes(ext)) {
        return plugin;
      }
    }
    return undefined;
  }
  
  listPlugins(): LanguageParserPlugin[] {
    return Array.from(this.plugins.values());
  }
}
```

### TypeScript パーサープラグインの実装例

```typescript
// plugins/typescript/index.ts

import ts from 'typescript';
import type { LanguageParserPlugin, RawToken, TokenKind } from '../types';

export const typescriptParser: LanguageParserPlugin = {
  id: 'typescript',
  name: 'TypeScript / JavaScript',
  extensions: ['.ts', '.tsx', '.js', '.jsx', '.mts', '.cts'],
  
  async extractTokens(source: string, filepath: string): Promise<RawToken[]> {
    const sourceFile = ts.createSourceFile(
      filepath,
      source,
      ts.ScriptTarget.Latest,
      true
    );
    
    const tokens: RawToken[] = [];
    
    const visit = (node: ts.Node) => {
      const token = extractToken(node, sourceFile);
      if (token) {
        tokens.push(token);
      }
      ts.forEachChild(node, visit);
    };
    
    visit(sourceFile);
    
    return tokens;
  }
};

function extractToken(node: ts.Node, sourceFile: ts.SourceFile): RawToken | null {
  let name: string | undefined;
  let kind: TokenKind = 'unknown';
  
  if (ts.isFunctionDeclaration(node) && node.name) {
    name = node.name.text;
    kind = 'function';
  } else if (ts.isMethodDeclaration(node) && ts.isIdentifier(node.name)) {
    name = node.name.text;
    kind = 'method';
  } else if (ts.isVariableDeclaration(node) && ts.isIdentifier(node.name)) {
    name = node.name.text;
    kind = 'variable';
  } else if (ts.isClassDeclaration(node) && node.name) {
    name = node.name.text;
    kind = 'class';
  } else if (ts.isInterfaceDeclaration(node)) {
    name = node.name.text;
    kind = 'interface';
  } else if (ts.isTypeAliasDeclaration(node)) {
    name = node.name.text;
    kind = 'type';
  } else if (ts.isPropertyDeclaration(node) && ts.isIdentifier(node.name)) {
    name = node.name.text;
    kind = 'property';
  } else if (ts.isParameter(node) && ts.isIdentifier(node.name)) {
    name = node.name.text;
    kind = 'parameter';
  } else if (ts.isEnumDeclaration(node)) {
    name = node.name.text;
    kind = 'enum';
  } else if (ts.isEnumMember(node) && ts.isIdentifier(node.name)) {
    name = node.name.text;
    kind = 'enum_member';
  }
  
  if (!name) return null;
  
  // 短すぎる識別子は除外（i, x, _ など）
  if (name.length < 2) return null;
  
  // 予約語っぽいものは除外
  if (isCommonKeyword(name)) return null;
  
  const { line, character } = sourceFile.getLineAndCharacterOfPosition(node.getStart());
  
  return {
    raw: name,
    kind,
    line: line + 1,
    column: character + 1,
  };
}

function isCommonKeyword(name: string): boolean {
  const keywords = new Set([
    'if', 'else', 'for', 'while', 'do', 'switch', 'case', 'break',
    'continue', 'return', 'throw', 'try', 'catch', 'finally',
    'new', 'this', 'super', 'class', 'extends', 'implements',
    'import', 'export', 'default', 'from', 'as',
    'const', 'let', 'var', 'function', 'async', 'await',
    'true', 'false', 'null', 'undefined', 'void',
    'typeof', 'instanceof', 'in', 'of',
  ]);
  return keywords.has(name);
}
```

---

## ディレクトリ構成

```
code-typing-trainer/
├── package.json
├── tsconfig.json
├── README.md
│
├── src/
│   ├── index.ts                    # エントリポイント
│   │
│   ├── cli/                        # CLI 層
│   │   ├── index.ts                # gunshi 設定・ルートコマンド
│   │   ├── commands/
│   │   │   ├── init.ts
│   │   │   ├── scan.ts
│   │   │   ├── status.ts
│   │   │   ├── export.ts
│   │   │   ├── stats.ts
│   │   │   └── repos.ts
│   │   └── utils/
│   │       └── output.ts           # 出力フォーマット
│   │
│   ├── core/                       # コアドメイン
│   │   ├── analyzer/
│   │   │   ├── index.ts
│   │   │   └── scanner.ts
│   │   ├── normalizer/
│   │   │   ├── index.ts
│   │   │   ├── casing.ts
│   │   │   └── splitter.ts
│   │   └── exporter/
│   │       ├── index.ts
│   │       ├── json-exporter.ts
│   │       └── text-exporter.ts
│   │
│   ├── plugins/                    # パーサープラグイン
│   │   ├── types.ts                # プラグインインターフェース
│   │   ├── registry.ts             # プラグイン登録
│   │   ├── typescript/
│   │   │   └── index.ts
│   │   ├── python/
│   │   │   └── index.ts
│   │   └── rust/
│   │       └── index.ts
│   │
│   ├── infra/                      # インフラ層
│   │   ├── database/
│   │   │   ├── index.ts
│   │   │   ├── schema.ts
│   │   │   └── migrations/
│   │   │       └── 001_initial.ts
│   │   └── git/
│   │       └── index.ts
│   │
│   ├── config/                     # 設定管理
│   │   ├── index.ts
│   │   ├── schema.ts               # Zod スキーマ
│   │   ├── loader.ts               # 設定ファイル読み込み
│   │   └── defaults.ts             # デフォルト値
│   │
│   └── types/                      # 共通型定義
│       ├── index.ts
│       ├── token.ts
│       ├── repository.ts
│       └── config.ts               # 設定ファイルの型
│
├── tests/
│   ├── core/
│   │   └── normalizer.test.ts
│   ├── plugins/
│   │   └── typescript.test.ts
│   └── fixtures/
│       └── sample-code/
│
└── data/                           # ランタイムデータ（.gitignore）
    └── ctt.db                      # SQLite データベース
```

---

## 技術スタック

| 領域 | 技術 | 理由 |
|------|------|------|
| 言語 | TypeScript | 型安全性、エコシステム |
| CLI | gunshi | 宣言的、型安全、プラグインシステム |
| DB | better-sqlite3 | 同期 API、高速、型定義あり |
| Git | simple-git | シンプルな API |
| TS Parser | typescript (compiler API) | 公式、正確な AST |
| Config | zod | 型安全なバリデーション、型推論 |
| Glob | tinyglobby | 高速、モダンな glob マッチング |
| Test | vitest | 高速、TypeScript ネイティブ |
| Build | tsup | シンプル、ESM/CJS 対応 |

---

## Phase 1 MVP スコープ

最小限の動作確認ができる機能セット:

### 含めるもの

- [x] `init` - リポジトリ登録
- [x] `scan` - トークン抽出（TypeScript のみ）
- [x] `status` - 更新検知
- [x] `export` - JSON/テキスト出力
- [x] SQLite によるデータ永続化
- [x] 正規化ロジック

### 含めないもの（Phase 2 以降）

- [ ] 練習統計の記録・表示
- [ ] 苦手単語の重点出題
- [ ] Python / Rust パーサー
- [ ] 自前タイピング UI
- [ ] モノレポのスコープ指定

---

## 開発ロードマップ

### Phase 1: MVP（〜2週間）

1. プロジェクト初期化、依存関係セットアップ
2. DB スキーマ・マイグレーション
3. 正規化ロジック実装
4. TypeScript パーサー実装
5. CLI コマンド実装（init, scan, status, export）
6. 基本的なテスト

### Phase 2: 統計機能（〜2週間）

1. 練習結果のインポート機能
2. 統計テーブル・集計ロジック
3. `stats` コマンド実装
4. 苦手単語の抽出・重み付け出力

### Phase 3: 拡張性（〜2週間）

1. Python パーサー
2. Rust パーサー
3. プラグイン自動検出
4. モノレポスコープ対応

### Phase 4: タイピング UI（〜4週間）

1. Monkeytype のコア実装調査
2. 自前 UI の設計・実装
3. SQLite 直接連携
4. リアルタイム統計更新

---

## 未解決の設計判断

### 1. 統計のインポート方法

**選択肢:**
- A: Monkeytype の結果を手動で CSV/JSON にエクスポートして取り込む
- B: ブラウザ拡張で Monkeytype の結果を自動取得
- C: 自前 UI を作って最初から統計を収集

**現時点の方針:** Phase 1 では A、Phase 4 で C に移行

### 2. 差分更新のアルゴリズム

**選択肢:**
- A: 全件削除 → 全件挿入（シンプル）
- B: ファイル単位で差分検出 → 該当ファイルのみ更新
- C: トークン単位で差分マージ

**現時点の方針:** Phase 1 では A、必要に応じて B に改善

### 3. プラグインの配布形式

**選択肢:**
- A: 本体にバンドル（ビルトイン）
- B: npm パッケージとして別配布
- C: ローカルファイルとして読み込み

**現時点の方針:** Phase 1 では A、拡張性が必要になれば B

---

## 次のステップ

1. ✅ 要件定義・設計ドキュメント作成（本ドキュメント）
2. ⏳ プロジェクト初期化（package.json, tsconfig.json）
3. ⏳ DB スキーマ・マイグレーション実装
4. ⏳ 正規化ロジック実装・テスト
5. ⏳ TypeScript パーサー実装・テスト
6. ⏳ CLI コマンド実装

---

## 設定ファイル仕様

### 概要

プロジェクトごとにフィルタリングルールや解析設定をカスタマイズできる。
設定ファイルは `ctt.config.json` としてリポジトリルートに配置する（任意）。

### スキーマ定義

```typescript
// types/config.ts

export interface CttConfig {
  /** 設定ファイルのバージョン */
  version: 1;
  
  /** フィルタリングルール */
  filter?: FilterConfig;
  
  /** 解析対象の設定 */
  scan?: ScanConfig;
  
  /** エクスポート設定 */
  export?: ExportConfig;
}

export interface FilterConfig {
  /** 最小文字数（デフォルト: 2） */
  minLength?: number;
  
  /** 最大文字数（デフォルト: 無制限） */
  maxLength?: number;
  
  /** 除外する単語のパターン（正規表現） */
  excludePatterns?: string[];
  
  /** 除外する単語のリスト（完全一致） */
  excludeWords?: string[];
  
  /** 除外するトークン種別 */
  excludeKinds?: TokenKind[];
  
  /** 数字を含む識別子を除外するか（デフォルト: false） */
  excludeWithNumbers?: boolean;
  
  /** 単一の単語（分割できない）を除外するか（デフォルト: false） */
  excludeSingleSegment?: boolean;
  
  /** 最小出現回数（デフォルト: 1） */
  minFrequency?: number;
}

export interface ScanConfig {
  /** 対象とするファイルパターン（glob） */
  include?: string[];
  
  /** 除外するファイルパターン（glob） */
  exclude?: string[];
  
  /** 特定の拡張子のみ対象 */
  extensions?: string[];
}

export interface ExportConfig {
  /** デフォルトの出力形式 */
  defaultFormat?: 'json' | 'text';
  
  /** デフォルトの出力件数 */
  defaultLimit?: number;
  
  /** 頻度で重み付けするか */
  weightByFrequency?: boolean;
}
```

### 設定ファイル例

#### 基本的な例

```json
{
  "version": 1,
  "filter": {
    "minLength": 3,
    "excludeWords": ["data", "value", "item", "temp", "tmp"],
    "excludeWithNumbers": true
  }
}
```

#### TypeScript プロジェクト向け

```json
{
  "version": 1,
  "filter": {
    "minLength": 2,
    "excludePatterns": [
      "^_",
      "^I[A-Z]",
      "^T[A-Z]"
    ],
    "excludeWords": [
      "props", "state", "ctx", "req", "res",
      "err", "cb", "fn", "args", "opts"
    ],
    "excludeKinds": ["parameter"]
  },
  "scan": {
    "exclude": [
      "**/node_modules/**",
      "**/dist/**",
      "**/*.test.ts",
      "**/*.spec.ts"
    ]
  }
}
```

#### Python プロジェクト向け

```json
{
  "version": 1,
  "filter": {
    "minLength": 2,
    "excludePatterns": [
      "^_",
      "^__.*__$"
    ],
    "excludeWords": [
      "self", "cls", "args", "kwargs",
      "df", "np", "pd", "plt"
    ]
  },
  "scan": {
    "exclude": [
      "**/__pycache__/**",
      "**/venv/**",
      "**/.venv/**",
      "**/test_*.py"
    ]
  }
}
```

#### 厳格なフィルタリング（上級者向け）

```json
{
  "version": 1,
  "filter": {
    "minLength": 4,
    "maxLength": 30,
    "excludeSingleSegment": true,
    "excludeWithNumbers": true,
    "minFrequency": 2
  },
  "export": {
    "defaultLimit": 500,
    "weightByFrequency": true
  }
}
```

### デフォルト設定

設定ファイルがない場合や、特定のフィールドが未指定の場合に適用されるデフォルト値:

```typescript
const DEFAULT_CONFIG: Required<CttConfig> = {
  version: 1,
  filter: {
    minLength: 2,
    maxLength: Infinity,
    excludePatterns: [],
    excludeWords: [],
    excludeKinds: [],
    excludeWithNumbers: false,
    excludeSingleSegment: false,
    minFrequency: 1,
  },
  scan: {
    include: ['**/*'],
    exclude: [
      '**/node_modules/**',
      '**/dist/**',
      '**/build/**',
      '**/.git/**',
      '**/vendor/**',
      '**/venv/**',
      '**/__pycache__/**',
    ],
    extensions: [], // 空 = プラグインがサポートする全拡張子
  },
  export: {
    defaultFormat: 'json',
    defaultLimit: 1000,
    weightByFrequency: false,
  },
};
```

### CLI での設定指定

```bash
# 設定ファイルを明示的に指定
ctt scan --config ./custom-config.json

# 設定ファイルなしで実行（デフォルト設定を使用）
ctt scan --no-config

# CLI オプションで一部上書き
ctt export --min-length 4 --exclude-numbers
```

---

## 参考資料

- [gunshi - Modern JavaScript Command-line Library](https://gunshi.dev/)
- [Monkeytype - GitHub](https://github.com/monkeytypegame/monkeytype)
- [TypeScript Compiler API](https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API)
- [better-sqlite3](https://github.com/WiseLibs/better-sqlite3)
