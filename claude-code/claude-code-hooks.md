公式: <https://docs.anthropic.com/en/docs/claude-code/hooks>

# フック

シェルコマンドを登録して Claude Code の挙動をカスタマイズ・拡張する

# はじめに

Claude Code フックは、Claude Code のライフサイクルのさまざまなタイミングで実行されるユーザー定義のシェルコマンドです。フックを使うと、LLM がコマンドを実行するかどうかに依存せず、必ず実行される確定的な制御を Claude Code に与えられます。

例としては次のようなものがあります。

- **通知**: Claude Code が入力待ちやコマンド実行の許可を求めているときに、独自の通知方法を実装する
- **自動フォーマット**: すべてのファイル編集後に `.ts` は `prettier`、`.go` は `gofmt` のように実行する
- **ログ取得**: コンプライアンスやデバッグのため、実行されたすべてのコマンドを追跡・集計する
- **フィードバック**: Claude Code がコードベースの規約に従わないコードを生成した場合に自動でフィードバックを返す
- **カスタム権限**: 本番ファイルや機密ディレクトリの変更をブロックする

これらのルールをプロンプト指示ではなくフックとしてコード化することで、「提案」ではなく「アプリレベルのコード」として毎回確実に実行させることができます。

<Warning>
  フックはあなたのユーザー権限でシェルコマンドを **確認なく** 実行します。フックの安全性とセキュリティを担保する責任は利用者にあります。フックの使用に伴うデータ損失やシステム障害について、Anthropic は一切の責任を負いません。必ず [セキュリティ上の注意](#security-considerations) を確認してください。
</Warning>

## クイックスタート

このクイックスタートでは、Claude Code が実行するシェルコマンドをログに記録するフックを追加します。

前提: コマンドラインで JSON を処理する `jq` がインストールされていること。

### ステップ 1: フック設定を開く

`/hooks` [スラッシュコマンド](/en/docs/claude-code/slash-commands) を実行し、`PreToolUse` フックイベントを選択します。

`PreToolUse` フックはツール呼び出しの **前** に実行され、ツール呼び出しをブロックしたり、Claude に対して修正方法をフィードバックできます。

### ステップ 2: マッチャーを追加する

`+ Add new matcher…` を選択し、Bash ツール呼び出しのみでフックを実行するようにします。

マッチャーに `Bash` と入力します。

### ステップ 3: フックを追加する

`+ Add new hook…` を選択し、以下のコマンドを入力します。

```bash
jq -r '"\(.tool_input.command) - \(.tool_input.description // "No description")"' >> ~/.claude/bash-command-log.txt
```

### ステップ 4: 設定を保存する

保存場所として「User settings」を選びます。ホームディレクトリにログを書き出すので、このフックはすべてのプロジェクトに適用されます。

Esc を押して REPL に戻れば、フックの登録は完了です！

### ステップ 5: フックを確認する

再度 `/hooks` を実行するか、`~/.claude/settings.json` を開いて設定を確認してみましょう。

```json
"hooks": {
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "jq -r '\"\\(.tool_input.command) - \\(.tool_input.description // \"No description\")\"' >> ~/.claude/bash-command-log.txt"
        }
      ]
    }
  ]
}
```

## 設定ファイル

Claude Code のフックは、以下の [設定ファイル](/en/docs/claude-code/settings) に記述します。

- `~/.claude/settings.json` - ユーザー設定
- `.claude/settings.json` - プロジェクト設定
- `.claude/settings.local.json` - ローカルプロジェクト設定（リポジトリにコミットしない）
- エンタープライズ向け管理ポリシー設定

### 構造

フックはマッチャーごとに整理され、マッチャーごとに複数のフックを持てます。

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here"
          }
        ]
      }
    ]
  }
}
```

- **matcher**: ツール名に一致させるパターン（`PreToolUse` と `PostToolUse` のみ）
  - 単純な文字列は完全一致: 例 `Write` は Write ツールのみ一致
  - 正規表現も利用可: `Edit|Write` や `Notebook.*`
  - 省略または空文字列の場合、該当イベントすべてに適用
- **hooks**: パターンが一致したときに実行されるコマンドの配列
  - `type`: 現在は "command" のみ対応
  - `command`: 実行する bash コマンド

## フックイベント

### PreToolUse

Claude がツールパラメータを生成した直後、ツールを実行する **前** に走ります。

**よく使われるマッチャー**:

- `Task` - Agent タスク
- `Bash` - シェルコマンド
- `Glob` - ファイルパターン検索
- `Grep` - コンテンツ検索
- `Read` - ファイル読み取り
- `Edit`, `MultiEdit` - ファイル編集
- `Write` - ファイル書き込み
- `WebFetch`, `WebSearch` - Web 操作

### PostToolUse

ツールが **正常に完了した直後** に走ります。

`PreToolUse` と同じマッチャーを認識します。

### Notification

Claude Code が通知を送信するときに走ります。

### Stop

Claude Code の応答が終了したときに走ります。

## フック入力

フックは stdin 経由で JSON データを受け取り、セッション情報とイベント固有の情報を含みます。

```typescript
{
  // 共通フィールド
  session_id: string
  transcript_path: string  // 会話 JSON のパス

  // イベント固有フィールド
  ...
}
```

### PreToolUse の入力

`tool_input` の正確なスキーマはツールによって異なります。

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../00893aaf-19fa-41d2-8238-13269b9b3ca0.jsonl",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.txt",
    "content": "file content"
  }
}
```

### PostToolUse の入力

`tool_input` と `tool_response` のスキーマはツールによって異なります。

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../00893aaf-19fa-41d2-8238-13269b9b3ca0.jsonl",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.txt",
    "content": "file content"
  },
  "tool_response": {
    "filePath": "/path/to/file.txt",
    "success": true
  }
}
```

### Notification の入力

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../00893aaf-19fa-41d2-8238-13269b9b3ca0.jsonl",
  "message": "Task completed successfully",
  "title": "Claude Code"
}
```

### Stop の入力

`stop_hook_active` が true の場合、Claude Code はすでに Stop フックの結果として継続中です。無限ループを防ぐため、この値を確認するかトランスクリプトを解析してください。

```json
{
  "session_id": "abc123",
  "transcript_path": "~/.claude/projects/.../00893aaf-19fa-41d2-8238-13269b9b3ca0.jsonl",
  "stop_hook_active": true
}
```

## フック出力

フックは Claude Code に対し 2 つの方法で出力を返します。いずれもブロックするかどうか、Claude やユーザーに見せるフィードバックを伝えるためのものです。

### シンプル: 退出コード

フックは終了コード、stdout、stderr で状態を伝えます。

- **退出コード 0**: 成功。`stdout` はトランスクリプトモード（CTRL-R）でユーザーに表示
- **退出コード 2**: ブロックエラー。`stderr` が Claude に返され自動処理される（イベント別動作は後述）
- **それ以外の退出コード**: 非ブロックエラー。`stderr` がユーザーに表示され、処理は続行

#### 退出コード 2 の動作

| フックイベント | 挙動                                               |
| -------------- | -------------------------------------------------- |
| `PreToolUse`   | ツール呼び出しをブロック、エラーを Claude に伝える |
| `PostToolUse`  | Claude にエラーを示す（ツールはすでに実行済み）    |
| `Notification` | N/A、stderr はユーザーのみに表示                   |
| `Stop`         | 停止をブロック、エラーを Claude に伝える           |

### 高度: JSON 出力

より高度な制御を行う場合、フックは `stdout` に構造化 JSON を返せます。

#### 共通 JSON フィールド

すべてのフックタイプで以下の任意フィールドを含められます。

```json
{
  "continue": true, // フック実行後 Claude を続行させるか（デフォルト: true）
  "stopReason": "string", // continue が false のときユーザーに表示（Claude には非表示）
  "suppressOutput": true // stdout をトランスクリプトに表示しない（デフォルト: false）
}
```

`continue` が false の場合、フック実行後 Claude は処理を停止します。

- `PreToolUse` では、これは "decision": "block" とは異なります。前者はセッション全体を停止、後者は特定ツール呼び出しのみブロックします。
- `PostToolUse` でも同様に異なります。
- `Stop` では "decision": "block" より優先されます。
- いずれの場合も `continue` = false が他の出力より優先されます。

#### `PreToolUse` の決定制御

`PreToolUse` フックはツール呼び出しの可否を制御できます。

- "approve" は権限確認を完全にバイパス。`reason` はユーザーに表示され、Claude には非表示
- "block" はツール呼び出しを防ぎ、`reason` を Claude に提示
- `undefined` は既存の許可フローに従う。`reason` は無視される

```json
{
  "decision": "approve" | "block" | undefined,
  "reason": "Explanation for decision"
}
```

#### `PostToolUse` の決定制御

`PostToolUse` フックはツール呼び出し後のフローを制御できます。

- "block" は `reason` を Claude に提示して自動フィードバック
- `undefined` は何もしません。`reason` は無視

```json
{
  "decision": "block" | undefined,
  "reason": "Explanation for decision"
}
```

#### `Stop` の決定制御

`Stop` フックは Claude が処理を終了するかどうかを制御できます。

- "block" は停止を防ぎます。Claude に指示するため `reason` を必ず指定
- `undefined` は停止を許可。`reason` は無視

```json
{
  "decision": "block" | undefined,
  "reason": "Must be provided when Claude is blocked from stopping"
}
```

#### JSON 出力例: Bash コマンドの検証

```python
#!/usr/bin/env python3
import json
import re
import sys

# (regex, message) タプルのリストでバリデーションルールを定義
VALIDATION_RULES = [
    (
        r"\bgrep\b(?!.*\|)",
        "より高性能な 'rg' (ripgrep) を使用してください",
    ),
    (
        r"\bfind\s+\S+\s+-name\b",
        "より高性能な 'rg --files | rg pattern' または 'rg --files -g pattern' を使用してください",
    ),
]


def validate_command(command: str) -> list[str]:
    issues = []
    for pattern, message in VALIDATION_RULES:
        if re.search(pattern, command):
            issues.append(message)
    return issues


try:
    input_data = json.load(sys.stdin)
except json.JSONDecodeError as e:
    print(f"Error: Invalid JSON input: {e}", file=sys.stderr)
    sys.exit(1)

tool_name = input_data.get("tool_name", "")
tool_input = input_data.get("tool_input", {})
command = tool_input.get("command", "")

if tool_name != "Bash" or not command:
    sys.exit(1)

# コマンドを検証
issues = validate_command(command)

if issues:
    for message in issues:
        print(f"• {message}", file=sys.stderr)
    # 退出コード 2 でツール呼び出しをブロックし、stderr を Claude に表示
    sys.exit(2)
```

#### `Stop` の決定制御

`Stop` フックはツール実行を制御できます。

```json
{
  "decision": "approve" | "block",
  "reason": "Human-readable explanation"
}
```

## MCP ツールとの連携

Claude Code フックは [Model Context Protocol (MCP) ツール](/en/docs/claude-code/mcp) とシームレスに連携します。MCP サーバーが提供するツールもフックの対象にできます。

### MCP ツールの命名規則

MCP ツールは `mcp__<server>__<tool>` という命名パターンです。

- `mcp__memory__create_entities` - Memory サーバーの create_entities ツール
- `mcp__filesystem__read_file` - Filesystem サーバーの read_file ツール
- `mcp__github__search_repositories` - GitHub サーバーの search ツール

### MCP ツールを対象にしたフック設定

特定の MCP ツール、または MCP サーバー全体を対象にできます。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__memory__.*",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Memory operation initiated' >> ~/mcp-operations.log"
          }
        ]
      },
      {
        "matcher": "mcp__.*__write.*",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/scripts/validate-mcp-write.py"
          }
        ]
      }
    ]
  }
}
```

## 例

### コードフォーマット

ファイル変更後に自動フォーマットを実行します。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/scripts/format-code.sh"
          }
        ]
      }
    ]
  }
}
```

### 通知カスタマイズ

Claude Code が許可を求めているときや入力がアイドル状態になったときに送る通知をカスタマイズします。

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/my_custom_notifier.py"
          }
        ]
      }
    ]
  }
}
```

## セキュリティ上の注意

### 免責事項

**自己責任でご利用ください**: Claude Code フックは任意のシェルコマンドを自動的に実行します。フックを使用することにより、以下を認識・同意したものとみなします。

- 設定したコマンドの責任はすべて利用者にある
- フックはユーザーアカウントがアクセスできるすべてのファイルを変更・削除・閲覧できる
- 悪意のある、または不適切なフックによりデータ損失やシステム障害が発生する可能性がある
- Anthropic はフックの利用によるいかなる損害についても責任を負わない
- 本番環境で使用する前に、安全な環境で十分にテストする

コマンドを追加する前に、必ず内容を確認・理解してください。

### セキュリティベストプラクティス

1. **入力の検証・サニタイズ**: 入力データを盲信しない
2. **シェル変数は必ず引用**: `$VAR` ではなく `"$VAR"` を使う
3. **パストラバーサルを防ぐ**: ファイルパスに `..` が含まれていないか確認
4. **絶対パスを使用**: スクリプトにはフルパスを指定
5. **機密ファイルを除外**: `.env`, `.git/`, 鍵などを避ける

### 設定の安全性

設定ファイルを直接編集しても、フックはすぐには反映されません。Claude Code は次のように動作します。

1. 起動時にフック設定のスナップショットを取得
2. セッション中はこのスナップショットを使用
3. 外部でフックが変更されると警告を表示
4. 変更を適用するには `/hooks` メニューでレビューが必要

これにより、不正なフック変更が現在のセッションに影響するのを防ぎます。

## フック実行の詳細

- **タイムアウト**: 最大 60 秒
- **並列実行**: 一致したすべてのフックを並列実行
- **実行環境**: カレントディレクトリで Claude Code の環境変数を継承
- **入力**: stdin で JSON を受け取る
- **出力**:
  - PreToolUse/PostToolUse/Stop: 進行状況がトランスクリプトに表示（Ctrl-R）
  - Notification: デバッグログのみ（`--debug`）

## デバッグ

フックをトラブルシュートするには:

1. `/hooks` メニューで設定が表示されるか確認
2. [設定ファイル](/en/docs/claude-code/settings) が有効な JSON であるか確認
3. コマンドを手動でテスト
4. 退出コードを確認
5. stdout と stderr のフォーマットを確認
6. 適切に引用がエスケープされているか確認

トランスクリプトモード (Ctrl-R) では次が表示されます:

- どのフックが実行中か
- 実行しているコマンド
- 成功/失敗ステータス
- 出力やエラーメッセージ
