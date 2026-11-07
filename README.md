# Premiere Pro と Claude を MCP でつなぐ方法（Windows 版）

Claude（Claude Code）から、Adobe Premiere Pro を日本語の指示で操作できるようにする手順です。
Windows 11 と日本語版 Premiere Pro 2026 で、実際に設定して動かしたときの記録をもとにまとめています。

> [!IMPORTANT]
> - ここで使う MCP サーバー [`premiere-pro-mcp`](https://github.com/leancoderkavy/premiere-pro-mcp) は、有志が作っているオープンソース（MIT ライセンス）です。**Adobe の公式ツールではありません**。
> - 2026年11月時点で、Adobe 公式の Premiere Pro 用 MCP は見つかりませんでした（Adobe 公式のコネクタは Express・Firefly・Acrobat などが対象で、Premiere Pro は操作できません）。
> - 初めて編集系の操作を試すときは、**プロジェクトのコピー**を使ってください。

---

## 目次

- [できること](#できること)
- [仕組み](#仕組み)
- [動作確認した環境](#動作確認した環境)
- [必要なもの](#必要なもの)
- [手順](#手順)
- [Claude への頼み方の例](#claude-への頼み方の例)
- [つまずきポイント](#つまずきポイント)
- [日本語版 Premiere での注意](#日本語版-premiere-での注意)
- [アンインストール](#アンインストール)
- [参考リンク](#参考リンク)

---

## できること

Claude に頼むだけで、Premiere Pro の中身を読み取ったり、編集したりできます。

- 開いているプロジェクト、シーケンス、トラック、クリップの情報を読む
- マーカーを打つ、一覧にする（色付きのマーカーを一度に最大200個）
- カット、トリム、リップル削除、トランジションの追加
- SRT ファイルからキャプショントラックを作る
- FCP XML・EDL の書き出しと読み込み、Media Encoder での書き出し
- エフェクトのプロパティやキーフレームの設定

実際にこんな使い方をしました。

- 文字起こしと照らし合わせて、「〇〇と言っている箇所」のタイムコードを探す
- 編集方針に沿って、テロップやアイキャッチを置く位置にマーカーを打つ
- ダイジェスト部分のカットに、ゆっくり寄るズームのキーフレームを付ける

---

## 仕組み

```mermaid
flowchart LR
  A["Claude Code<br>（MCP クライアント）"] -- "MCP（stdio）" --> B["premiere-pro-mcp<br>（Node.js の MCP サーバー）"]
  B -- "命令ファイル" --> C[("中継フォルダ<br>Bridge directory")]
  C -- "結果ファイル" --> B
  C <--> D["MCP パネル<br>（Premiere 内で動く CEP 拡張）"]
  D -- "ExtendScript" --> E["Premiere Pro"]
```

1. Claude が MCP サーバーに「この操作をして」と送ります。
2. MCP サーバーが、Premiere 用のスクリプト（ExtendScript）を**中継フォルダ**に書き出します。
3. Premiere の中で動いている **MCP パネル**がそれを実行し、結果を中継フォルダに書き戻します。

このため、使うときは **Premiere 側で MCP パネルを開いて「Running」にしておく**必要があります。

---

## 動作確認した環境

| 項目 | バージョン |
|---|---|
| OS | Windows 11 Home |
| Premiere Pro | 2026（日本語版） |
| Node.js | 24.15.0 |
| Claude | Claude Code（デスクトップアプリ） |
| premiere-pro-mcp | 1.21.0 |

2026年11月時点の最新版は 1.29.0 です。公式の導入手順は 1.21.0 のときと変わっていないので、以下のコマンドのバージョン番号を差し替えれば使えるはずです（このガイドの内容は 1.21.0 で確認したものです）。
最新版の番号は、次のコマンドで確認できます。

```bat
npm view premiere-pro-mcp version
```

---

## 必要なもの

- Windows 10 / 11（macOS でも使えますが、このガイドは Windows の手順です）
- Adobe Premiere Pro 2020〜2026
- [Node.js](https://nodejs.org/ja) 20.19 以上（`node --version` で確認できます）
- [Claude Code](https://claude.com/product/claude-code)（ターミナル版でもデスクトップアプリでも可）

> [!NOTE]
> 以下のコマンドは、すべて **コマンドプロンプト**（スタートメニューで「cmd」と検索）で実行してください。
> PowerShell では、`npx` や `claude` の入り方によって `--` の扱いが変わったり、スクリプトの実行が止められたりすることがあります。

---

## 手順

### 1. Premiere Pro を終了する

MCP パネルは Premiere の起動時に読み込まれます。入れる前に、Premiere を完全に終了しておきます。

### 2. MCP パネル（CEP 拡張）を入れる

```bat
npx -y premiere-pro-mcp@1.21.0 --install-cep
```

このコマンドは、次の2つを行います。

- `%APPDATA%\Adobe\CEP\extensions\MCPBridgeCEP` に MCP パネルを置く
- 署名のない CEP 拡張を読み込めるように、デバッグモードを有効にする（レジストリ `HKEY_CURRENT_USER\Software\Adobe\CSXS.9`〜`CSXS.14` の `PlayerDebugMode` を `1` にする）

> [!WARNING]
> デバッグモードを有効にすると、署名のない CEP 拡張全般が読み込まれるようになります。出どころの分からない拡張は入れないようにしてください。

### 3. Claude Code に MCP サーバーを登録する

```bat
claude mcp add --scope user premiere-pro -- cmd /c npx -y premiere-pro-mcp@1.21.0
```

- `--scope user` を付けると、どのフォルダで Claude Code を開いても使えるようになります。
- Windows では、`npx` を `cmd /c` 経由で起動します。
- **Git Bash** で実行する場合は、先頭に `MSYS_NO_PATHCONV=1 ` を付けてください（付けないと `/c` が `C:/` に書き換えられ、起動できなくなります）。

登録できたかを確認します。

```bat
claude mcp list
```

`premiere-pro: cmd /c npx -y premiere-pro-mcp@1.21.0 - ✓ Connected` と表示されれば成功です。

<details>
<summary>Claude Desktop（チャットアプリ）で使う場合（筆者は未検証）</summary>

`claude_desktop_config.json` の `mcpServers` に、次を追加します。

```json
{
  "mcpServers": {
    "premiere-pro": {
      "command": "cmd",
      "args": ["/c", "npx", "-y", "premiere-pro-mcp@1.21.0"]
    }
  }
}
```

</details>

### 4. 診断する

```bat
npx -y premiere-pro-mcp@1.21.0 --doctor
```

`MCP server`・`Node.js runtime`・`Premiere Connector` の3つが `Ready` なら大丈夫です。
`Live Premiere check` は `Not checked` のままで問題ありません（Premiere との実際の接続は、手順6で確かめます）。

### 5. Premiere で MCP パネルを開く

1. Premiere を起動し、シーケンスのあるプロジェクトを開きます。
2. メニューの **ウィンドウ > エクステンション > MCP for Adobe Premiere Pro** を開きます。
3. パネル上部の BRIDGE STATUS が、緑の **Running** になれば成功です。

赤い **Connector needs attention** になった場合は、[つまずきポイント 1](#1-パネルが-connector-needs-attention-になるbridge-directory-rejected) を見てください。

### 6. Claude から接続を確かめる

MCP を登録したあとは、Claude Code で**新しいセッション**を開いてください（登録前から開いていたセッションには反映されません）。

新しいセッションで、次のように頼みます。

> Premiere の接続確認をして（変更はしない）

読み取り専用の `verify_premiere_connection` が実行され、つながっているかを確かめてくれます。
続けて「開いているプロジェクトの情報を教えて」と頼めば、プロジェクト名やシーケンスの情報が返ってきます。

---

## Claude への頼み方の例

| 頼み方 | 中で使われる主なツール |
|---|---|
| Premiere の接続確認をして（変更はしない） | `verify_premiere_connection` |
| 開いているプロジェクトとシーケンスの情報を教えて | `get_project_info` / `get_active_sequence` |
| シーケンスのマーカーを一覧にして | `list_markers` |
| 45分あたりのクリップの並びを教えて | `get_active_sequence`（範囲指定） |
| 編集する前に、プロジェクトのバックアップを作って | `create_project_backup` |
| この SRT からキャプショントラックを作って | `import_media` / `create_caption_track` |

> [!TIP]
> 最初のうちは「変更はしない」「まず確認だけ」と添えて、読み取りから始めるのがおすすめです。

---

## つまずきポイント

### 1. パネルが「Connector needs attention」になる（Bridge directory rejected）

パネル下部のログに、次のようなエラーが出る場合です。

```text
Bridge directory rejected: Bridge directory grants write access to untrusted identities (S-1-5-21-...): C:\Users\<ユーザー名>\AppData\Local\Temp\premiere-mcp-bridge
```

この状態で **Start Bridge** を押しても、同じ理由で拒否されます。

**原因**

MCP パネルは、中継フォルダに置かれた命令を Premiere の中でそのまま実行します。ほかの人が命令を置けると危険なので、このパッケージは中継フォルダ**と、その上の階層のすべてのフォルダ**について、「自分・SYSTEM・Administrators 以外が書き込めないか」を検査しています。

次のような環境では、ユーザーフォルダの配下がどこも検査を通らず、既定の中継フォルダ（`%TEMP%\premiere-mcp-bridge`）が使えません。

- 別のローカルアカウントに、ユーザーフォルダへの書き込み権限を付けている
- Codex CLI のサンドボックスなど、ほかのツールが `AppData` や `Temp` に権限を追加している

権限は次のコマンドで確認できます。自分・SYSTEM・Administrators 以外のアカウントやグループに `(F)`（フルコントロール）や `(M)`（変更）が付いていたら、これに当てはまります。

```bat
icacls "%USERPROFILE%"
icacls "%LOCALAPPDATA%\Temp"
```

> [!CAUTION]
> これらの権限は、ほかのツールが動くために付けられていることがあります。むやみに削除しないでください。

**対処：中継フォルダを `C:\ProgramData\premiere-mcp-bridge` に変える**

`C:\ProgramData` の中に作ったフォルダは、作った本人にだけフル権限が付くので、検査を通ります。

1. **フォルダは自分で作らないでください。** パッケージ（MCP パネルまたは MCP サーバー）が最初に使うときにフォルダを作り、その場で権限を「自分・SYSTEM・Administrators だけ」に絞ります。エクスプローラーなどで手作りしたフォルダは権限が広いままになり、拒否されます。作ってしまった場合は、削除してからやり直してください。
2. MCP サーバーに、中継フォルダの場所を環境変数で教えて登録し直します。

   ```bat
   claude mcp remove premiere-pro --scope user
   ```

   ```bat
   claude mcp add premiere-pro --scope user -e "PREMIERE_TEMP_DIR=C:\ProgramData\premiere-mcp-bridge" -- cmd /c npx -y premiere-pro-mcp@1.21.0
   ```

3. MCP パネルの **Bridge directory** 欄に `C:\ProgramData\premiere-mcp-bridge` と入力して **Save** を押し、**Start Bridge** を押します。
4. Claude Code で新しいセッションを開き、接続を確かめます（[手順6](#6-claude-から接続を確かめる)）。

> [!NOTE]
> `C:\` の直下（例：`C:\premiere-mcp-bridge`）は使えません。そこに作ったフォルダでは、作った本人に「所有者を設定する権限」がないため、パッケージによる権限の設定が失敗します。

### 2. Git Bash で登録したら起動しない

Git Bash は `/c` のような引数を `C:/` に自動で書き換えます。`claude mcp add` の先頭に `MSYS_NO_PATHCONV=1 ` を付けるか、コマンドプロンプトで実行してください。

### 3. Claude に Premiere のツールが出てこない

- `claude mcp list` で `premiere-pro` が `✓ Connected` になっているか確認してください。
- MCP を登録・変更したあとは、Claude Code の**新しいセッション**を開く必要があります。

### 4. ツールを呼ぶと止まる、タイムアウトする

- MCP パネルが開いていて、**Running** になっているか確認してください。パネルを閉じると命令が実行されません。
- Premiere でプロジェクトを開き、シーケンスをアクティブにしてください。
- 中継フォルダを変えた場合は、MCP サーバー（`PREMIERE_TEMP_DIR`）とパネル（Bridge directory 欄）が**同じパス**になっているか確認してください。

---

## 日本語版 Premiere での注意

premiere-pro-mcp 1.21.0 と日本語版 Premiere Pro 2026 の組み合わせで、実際に使って分かったことです。

- **エフェクト名は日本語で指定します。** 例：「モーション」（スケール・位置）、「ボリューム」（レベル）。英語の `Motion` / `Volume` では見つかりません。
- **音量のツールがエラーになることがあります。** `set_clip_volume` / `get_clip_volume` が「オーディオクリップではない」というエラーになりました。代わりに、エフェクト「ボリューム」のプロパティ「レベル」を変更すると操作できます。
  - 「レベル」の内部の値は、`0.17783` が 0dB です。dB に直すと `20 × log10(値 ÷ 0.17783)` になります。
- **テキスト（グラフィック）の文言は、MCP から読み取れません。**
- **59.94fps などのシーケンスでは、`export_frame` が指定と違うコマを書き出すことがありました。**

このほか、Premiere のスクリプト API に用意されていない操作はできません。最新の制限事項は、[公式の README](https://github.com/leancoderkavy/premiere-pro-mcp) を確認してください。

---

## アンインストール

Premiere を完全に終了してから実行します。

```bat
npx -y premiere-pro-mcp@1.21.0 --uninstall-cep
```

```bat
claude mcp remove premiere-pro --scope user
```

- `--uninstall-cep` は、デバッグモードの設定（`PlayerDebugMode`）を残します。ほかの CEP 拡張でも使う設定なので、そのままで構いません。
- 中継フォルダを `C:\ProgramData\premiere-mcp-bridge` に変えていた場合は、そのフォルダも削除してください。

---

## 参考リンク

- [premiere-pro-mcp（GitHub）](https://github.com/leancoderkavy/premiere-pro-mcp)：本体。日本語の README もあります
- [公式サイトのセットアップガイド（英語）](https://premiere-pro-mcp.com/blog/how-to-set-up-premiere-pro-mcp/)
- [公式サイトのトラブルシューティング（英語）](https://premiere-pro-mcp.com/docs/troubleshooting/)
- [Claude Code の MCP の説明（Anthropic 公式）](https://code.claude.com/docs/ja/mcp)

---

このガイドは個人の動作記録です。内容は 2026年11月時点のもので、ソフトの更新によって変わることがあります。
premiere-pro-mcp は Adobe Inc. と関係のない有志のプロジェクトです。Adobe、Premiere Pro は Adobe Inc. の商標です。
