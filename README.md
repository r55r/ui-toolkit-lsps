# ui-toolkit-lsps

Claude Code 向けの Unity UI Toolkit LSP プラグイン集です。

現在このリポジトリで利用できるプラグイン:

- `uss-lsp`: `unity_code_native` を使って `.uss` を LSP 接続
- `uxml-lsp`: `LemMinX` を使って `.uxml` を LSP 接続（同梱XSDで補完/検証）

## セットアップ

### 1. LSP 実行ファイルを用意する

#### `uss-lsp` 用: `unity_code_native`

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

#### `uxml-lsp` 用: `lemminx`

`uxml-lsp` は XML Language Server の `LemMinX` を利用します。  
`lemminx` コマンドが `PATH` で実行できる状態にしてください。

`lemminx` の代わりに `java -jar /path/to/org.eclipse.lemminx-uber.jar` で運用したい場合は、`plugins/uxml-lsp/.lsp.json` の `command` / `args` を調整してください。

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

`LemMinX` 側は次で起動確認できます:

```bash
lemminx
```

### 3. Claude Code にマーケットプレースを登録

このリポジトリをマーケットプレースとして追加:

```bash
claude plugin marketplace add /path/to/ui-toolkit-lsps
```

`uss-lsp` をインストール:

```bash
claude plugin install uss-lsp@ui-toolkit-lsps --scope project
```

`uxml-lsp` をインストール:

```bash
claude plugin install uxml-lsp@ui-toolkit-lsps --scope project
```

確認:

```bash
claude plugin list
```

## 実装内容

### USS (`plugins/uss-lsp/.lsp.json`)

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

### UXML (`plugins/uxml-lsp/.lsp.json`)

`plugins/uxml-lsp/.lsp.json` は `lemminx` を起動し、`.uxml` を XML として関連付けています。  
さらに `xml.fileAssociations` で `**/*.uxml` を `urn:ui-toolkit-lsps:uxml` に関連付け、  
XML Catalog で実体スキーマを解決します。

```json
{
  "xml": {
    "command": "lemminx",
    "extensionToLanguage": {
      ".uxml": "xml"
    },
    "initializationOptions": {
      "settings": {
        "xml": {
          "catalogs": [
            "./.claude/uxml-lsp/catalog.xml",
            "file://${CLAUDE_PLUGIN_ROOT}/schemas/catalog.xml"
          ],
          "fileAssociations": [
            {
              "pattern": "**/*.uxml",
              "systemId": "urn:ui-toolkit-lsps:uxml"
            }
          ]
        }
      }
    }
  }
}
```

### 同梱スキーマ

`plugins/uxml-lsp/schemas/` に以下を同梱しています:

- `catalog.xml`: `urn:ui-toolkit-lsps:uxml` -> `UIElements.xsd`
- `UIElements.xsd`: エントリポイント（他の XSD を import）
- `UnityEngine.UIElements.xsd`, `UnityEditor.UIElements.xsd` ほか関連 XSD 一式
- `UNITY_SCHEMA_SOURCE.md`: 生成元 Unity バージョン情報

これらの XSD は **Unity Editor 6000.2.10f1** で `UnityEditor.UIElements.UxmlSchemaGenerator.UpdateSchemaFiles()` を実行して生成した公式スキーマです（手書きではありません）。
再生成は `scripts/sync-unity-uxml-schema.sh` で行えます。

```bash
./scripts/sync-unity-uxml-schema.sh /Applications/Unity/Hub/Editor/6000.2.10f1/Unity.app 6000.2.10f1
```

### 利用者によるスキーマ差し替え

`uxml-lsp` は、プロジェクト側のカタログを優先して読みます。  
以下を作成すると、同梱スキーマを差し替えできます（Unity バージョンごとの運用を推奨）。

1. `.claude/uxml-lsp/` を作成
2. 任意名でカスタムXSDを配置（例: `.claude/uxml-lsp/MyUIElements.xsd`）
3. `.claude/uxml-lsp/catalog.xml` を作成

```xml
<?xml version="1.0" encoding="UTF-8"?>
<catalog xmlns="urn:oasis:names:tc:entity:xmlns:xml:catalog">
  <uri name="urn:ui-toolkit-lsps:uxml" uri="MyUIElements.xsd"/>
</catalog>
```

`catalog.xml` を置いた後、Claude Code を再起動すると新しいスキーマが適用されます。
