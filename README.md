# ui-toolkit-lsps

Claude Code 向けの Unity UI Toolkit LSP プラグイン集です。

現在このリポジトリで利用できる USS 用プラグイン:

- `uss-lsp`: `unity_code_native` を使って `.uss` を LSP 接続

## セットアップ

### 1. `unity_code_native` を用意する

`unity_code_native` は USS Language Server を内蔵しており、LSP は stdio（stdin/stdout）で通信します。  
起動時に **Unity プロジェクトのパスを第1引数** で渡す必要があります。

```bash
git clone https://github.com/hackerzhuli/unity_code_native.git
cd unity_code_native
cargo build --release
```

生成物:

- macOS/Linux: `target/release/unity_code_native`
- Windows: `target/release/unity_code_native.exe`

生成した実行ファイルを `PATH` の通る場所に置いてください。  
`PATH` に置かない場合は、`plugins/uss-lsp/.lsp.json` の `command` を絶対パスに変更してください。

### 2. 単体起動テスト（重要）

Unity プロジェクト直下で:

```bash
unity_code_native .
```

または:

```bash
unity_code_native /path/to/YourUnityProject
```

`usage` が出ずに起動できれば、引数付き起動は正常です。

### 3. Claude Code にマーケットプレースを登録

このリポジトリをマーケットプレースとして追加:

```bash
claude plugin marketplace add /path/to/ui-toolkit-lsps
```

`uss-lsp` をインストール:

```bash
claude plugin install uss-lsp@ui-toolkit-lsps --scope project
```

確認:

```bash
claude plugin list
```

## 実装内容（USS）

`plugins/uss-lsp/.lsp.json` は以下のように設定してあります。

```json
{
  "uss": {
    "command": "unity_code_native",
    "args": ["."],
    "extensionToLanguage": {
      ".uss": "uss"
    }
  }
}
```

Claude Code を Unity プロジェクトのルートで開くことで、`args: ["."]` が Unity プロジェクトパスとして渡される想定です。
