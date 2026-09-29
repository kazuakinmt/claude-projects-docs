# 記事資料をGitHubで共有する仕組み

Blogger・note・Facebookの記事で参照する資料(仕様書、ロードマップ、ルート情報など)を、本文に埋め込まずにこのリポジトリ `shared-docs` に置き、記事からURLでリンクしています。記事は読みやすく、資料は後から更新しやすくするのが狙いです。

資料の公開は、VSCodeの拡張機能 Front Matter CMS のボタンから行います。公開前に機密情報の検査を必ず通す作りにしています。

| 項目 | 内容 |
| --- | --- |
| 公開先 | [kazuakinmt/shared-docs](https://github.com/kazuakinmt/shared-docs)(Public) |
| 原稿の管理 | VSCode + Front Matter CMS(記事ごとのフォルダ) |
| 作業ツール | Python、git、gh CLI、Claude Code |
| コミット用メール | GitHubのnoreplyアドレス(このリポジトリのみ設定) |

## 全体の流れ

```
記事フォルダ\
├── fb.md / blog.md / note.md    ← 各媒体の原稿
└── docs\                        ← 公開したい資料だけを置く
    ├── docs.yaml                ← 公開先・公開するファイルの指定
    └── guide.md など
          │
          │ ボタン1「資料チェック」(読み取りのみ)
          │ ボタン2「資料を公開(GitHub)」
          ▼
shared-docs\<カテゴリ>\<スラッグ>\    ← コピー → commit → push
          │
          ▼
docs.yaml に公開URLを書き戻し → 各媒体の記事に挿入
```

## docs.yaml の書き方

```yaml
category: claude           # claude / cycling / travel
slug: docs-repo-setup      # 英小文字・数字・ハイフン(URLになる)
title: 資料公開の仕組み      # 資料一覧に表示する名前
summary: 構築手順と運用ルール
files:                     # 公開するファイル。ここに書いたものだけが公開される
  - path: guide.md
    label: 構築・運用ガイド
insert:                    # 媒体ごとにリンクを入れるか
  blogger: true
  note: true
  facebook: true
```

公開後は、スクリプトが末尾に `published`(URL・日時・コミットID)を書き足します。

## ボタン1:資料チェック

shared-docsには何も書き込まず、次の検査結果を表示します。エラーが1件でもあれば「公開不可」です。

- docs.yaml の必須項目・カテゴリ・スラッグの形式
- ファイルの拡張子(md・pdf・画像)とサイズ(10MB超は警告、50MB超はエラー)
- md内で参照している画像が存在し、公開対象に含まれているか
- 機密情報:ローカルのフルパス、IPアドレス・ホスト名、メールアドレス、APIキー・トークンの形式
- PDF:作成者などのメタデータ(公開時に削除)、文字を抽出できないPDFは目視確認を促す
- 画像:GPS位置情報が自宅など登録した「非公開地点」の範囲内なら公開不可(それ以外の位置情報は残す)
- 公開先で新規・更新・変更なしになるファイルの予告

検出した値や座標は、画面上でも伏せ字にします。チェック結果をチャットなどに貼って共有することがあるためです。

## ボタン2:資料を公開(GitHub)

このボタンを押すことを「チェック結果の承認」とみなします。確認ダイアログは出ない代わりに、次の条件がそろわなければ中止します。

1. チェック結果があり、チェック後にファイルが1つも変わっていない(ハッシュで照合)
2. 公開先が `kazuakinmt/shared-docs` の main ブランチで、未コミットの変更がない
3. リモートとの分岐がない(`git pull --ff-only` で最新にしてから作業)
4. コミットの作成者・コミッターがnoreplyアドレス

問題がなければ、PDFのメタデータを削除してコピーし、READMEの「資料一覧」を更新して commit → push します。途中で失敗した場合は、触ったファイルだけを元に戻します。pushだけ失敗した場合は、次回ボタンを押したときにpushだけをやり直します。

## 公開前に人が確認すること

スクリプトで検査できない部分は、目で確認します。一度pushした内容は、削除してもGitHubの履歴に残るためです。

- [ ] 地図やルートの画像・PDFに、自宅など伏せたい場所が写っていない
- [ ] 家族・知人など個人が特定できる情報がない
- [ ] 口座・保有資産に関する情報がない
- [ ] 他者が作成したPDFや画像を再配布していない(元のURLへのリンクにする)

## 初回セットアップ(Windows)

1. gh CLIをインストール:`winget install --id GitHub.cli`(止まって見えたら、タスクバーに隠れた管理者権限の確認を探す)
2. PowerShellとVSCodeを開き直してから、別のPowerShellで `gh auth login`(GitHub.com → HTTPS → Yes → Login with a web browser)
3. GitHubの Settings → Emails で「Keep my email addresses private」をOn にし、noreplyアドレスを控える
4. クラウド同期の対象外のフォルダに `git clone https://github.com/kazuakinmt/shared-docs.git`
5. cloneしたフォルダで `git config --local user.email "<noreplyアドレス>"`
6. PDF処理用に `python -m pip install pypdf`

## つまずいた点

| 状況 | 原因 | 対処 |
| --- | --- | --- |
| gitリポジトリをクラウド同期フォルダに置いた | 同期ソフトが `.git` 内を同期し、壊れることがある | 同期対象外のフォルダに置く |
| インストール後に `gh` が見つからない | 起動済みのターミナルは古いPATHのまま | すべて閉じて開き直す |
| `gh auth login` を操作できない | 矢印キーで選ぶ対話形式で、Claude Code内では動きにくい | 別のPowerShellで実行する |
| Front Matterのボタンで確認入力ができない | ボタンは対話入力を受け付けない | 「チェック」と「公開」の2つのボタンに分ける |
| 資料フォルダのmdが記事一覧に出る | Front Matterは記事フォルダの中まで探す | 設定の除外パスに `docs/**` を追加 |
