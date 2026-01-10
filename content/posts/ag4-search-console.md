+++
date = '2025-12-29T01:02:30+09:00'
draft = false
title = 'HugoでGA4とSearch Consoleを導入'
description = 'Hugoでブログを公開した後に必須となるGA4とGoogle Search Consoleの導入手順。設定ファイルへの追記方法からDNS設定、運用初期の注意点まで解説します。'
tags = ['GA4', 'GSC', 'Netlify', 'Hugo']
+++

Hugoでブログを公開したら、最初に行うべきGA4（Googleアナリティクス）とGoogle Search Consoleの導入手順をまとめます。 
解析ツールを早期に導入することで、誰がどんな言葉でサイトに訪れたのか把握することができます。

今回はNetlifyにデプロイしている環境を前提に解説します。

## Google Analytics (GA4) の設定

サイト内の行動データ（PV数や滞在時間など）を計測するための設定です。

### 測定IDの取得
1. [Googleアナリティクス](https://analytics.google.com/analytics/web/)でプロパティを作成
2. 「データストリーム」からウェブを選択し、サイトURLを登録
3. 発行された 「測定ID（G-XXXXXXXXXX）」 をコピー

### Hugoへ組み込み
多くのHugoテーマではGA4用のテンプレートが内蔵されています。設定ファイルとヘッドタグを編集するだけで完了します。

hugo.toml（または config.toml）のルート階層にIDを追記します。

```toml
googleAnalytics = 'G-XXXXXXXXXX'
```

layouts/partials/head.html内に、以下の1行を追加します。

```html
{{- template "_internal/google_analytics.html" . -}}
```

テーマによっては、すでに「google_analytics.html」を含むコードがあるのでそれは削除してください。

### 運用時の注意点

設定直後は「リアルタイムレポート」で自分のアクセスがカウントされているか確認します。
確認後は、正確なデータを取るために「内部トラフィックの除外」設定を行うことを推奨します。

## Google Search Console の設定
検索エンジンのパフォーマンスを管理する設定です。読者がどのようなキーワードで検索して辿り着いたかを分析するのに役立ちます。

### ドメインプロパティの登録

1. [Search Console](https://search.google.com/search-console/about?authuser=1)で「ドメイン」を選択し、設定したいURLのドメインを入力
2. 所有権確認用の TXTレコード（google-site-verification=...）をコピー

### 所有権の確認（DNS設定）
Netlifyでドメイン管理を行っている場合、TXTレコードによる確認がスムーズです。

1. Search Consoleで「ドメイン」プロパティを選択し、URLを入力します。
2. 表示された所有権確認用のTXTレコード（google-site-verification=...）をコピーします。
3. Netlifyの Domain management > DNS panel を開き、「Add new record」をクリックして以下を入力
- Type: TXT
- Name: @
- Value: 先ほどSearch ConsoleでコピーしたTXTレコード
3. 保存後、Search Consoleで「確認」ボタンを押して完了

## データの反映待ちについて
設定がすべて完了しても、データが反映されるまでには少し時間がかかります。

### Google Analytics
設定直後から「リアルタイムレポート」で自分のアクセスは確認できますが、全体の統計データが表示されるまでには24〜48時間ほどかかります。

### Search Console
こちらはデータの収集が始まるまで**数日（2〜3日）**かかります。最初は「データを処理しています。数日後にもう一度ご確認ください」と表示されますが、所有権の確認さえ済んでいれば、あとは放置して通知（メール）が来るのを待つだけでOKです。

## まとめ
GA4とSearch Consoleは、導入してすぐに効果が出るものではありません。しかし、記事数が増えた際にどの記事が検索上位に入っているか、どの記事が読まれているかを比較・分析するための貴重なデータ源になります。