---
title: "Claudeに書いてもらったノートを外出中のiPhoneで読み返せるObsidian + git構成(Working Copy)"
emoji: "📓"
type: "tech"
topics: ["obsidian", "git", "workingcopy", "ios", "claudecode"]
published: true
---

## TL;DR

- Claude Code に書いてもらった markdown ノートを、外出中の iPhone でも読み返せるようにした構成の話
- 同期エンジンは Linux 常駐機の **systemd user timer**(10 分周期で `git pull --rebase + git push`、commit は別途 Claude Code 側で実行)、Windows は Obsidian + Obsidian Git plugin、iPhone は **Working Copy + Obsidian iOS**
- iPhone 側が一番のハマりどころ。Obsidian iOS には「既存 vault を開く」入口が無く、**Working Copy の `Link external repository → Directory`** で空 vault フォルダに repo を後付け Link して抜ける
- Working Copy で `Link external repository` と push を使うには Pro IAP(¥4500 の買い切り、本記事執筆時点 2026年5月)が必要。7 日間の trial があるので、まず trial で構成全体を試してから判断するのが手堅い
- Obsidian Sync(月$4 / 年払い)を契約すれば最小手間で組めます。本記事は **vault を Claude Code(CLI agent)の workspace としてそのまま git で扱いたい / commit history と branch を運用に組み込みたい / Sync subscription に縛られたくない** といった条件で git ベースを選んだ場合の話

## はじめに

Claude Code に頼んで学習メモやリサーチをまとめてもらうと、自分の vault ディレクトリ(私の場合は `~/notes`)に markdown が溜まっていきます。家のデスクで読み返すには十分ですが、**通勤電車や移動中に「あの件、Claude にメモらせたっけ?」と思ったとき**に、その markdown を iPhone でも読み返せると体感がぐっと良くなります。

この sync は Obsidian Sync(月$4 / 年払い)で一発で組めます。2026年2月から open beta として提供されている **Obsidian Headless**(Sync subscription 必須、Node.js 22+ の CLI 専用クライアント、GUI 機能は持たず sync だけを担当)を使えば server 側で sync 専任プロセスを回す選択肢もあります(本記事執筆時点では beta 段階)。

本記事は **vault を Claude Code(CLI agent)の workspace としてそのまま扱いたい**、**git の commit history と branch を vault 運用に取り込みたい**、**Sync subscription に縛られたくない** といった条件があったので、Obsidian Sync ではなく **private GitHub repo + git** で自前構成してみた話です。コスト削減が目的ではなく、**Git workflow と agent workflow の親和性** を取りに行ったのが動機です。

想定読者:

- Claude Code 等で markdown を生成して個人ノートに溜めているが、外出中の iPhone でも読み返したい人
- vault を Claude Code の workspace と兼ねて、commit message / branch / revert を運用に組み込みたい人
- iPhone で Obsidian と Working Copy の連携で詰まった、または詰まりそうな人

## 現在の運用

vault は Claude Code が生成した markdown(学習ノート、調査メモ、読書メモ等)を集約する個人ナレッジベースです。3 台のデバイスがそれぞれ違う役割を持ちます。

| デバイス | 役割 |
|---|---|
| **Linux 常駐機** | vault の sync 実行端末。Claude Code から markdown を生成して `learning/`, `inbox/` 配下に流し込む経路 |
| **Windows PC** | Obsidian デスクトップで主力編集。GUI でリッチに整理する場 |
| **iPhone** | 外出中の閲覧と軽編集。Obsidian iOS + Working Copy |

3 台が独立して動き、git で結ばれています(remote は GitHub の private repo)。

## git ベースを選んだ理由

Obsidian Sync(または 2026年2月から open beta 提供されている Obsidian Headless)で済むなら、最小手間で組めるのはそちらです。本構成は以下の条件があったので git ベースにしています。

- **Git workflow と agent workflow の親和性**: Claude Code(CLI agent)が vault `~/notes` をそのまま workspace として扱える。検索・編集・diff・commit が一連の git 操作で回り、Obsidian アプリを介さずに完結する
- **意味単位の commit history(Linux / Claude Code 経由の編集分)**: Linux 側で Claude Code から書く分は、編集の度に意味単位の commit を作る運用にすれば編集意図の履歴が手元に残る。Windows / iPhone から GUI で書き換える分は Obsidian Git plugin の同期 commit(`vault: hostname auto sync` 系)が混じるので、history は両者の混在になる
- **branch / review / revert が効く**: 過去の調査メモを branch 単位で隔離する、書きかけの記事 draft を branch で並行管理する、誤って消した markdown を `git checkout` で戻す、のような運用ができる
- **Sync subscription 不要 / vendor lock-in 薄い**: GitHub private repo を remote にしているが、self-host や NAS の bare repo にも切り替えられる。Sync 前提の運用に縛られない
- **既存 homelab の流用**: 家にすでに Linux 常駐機がある環境では、その homelab を vault の sync 補助点としてそのまま使える(差別化軸ではないが、本記事の前提条件)
- **機微情報(キャリア・収入・健康・第三者実名)は vault に書かない方針**: 3 台に分散して残るので、漏洩リスクの面積を意識的に小さくしておく(別の private 領域に隔離する)

逆に、**Git 構成の弱点** も書いておきます。Obsidian Sync は **リアルタイム同期 / モバイル統合 / 非技術者向けの UX** で本構成より優位です。10 分周期の git timer では同期遅延が出ますし、conflict 時は手で解消するなど git の運用知識を要求します。技術記事を書く / agent と暮らす想定の人には git のほうが手に馴染みますが、ただメモを 3 端末で見たいだけなら Sync のほうが向きます。

### コスト感

差別化軸ではないですが補助情報として(本記事執筆時点 2026年5月の概算):

| 構成 | subscription | 一回購入 |
|---|---|---|
| Obsidian Sync | 月$4(年払い) / 月$5(月払い) | — |
| 本構成(Git remote 流用) | — | Working Copy Pro IAP ¥4,500(iPhone 連携で必須) |

5 年スパンなら本構成のほうが累計は安いですが、コスト目当てで切り替えるほどの差ではないです。

## Linux 側

最初に GitHub に private repo (`<user>/notes`) を作って、Linux 側で初期化と systemd user timer での auto sync を仕込みます。Linux は **vault の sync 実行端末**(常駐の git client + Claude Code の workspace)で、remote はあくまで GitHub です。

:::message
**手元に Linux 常駐機が無い人へ**: ここから「Linux 側」セクションは丸ごと読み飛ばして OK です。Mac なら `launchd`、Windows しか無ければ Obsidian Git plugin で同等のことができるので、その場合は「Windows 側」「iPhone 側」だけ拾ってください。
:::

### vault 雛形

vault 直下のディレクトリ構成:

```
notes/
  daily/
  inbox/
  tech/
  learning/
  archive/
  attachments/
  templates/
```

`.gitignore`(Obsidian の workspace 系を除外):

```
.obsidian/workspace*
.obsidian/cache
.obsidian/graph.json
.trash/
```

`.gitattributes`(改行 LF 統一 + 添付バイナリを Git LFS で track):

```
* text=auto eol=lf

*.png  filter=lfs diff=lfs merge=lfs -text
*.jpg  filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.gif  filter=lfs diff=lfs merge=lfs -text
*.webp filter=lfs diff=lfs merge=lfs -text
*.svg  filter=lfs diff=lfs merge=lfs -text
*.pdf  filter=lfs diff=lfs merge=lfs -text
*.mp4  filter=lfs diff=lfs merge=lfs -text
*.mov  filter=lfs diff=lfs merge=lfs -text
*.mp3  filter=lfs diff=lfs merge=lfs -text
*.zip  filter=lfs diff=lfs merge=lfs -text
```

```bash
cd ~/notes
git init -b main
git lfs install
git add .gitignore .gitattributes
git commit -m "init: vault skeleton with Git LFS for binary attachments"
git remote add origin git@github.com:<user>/notes.git
git push -u origin main
```

### Linux 側 auto sync(systemd user timer)

cron でも同じことができますが、headless 常駐機では systemd user timer (Linux 標準のスケジューラ)のほうが、cron に比べてログ・取りこぼし吸収・user service 管理が扱いやすいので、これを使います。

- ログが `journalctl --user -u notes-sync.service` で構造化されて見える
- `Persistent=true` でサーバーが落ちていた時間を後追いで実行(取りこぼしを 1 回吸収)
- ユーザーレベルの常駐サービス管理を systemd 側に寄せると他の設定との一貫性が出る

`~/.config/systemd/user/notes-sync.service`:

```ini
[Unit]
Description=Sync ~/notes via git pull --rebase + git push
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=%h/notes
ExecStart=/bin/sh -c '/usr/bin/git pull --rebase && /usr/bin/git push'
```

ここで重要なのは、**`ExecStart` には `git add` / `git commit` を含めていない** こと。**commit は意味単位で人間 / Claude Code 側で作る前提なので、自動 timer には任せていません**。Claude Code が markdown を生成・編集したら、そのセッション内で `git add ... && git commit -m "..."` まで明示的に行う運用にしておきます。timer はあくまで「コミット済みの内容を 10 分周期で push、リモートの変更を pull する」sync 専用です。

未コミット変更を溜めたままにすると `git pull --rebase` が失敗して timer が止まる(後述「運用上の注意」で再掲)ので、commit は溜め込まない方針が安全です。

`~/.config/systemd/user/notes-sync.timer`:

```ini
[Unit]
Description=Periodic sync of ~/notes (every 10 min, offset by 5)

[Timer]
OnCalendar=*:05/10
Persistent=true
Unit=notes-sync.service

[Install]
WantedBy=timers.target
```

`OnCalendar=*:05/10` は :05, :15, :25, … で発火します。Windows 側 Obsidian Git の発火が :00 系に当たりやすいので、5 分ずらしておくと衝突しにくくなります(後述「衝突を避ける時刻オフセット」)。

有効化:

```bash
systemctl --user daemon-reload
systemctl --user enable --now notes-sync.timer

# SSH 切断後・再起動後ログイン前も継続実行されるよう linger を有効化
sudo loginctl enable-linger $USER
```

`enable-linger` (= ユーザーがログアウトしてもユーザーサービスを動かし続ける設定) を入れないと、ユーザーがログインしている間しか user service が動きません。ヘッドレス常駐機ではこれが必須です。

ログ確認:

```bash
# 直近の実行ログ
journalctl --user -u notes-sync.service -n 50

# tail -f 相当
journalctl --user -u notes-sync.service -f

# 次回発火タイミングと最終発火を確認
systemctl --user list-timers notes-sync.timer
```

### シンプル派の代替: cron

systemd を使わず cron で済ませるなら以下で同等:

```cron
5-59/10 * * * * (cd ~/notes && git pull --rebase && git push) 2>&1 | logger -t notes-sync
```

`*/10`(:00 系発火)ではなく `5-59/10`(:05 開始)が肝です。`Persistent=true` 相当の取りこぼし吸収は cron では効かないので、再起動直後の取りこぼしが気になるなら systemd 側を選んだほうが楽です。

## Windows 側

PowerShell でツール準備、SSH 疎通確認、clone、Obsidian インストール、プラグイン設定の順で進めます。

### ツール準備

```powershell
winget install --id Obsidian.Obsidian
winget install --id Git.Git
winget install --id GitHub.GitLFS
```

(既存ならスキップ可。)

### SSH 鍵

GitHub に Windows 側用の SSH 公開鍵(ed25519 推奨)を登録しておきます。疎通確認:

```powershell
ssh -T git@github.com
# Hi <user>! You've successfully authenticated, but GitHub does not provide shell access.
```

### vault clone

```powershell
cd $env:USERPROFILE
git clone git@github.com:<user>/notes.git
cd notes
git lfs install
git lfs pull
```

### Obsidian で vault open

- Obsidian 起動 → 「Open folder as vault」→ `C:\Users\<user>\notes` を選択
- 「Trust author and enable plugins」のダイアログで **Trust author**

### プラグイン設定

**Core plugins**(Settings → Core plugins):

- **Daily notes** を ON
  - `New file location` = `daily`

**Community plugins**(Settings → Community plugins):

- 「Turn on community plugins」で Restricted mode 解除
- Browse から install + Enable:
  - **Templater**(SilentVoid)
  - **Obsidian Git**(Vinzent03)

#### Obsidian Git plugin の auto sync が要る理由

Linux で書いた markdown を Windows で見るには、**Windows 側の clone を `git pull` で更新する必要があります**。Obsidian は file watch でローカルファイルの変更は拾いますが、**GitHub 側の更新は誰かが pull しない限りローカルに降りてきません**。

なので、Windows でも Obsidian Git plugin を enable して以下の auto sync 設定を入れるのが推奨です(読むだけの運用でも auto pull のため省略不可)。Windows でも markdown を書き換える運用なら、auto pull に加えて auto commit / auto push も同じ plugin で回ります。

| 項目 | 値 |
|---|---|
| Auto commit-and-sync interval (minutes) | `10` |
| Auto pull interval (minutes) | `10` |
| Pull updates on startup | ON |
| Commit message on auto commit-and-sync | `vault: {{date}} {{hostname}}` |

旧バージョンの Obsidian Git では「Vault backup interval (minutes)」「Commit message」というラベルでしたが、現バージョン(本記事執筆時点)では **「Auto commit-and-sync interval」「Commit message on auto commit-and-sync」** に改名されています。古い記事や README を見ながら設定すると文字列が見つからずに迷うので注意です。

`Commit message on auto commit-and-sync` と `Commit message on manual commit` は別フィールドなので、auto 側に入れるのを間違えないようにします。

### .obsidian/ を commit + push

プラグインを enable すると `.obsidian/community-plugins.json` 等が更新されます:

```powershell
cd $env:USERPROFILE\notes
git add .obsidian/
git commit -m "obsidian: enable Daily Notes / Templater / Obsidian Git"
git push
```

以後は Obsidian Git plugin の auto sync 任せで、手動 commit はほぼ不要になります。

## iPhone 側

iOS は Working Copy で repo を持って、Obsidian iOS は同じフォルダを vault として読む構成です。本記事の本題はここの繋ぎ込みです。

### Working Copy で clone

App Store から Working Copy をインストール後、まず Working Copy の `Settings → Keys` で **SSH 鍵を Generate / Import** するか、`Repositories → Clone` 時に **HTTPS + GitHub Personal Access Token** を保存。そのうえで GitHub にログイン、`<user>/notes` を clone。

**料金について**: Working Copy で **`Link external repository`(後述)と push** を使うには **Pro IAP** が必要です(¥4500 の買い切り、2020 年 9 月以降の Pro 購入で解放、本記事執筆時点 2026年5月)。7 日間の trial があるので、まず trial で構成全体を試してから IAP の判断をすると手堅いです。trial 終了後は無料機能(clone と pull のみ)に戻り、push や Link external repository は止まる想定です。

### Obsidian iOS は「既存 vault を開く」が無い

ここがハマりポイントです。Obsidian iOS を起動すると vault の保存先選択画面が出るのですが、選択肢は **Obsidian Sync / iCloud / Other** の 3 つしかありません。「Other」を選んでも、出るのは:

> To use an alternate syncing methods, create a vault and follow the instructions provided by the plugin or third-party sync provider.

という案内文だけで、**他 App(Working Copy)が clone したフォルダを「既存の vault として開く」直接の道は無い**。

### 解決: Working Copy 側から Obsidian の vault フォルダへ Link

最終的に動いた手順:

1. **Obsidian iOS** で vault `notes` を「**On my iPhone**」に新規作成
   - iCloud は選ばない(Working Copy との二重管理になる)
   - これで `Files App → On my iPhone → Obsidian → notes/` に空フォルダが生成される
2. **Working Copy** でリポジトリ操作メニュー(右上のステータス画面)を開き、「**Link external repository → Directory**」を辿る(公式 docs では `Link Repository to Folder` 表記。検索時はそちらでも引ける)

![Working Copy の Link external repository メニュー、Directory と Document Package の選択肢](/images/working_copy_link_directory.jpg)

3. ファイル選択画面で **`On my iPhone/Obsidian/notes`**(手順 1 で Obsidian iOS が作った空フォルダ)を指定
4. Working Copy がリポジトリの中身をそのフォルダに展開する

これで Working Copy の git working directory と Obsidian iOS の vault が **同一の物理フォルダ**を指すようになります。Obsidian iOS で vault を開けば、Linux / Windows でこれまでに書いたノート、`.obsidian/` 配下のプラグイン設定がすべて反映された状態で見えます。

### Add Remote: Link 直後は origin が消えているので手動で戻す

ここがもう一段ハマりポイントで、**「Link external repository → Directory」だと git remote の設定が引き継がれません**。Working Copy 上の repo に対して pull / push しようとすると「リモートが無い」エラーになります。

Working Copy のリポジトリ設定画面(右上の歯車のような Status アイコン)を開くと、`Remotes` セクションが空になっているか、`origin` が外れています。`Add Remote` ボタンから手動で `origin` を再追加します。下のスクリーンショットは **本記事用に origin を再追加した後の状態**(`Add Remote` ボタンの上に origin 行が見える)で、Link 直後はこの origin 行が無い状態になっているはずです。

![Working Copy のリポジトリ設定画面、Remotes セクションの origin 行と Add Remote ボタン(本記事用に origin 再追加後の状態)](/images/working_copy_repo_config_masked.jpg)

`Add Remote` をタップすると `New Remote` 入力画面に遷移します:

![Working Copy の New Remote 入力画面、Name / URL / Allow Fetch / Allow Push の入力欄と Save ボタン](/images/working_copy_add_remote_form.jpg)

| 項目 | 値 |
|---|---|
| Name | `origin` |
| URL | `git@github.com:<user>/notes.git` |
| Allow Fetch | ON |
| Allow Push | ON(Working Copy Pro IAP / trial 期間中のみ有効) |

入れ終わったら右上の `Save` で確定。

### 同期操作(pull / push)

Add Remote まで終われば、リポジトリ画面右上のメニューから `Pull` / `Push` が打てるようになります。

![Working Copy の同期操作メニュー、Commit / Revert / Merge / Fetch / Pull / Push の各項目](/images/working_copy_sync_menu.jpg)

iOS は Background Refresh の制約で頻繁な auto sync が信頼できないので、**Working Copy アプリを開いた時に手動で `Pull` するのが現実解** です。書き換えがあれば `Commit → Push`(無料版は push 不可、trial / Pro が必要)。

### iPhone 側の注意点

- vault 作成時に Obsidian iOS が自動生成する `.obsidian/` と、リポジトリ側の `.obsidian/` が競合する可能性があります。Link 時に上書き確認が出たらリポジトリ側を優先する
- iOS でも Obsidian Git plugin の install と enable はできますが、Background 同期と git 実行環境の iOS 側制約で **信頼できる auto sync は期待できない** ので、git 操作は Working Copy 側に寄せる前提で組んだほうが現実的です
- iPhone から push する場合は Working Copy Pro が必要(または trial 期間中)。私は push を Linux / Windows 側に任せて iPhone は Pull だけの読み返し運用にしています

## 衝突を避ける時刻オフセット

Windows と Linux を素直に同じ周期(:00 系発火)で動かしていると、同時刻に push しに行って一方が失敗、次の周期まで反映が遅れることがあります。git の rebase でリトライ自体は吸収できますが、無駄な失敗を減らすために **周期は 10 分のまま、Linux 側だけ 5 分オフセット**(:05 開始)にしています。

| デバイス | 周期 | 発火タイミング |
|---|---|---|
| Windows (Obsidian Git) | 10 分 | plugin の interval ベース(:00 系に当たりやすい) |
| Linux (systemd timer) | 10 分 | `*:05/10` で :05 開始 |
| iOS (Working Copy) | 手動 / Background Refresh | Pro なら push 自動化可、無料は手動 |

iPhone は Background 制約で頻繁な auto sync が信頼できないので、Working Copy アプリを開いた時に手動 pull / push する運用が現実的です。

### それでも衝突したら

オフセットでも衝突は起きうるので、起きたときの直し方:

- 失敗ログを `journalctl --user -u notes-sync.service` で確認、どの周期で sync が止まったかを把握
- 失敗した端末で `git pull --rebase` → 競合を解消(同一ファイルの並行編集は手動解消が必要)→ `git push`
- 同じファイルを 2 台以上で同時編集していた場合は **正とする端末を決めて** 他端末を pull --rebase で寄せる

systemd timer / Obsidian Git plugin / Working Copy はいずれも基本的に **conflict で sync を止める** だけで vault 自体は壊れにくい構造ですが、Working Copy の Link 時の上書き、`.obsidian/` の同時編集、複数端末で同じファイルを並行 rename したケースでは実害が出得ます。ログを見て直すのが基本ですが、上書き確認と競合解消は慎重に。

## 運用上の注意

- **機微情報**(キャリア・収入・健康・第三者実名)は vault に書かない。別の private 領域(Git 管理外のローカルディレクトリ)に隔離する。3 台に分散して残るぶん、漏洩リスクの面積を意識する
- vault のディレクトリ構造を大きく変えるときは、各デバイスの auto sync を一旦止めてから揃える(動いたまま rename すると衝突が連鎖しやすい)
- **未コミット変更を溜めない**: Linux 側 timer の `git pull --rebase` は未コミット変更があると失敗して止まる。Claude Code セッションで markdown を編集したら、そのセッション内で commit まで完了させる運用にしておく
- **force push は使わない**: 複数端末から push が走るので、history を書き換えると他端末の rebase が壊れる。`git push --force` 系は意図的に避ける
- **conflict 時は timer / plugin が止まる**: 自動 sync が止まっているのが「気付いてほしいシグナル」なので、ログを見る習慣を付けておく(止まったまま放置するとどんどん遅延が積み上がる)

## おわりに

できあがった構成では、家のデスクで Claude Code が書いた markdown を、外出先で iPhone の Working Copy を開いて Pull → Obsidian iOS で開く、の 2 ステップで読み返せるようになります。通勤電車で「先週 Claude にまとめてもらった〇〇の話、何だったっけ」と思ったときに vault 全体を検索できるのは、思っていた以上に便利でした。

書き換え先は GUI(Windows / iPhone Obsidian)と CLI(Linux / Claude Code)が混ざります。多くの衝突は git rebase で整理できますが、同じファイルを並行編集した場合は手動の競合解消が必要です。それでも git の commit history と branch が手元に残るのは、agent と一緒にメモを書いていく上で素直に効きます。

iPhone 連携は Obsidian iOS の仕様(「Open existing vault」が無い)と、Working Copy の Link 後 remote が消える挙動の 2 段で詰まりますが、本記事の手順を踏めば 1 本道で抜けられます。Working Copy Pro は trial があるので、まず試してみて自分のワークフローに合うか判断すると手堅いです。

Obsidian Sync(または 2026年2月から open beta の Obsidian Headless)は **リアルタイム同期 / モバイル統合 / 非技術者向け UX** で本構成より優位です。3 台でメモを見たいだけなら Sync のほうが向きますし、git の運用知識を要求しない分のメリットも大きいです。本構成は **vault を Claude Code の workspace としてそのまま扱いたい / commit history と branch を運用に組み込みたい** 人向けに、構成と詰まりどころを書き残しました。同じ場所で詰まる人の参考になれば。
