---
title: "Claude Code のデスクトップ操作、内蔵の computer-use ではなく Windows-MCP を使っている理由"
emoji: "🪟"
type: "tech"
topics: ["claudecode", "mcp", "windows", "uiautomation", "ai"]
published: true
---

> Claude Code にデスクトップ操作を任せる手段には、内蔵の computer-use があります。画面を撮ってピクセル座標でクリックするこの方式のまま任せ続けてよいのか気になって代わりを探し、UI Automation で画面を構造として読む Windows-MCP に入れ替えました。メモ帳への入力という同じタスクを両方に実行させて、操作の中身がどこで変わるのかを確かめてみました。

## TL;DR

- computer-use は操作のたびに「画面を撮る → 位置を推定する → 座標をクリックする」を繰り返します。Windows-MCP は UI Automation の要素ツリーを最初に1回読み、**要素を名前と座標で直接指定します**
- メモ帳にテキストを入力する同じタスクで、起動を除いた操作数は **computer-use が5、Windows-MCP が2** でした。減ったのは位置を知るための往復です
- 導入は `claude mcp add` の1コマンドですが、**日本語環境では `PYTHONUTF8=1` が必須**です。無いと cp932 のまま起動してサーバーが落ちます
- 万能ではありません。UI Automation の要素が取れないアプリでは名前指定の利点が消えます。手元では Claude Code のデスクトップアプリ自身がそうでした

## はじめに

Claude Code のデスクトップアプリに、コードの外の用事を頼む場面が増えてきました。アプリのウィンドウを撮って記事に貼る、GUI にしか出ない設定画面を確認する、といった細かい操作です。デスクトップアプリにはそのための computer-use という仕組みが内蔵されています。

仕組みを見ると、デスクトップのスクリーンショットを撮り、画像から目的の要素の位置を推定して、そのピクセル座標をクリックする、という流れを操作のたびに繰り返します。推定が外れれば隣の要素を押すことになり、画面が変われば撮り直しです。動きはするのですが、位置を毎回画像から当て直すこの前提のまま任せる範囲を広げてよいのかが引っかかり、Windows 向けの代替を探して Windows-MCP に行き着きました。以来、デスクトップ操作はこちらへ寄せています。

ただ、乗り換えの根拠を印象で終わらせたくなかったので、同じタスクを両方に実行させて操作の中身を並べました。先に結論を言うと、**差は要素を名前で指定できるかどうかに尽き、位置を知るための往復が消えるぶん操作数が減ります**。

## Windows-MCP とは

[Windows-MCP](https://github.com/CursorTouch/Windows-MCP) は Windows 専用のデスクトップ操作 MCP サーバーです。computer-use との違いは、画面を「画像」ではなく「構造」として読み取れる点にあります。

中心になるのは Snapshot というツールで、既定では Windows の UI Automation API を使って、画面上の要素を構造化されたツリーとして返します。実際にメモ帳から取れるツリーはこうです (メモ帳のウィンドウに関する部分の抜粋)。

```text
window "Untitled - Notepad"
├── (806,635) document "Text editor"  [action: scroll]  [focused]
├── (376,192) tab item "Untitled. Unmodified."  [action: click]
├── (199,232) menu item "File"  [action: click]
├── (262,232) menu item "Edit"  [action: click]
├── (331,232) menu item "View"  [action: click]
├── (762,232) button "Bold (Ctrl+B)"  [action: click]  [toggle:off]
├── (802,232) button "Italic (Ctrl+I)"  [action: click]  [toggle:off]
├── (937,232) button "Table"  [action: click]
└── (1416,179) button "Close"  [action: click]
```

要素が名前とロールと座標を持って並びます。テキストの入力先は `document "Text editor"` として名前で特定でき、座標もツリーと一緒に返るので、画像から位置を推定する工程がそもそも要りません。トグルボタンには `[toggle:off]` のように状態まで付きます。

操作系の Click や Type は、このツリーで得た座標や要素を指定して使います。ツリーが取れないアプリに備えて Snapshot にはスクリーンショットを付けて返すオプションもあり、その場合は computer-use と同じ画像ベースの操作に切り替わります。

## 同じタスクを両方で実行する

「メモ帳に `Hello from Claude Code` と入力する」という同じタスクを、同じマシンで両方に実行させました。どちらもメモ帳を開き直したまっさらな状態から、同じ内容の指示を出しています。メモ帳の起動はどちらにも専用の呼び出しがあるので数に入れず、起動後の操作を数えます。

### computer-use の場合

```text
screenshot
  → デスクトップの画像

left_click  coordinate=[585, 450]     → Clicked.
key         ctrl+a                    → Key pressed.
key         Delete                    → Key pressed.
type        "Hello from Claude Code"  → Typed (via clipboard).
```

![computer-use の実行ログ。open application と computer batch が並んでいる](/images/windows-mcp-vs-computer-use/computer-use-log.png =600x)
*実行ログ。畳んである computer batch がスクリーンショットの取得で、その下が座標クリックからの4アクション*

操作は5つです。まず画面を撮って入力欄の位置を画像から確かめ、その座標をクリックしてフォーカスを取り、それから入力します。位置を知るための往復が先に1回入るのが、そのまま操作数の差になります。

### Windows-MCP の場合

```text
Snapshot
  → UI ツリーを取得 (先ほどのツリーが返る)

Type  loc=[806, 635]  text="Hello from Claude Code"  clear=true
  → Typed Hello from Claude Code at (806,635).
```

![Windows-MCP の実行ログ。App、Snapshot、Type の3つの呼び出しが並んでいる](/images/windows-mcp-vs-computer-use/windows-mcp-log.png =600x)
*実行ログ。Snapshot の中身は先ほどのツリー、Type には座標とテキストがそのまま渡っている*

操作は2つです。ツリーに `document "Text editor"` があり、その座標が一緒に返るので、取得した値をそのまま入力先に指定できます。既存テキストの消去も `clear=true` で済み、全選択と削除のキー操作が要りません。

![メモ帳に Hello from Claude Code が入力された状態](/images/windows-mcp-vs-computer-use/notepad-result.png =600x)
*Windows-MCP から入力した結果*

### 比較

| | computer-use | Windows-MCP |
|---|---|---|
| 要素の特定 | 画面を撮って座標を推定 | UI Automation で名前指定 |
| 操作数 (起動を除く) | 5 | 2 |
| 方式 | ピクセルベース (vision) | UI Automation ベース (vision 併用可) |
| 対象 OS | 全 OS | Windows 専用 |
| 導入 | 不要 (内蔵) | 別途登録が必要 |

違いは要素を名前で直接指定できるかどうかの一点です。ボタンの位置が数ピクセルずれても要素名で特定できるので、ディスプレイのスケーリング設定が違う環境でも同じ手順が通ります。

## 導入手順

### 登録

```bash
claude mcp add windows-mcp -s user -e PYTHONUTF8=1 -- uvx windows-mcp serve
```

各オプションの意味です。

- `-s user`: ユーザースコープで登録。すべてのプロジェクトで使える
- `-e PYTHONUTF8=1`: 後述する文字コード問題の回避に必須
- `uvx windows-mcp serve`: PyPI から取得して起動。Python は uvx が必要なバージョンを自動で用意するので、手動で入れる必要はない

登録後は Claude Code の再起動が要ります。MCP サーバーはセッション起動時に読み込まれるためです。再起動後、ツール一覧に `mcp__windows-mcp__Snapshot` などが出れば導入完了です。

### 危険なツールの除外

Windows-MCP には PowerShell 実行やレジストリ操作のツールも含まれます。不要なら除外できます。

```bash
claude mcp add windows-mcp -s user -e PYTHONUTF8=1 -- uvx windows-mcp serve --exclude-tools PowerShell,Registry
```

逆に `--tools Snapshot,Click,Type` のように、使うものだけを明示する許可リスト形式もあります。

## 導入で詰まった所

動かせるようになるまでに2つ詰まりました。2つとも、出てくる症状が原因を素直に指していないタイプです。

### 1. PYTHONUTF8=1 を忘れるとクラッシュする

Windows の既定の文字エンコーディングは cp932 です。操作対象やツリーの中身とは関係なく、windows-mcp 自身が出力するヘルプやログの文字列にダッシュなど cp932 で表現できない文字が含まれていて、`UnicodeEncodeError` でサーバーが落ちます。何かを操作する前の話で、`--help` を表示するだけでも再現します。

```text
UnicodeEncodeError: 'cp932' codec can't encode character '—'
```

`PYTHONUTF8=1` を渡すと Python が常に UTF-8 を使うようになり、回避できます。上の登録コマンドには含めてあります。

:::message
既定のコードページが cp932 になっている日本語環境の問題です。英語環境では遭遇しないことがあります。
:::

### 2. 初回起動が接続タイムアウトに当たる

こちらの方が原因に辿り着きにくいです。症状は「サーバーは接続済みなのにツールが1つも見つからない」で、ツール検索をかけても空振りします。

MCP のログを見ると実態が分かります。

```text
Server stderr: Installed 92 packages in 12.21s
Connection failed after 30041ms (CONNECT_TIMEOUT): Request timed out
```

初回は uvx がパッケージの取得から始めるため、起動完了が接続タイムアウトの30秒に間に合いません。サーバー側は起動しているのにハンドシェイクが成立せず、ツールが登録されないままになります。

先に一度起動してキャッシュを作っておけば回避できます。

```bash
uvx windows-mcp serve --help
```

ログは `%LOCALAPPDATA%\claude-cli-nodejs\Cache\<プロジェクト>\mcp-logs-windows-mcp\` に残ります。ツールが見つからないときは、まずここで `CONNECT_TIMEOUT` の有無を確認してください。

## 実際に動かして分かったこと

操作数の比較そのものより、回している途中で見えた挙動の方に発見がありました。ほとんどは computer-use 側の話です。

computer-use は、Claude 側のウィンドウがキーボードフォーカスを持っている間はキー入力を拒否します。Windows-MCP でメモ帳を前面に出しても解消せず、computer-use 自身のクリックでフォーカスを取る必要がありました。自分がフォーカスを持ったまま入力すると、意図しない先に文字が飛ぶためだと思われます。

操作許可の承認カードは Claude Code のウィンドウ内に出ます。最大化した別のウィンドウで覆っていると気付けないまま約5分でタイムアウトし、拒否として返ります。操作していないのに拒否になるので紛らわしいのですが、ログに `still running (N s elapsed)` が積まれていれば、それは即時拒否ではなく応答待ちの満了です。

文字入力はクリップボード経由で、戻り値が `Typed (via clipboard)` になります。実行後に内容は元へ戻っていましたが、共有リソースを一時的に書き換える点は把握しておいた方がよいです。

スクリーンショットには、許可したアプリ以外を伏せる機能もあります。メモ帳だけを許可した状態で撮ると、背後の他アプリはデスクトップ背景に置き換わりました。作業中の画面を写さずに記録を残したいときに使えます。

並べてみると、computer-use は許可まわりの設計が細かいことが分かります。操作できるアプリを1つずつ承認させ、スクリーンショットでは許可外を伏せる、という作りです。Windows-MCP にこうしたアプリ単位の縛りはなく、絞れるのは `--exclude-tools` によるツール単位の除外だけです。操作の速さと自由度を取った設計なので、何をどこまで任せるかは使う側が決めることになります。

## 使い分け

| 場面 | 使うもの |
|---|---|
| Windows のネイティブアプリ | Windows-MCP |
| ブラウザ | Playwright MCP |
| UI Automation の要素が取れないアプリ | computer-use か、Windows-MCP の vision オプション |
| アプリ単位で許可を縛りたい作業 | computer-use |

Windows でネイティブアプリを操作するなら Windows-MCP が向いています。ブラウザ操作はアクセシビリティツリーで要素を扱える Playwright MCP の方が堅牢なので、併用するのが現実的です。

ただし Windows-MCP も万能ではありません。UI Automation の要素が取れないアプリでは名前指定の利点が消えます。手元では Claude Code のデスクトップアプリ自身がこれに当たり、Snapshot のツリーに要素が1つも出ませんでした。その場合はスクリーンショット方式に戻ることになります。

## おわりに

Windows-MCP に寄せてから、デスクトップ操作で computer-use を使うことはほぼなくなりました。同じタスクの操作数が5から2に減ったのは、要素を名前で指定できるぶん、位置を知るための往復が消えたからです。数字にすると数手の差ですが、ウィンドウを撮らせる、設定画面を開かせる、対話プロンプトの選択肢を押させるといった細かい用事を気軽に投げられるようになったのが、体感では一番の違いでした。いまは Windows のネイティブアプリは Windows-MCP、ブラウザは Playwright、という分担に落ち着いています。

この2ヶ月、運用していて特に困ったことはありません。操作数が減ればスクリーンショットの受け渡しやモデルとの往復も減るので、トークンの消費も抑えられているはずです。困りごとがないまま消費だけ軽くなるのは、使っていて少し気分がいいです。

（動作確認: Windows 11 の Claude Code デスクトップアプリ + windows-mcp (PyPI 版、uvx 経由)。実行ログは 2026-08-20 時点のものです）
