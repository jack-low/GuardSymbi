# GuardSymbi
新言語

## GuardSymbi 言語設計ドキュメント

### 1. 背景と目的

* **背景**: 既存言語のエラー処理の煩雑さ、保守性の低さ、AI連携不足を解消
* **目的**: シンプルかつ堅牢な宣言的タスク／手続き混在型スクリプト言語を提供し、内部MCPを用いたAI支援による自動修正・最適化を実現

---

### 2. 概要設計

#### 2.1 コアコンセプト

* **タスク定義**: `task` ブロックでパイプラインを宣言
* **ガード句**: `guard ... else { ... }` による早期リターン＋AI提案
* **onError**: 各ステップ固有のエラーハンドラ埋め込み
* **@mcpアノテーション**: AI最適化／ドキュメント生成トリガー
* **手続き関数**: `function` キーワードによる通常関数定義

#### 2.2 モジュール構成

* **標準モジュール**: File, JSON, PDF, DataValidator…
* **MCPモジュール**: AI（`AI.fix`, `AI.suggestFix`, `AI.optimize`, `AI.explainError`）
* **拡張機構**: 外部パッケージは同一形式で `import 名 from "リポジトリ"`

#### 2.3 実行モデル

1. パーサで AST を生成
2. モジュール／タスク依存グラフを構築
3. `run` されたタスクをトップダウン実行
4. 各ステップでガード・onError をチェック
5. 必要時に MCP 経由で AI モデル呼び出し
6. レポート出力／ログ出力

#### 2.4 エラー処理ワークフロー

```
step: 処理呼び出し
└─ guard 成功? → 続行 : else → AI.suggestFix + return
step: 次処理
└─ onError ブロック? → 内部 retry / exit / AI.fix
```

#### 2.5 サンプルコード

```symbi
module "ReportGenerator" version "1.0" {
  import File from "std"
  import JSON from "std"
  import AI   from "mcp"

  task load {
    from "data.json"
    parse JSON
    onError {
      AI.fix("JSON parse failed", context)
      retry
    }
  }

  @mcp task validateAndOptimize {
    input load.output
    guard let valid = DataValidator.check(input) else {
      AI.suggestFix("Invalid schema", input)
      return
    }
    let optimized = AI.optimize(valid)
    output optimized
  }

  task report {
    input validateAndOptimize.output
    compute stats from input
    render PDF as "report.pdf"
    onError {
      AI.explainError("PDF.render", context)
      exit
    }
  }

  run report
}
```

---

## 3. フェーズ移行

### 3.1 仕様書フェーズ（要件定義・設計仕様）

#### 3.1.1 機能要件

* **言語構文**

  * モジュール宣言: `module "Name" version "x.y"`
  * タスク定義: `task <name> { ... }`
  * ガード句: `guard <expr> else { ... }`
  * エラーハンドラ: `onError { ... }`
  * アノテーション: `@mcp`, `@ai`
  * 手続き関数: `function <name>(...) { ... }`
  * 実行開始: `run <task>`
* **モジュールインタフェース**

  * インポート構文: `import <Name> from "<repo>"`
  * 標準モジュール API 一覧

    * **File**: `read`, `write`, `exists` など
    * **JSON**: `parse`, `stringify`
    * **PDF**: `render`, `export`
    * **DataValidator**: `check`, `schema`
    * **AI**: `fix`, `suggestFix`, `optimize`, `explainError`
* **エラー処理パターン**

  * `guard` + `onError` による早期リターン・再試行
  * `retry`, `exit`, `AI.fix`, `AI.suggestFix` の動作定義
* **MCP連携要件**

  * AI 呼び出しエンドポイントの URI/メソッド仕様
  * コンテキストデータ構造（JSON フォーマット定義）
  * 同期／非同期呼び出しの制御（タイムアウト、コールバック方式）

#### 3.1.2 非機能要件

* **パフォーマンス**

  * ランタイム起動時間: < 100ms
  * 大規模タスク依存グラフ (1,000 ノード) の実行時間: < 5s
* **セキュリティ**

  * モジュール署名検証とハッシュチェック
  * AI 通信の TLS1.2 以上による暗号化
  * サンドボックス実行オプションの提供
* **拡張性／後方互換性**

  * モジュールセマンティクスのバージョニング管理
  * プラグイン API 安定性保証（SEMVER 準拠）

#### 3.1.3 アーキテクチャ

* **コンポーネント構成**

  * **Parser**: 字句解析 → 構文解析 → AST 生成
  * **Executor**: AST 解釈 → タスク実行 → ガード/onError 適用
  * **ErrorChecker**: 実行時エラー検知・分類
  * **MCP Gateway**: AI モデル呼び出し管理 → レスポンス統合
  * **Logger**: 実行ログ収集・可視化

* **データフロー図**

```text
+---------------+     +---------+     +----------+
| ソースコード  | --> | Parser  | --> |   AST    |
+---------------+     +---------+     +----------+
                           |               |
                           v               v
                     +-----------+     +----------+
                     | Executor  | --> | TaskExec |
                     +-----------+     +----------+
                           |               |
                           v               |
                    +----------------+      |
                    | ErrorChecker   |      |
                    +----------------+      |
                           |               |
                           v               |
                    +-------------+        |
                    | MCPGateway  | <------+   
                    +-------------+            
                           |                   
                           v                   
                     +-----------+             
                     | AI Model  |             
                     +-----------+             
                           |
                           v
                     +-----------+
                     | Executor  |
                     +-----------+
```

#### 3.1.4 ユースケースシナリオ

* **シナリオ1: レポート生成**

```
  [Start: data.json]
        |
        v
      [load]
        |
        v
```

\[validateAndOptimize]
|
v
\[report]
|
v
\[End: report.pdf]

```

- **シナリオ2: バッチ処理**

```

\[Start: file list]
\|       |       |
v       v       v
\[load]  \[load]  \[load]
\|       |       |
v       v       v
\[validateAndOptimize] x3
\|       |       |
v       v       v
\[report] \[report] \[report]
\|       |       |
v       v       v
\[End: individual PDFs]

- **シナリオ3: インタラクティブデバッグ**
```

> run load    # ロードタスクを実行
> ↳ guardチェック → 成功 → 次へ
> inspect context  # 現在の状態確認
> run validateAndOptimize
> ↳ onError? → データ不整合 → AI.suggestFix起動
> ↳ 修正後 retry → 成功 → 次へ
> run report
> ↳ PDF生成 → 終了

```

### 3.2 詳細設計フェーズ（設計仕様→実装準備）1.4 ユースケースシナリオ

- **シナリオ1: レポート生成**
1. **入力**: `data.json`
2. **タスク**: `load` → `validateAndOptimize` → `report`
3. **正常終了**: `report.pdf` を出力
4. **エラー時**: JSON parse error → `AI.fix` で自動修正 → 再試行
- **シナリオ2: バッチ処理**
- 複数ファイルを並列実行し、個別失敗時は `onError` による `retry` または `continue`
- **シナリオ3: インタラクティブデバッグ**
1. REPL で `run load` 実行
2. `guard` 内部状態を確認しながら逐次デバッグ

### 3.2 詳細設計フェーズ（設計仕様→実装準備）（設計仕様→実装準備）

- **文法定義**
- BNF/EBNFによる正式文法
- トークン定義／字句規則
- **型システム詳細**
- 型推論ルール
- ガード句・タスク間の型整合性
- **MCPプロトコル仕様**
- AI 呼び出しエンドポイント
- コンテキスト構造体定義
- **ランタイム／データ構造**
- AST ノード構造
- タスク実行キュー
- **モジュールインタフェース**
- 各標準モジュールの関数シグネチャ
- 外部パッケージAPI仕様
- **テスト計画**
- 単体テスト／統合テスト項目
- フレームワーク選定

---

