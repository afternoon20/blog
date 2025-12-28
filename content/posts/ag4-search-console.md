+++
date = '2025-12-29T01:02:30+09:00'
draft = false
title = 'HugoでGA4とSearch Consoleを導入'
description = 'Hugoでサイトを公開した後に、サイト解析の設定を行うためのツール導入方法を解説します。GA4とSearch Consoleについて解説しています。'
tags = ['GA4', 'GSC', 'Netlify', 'Hugo']
+++

Hugoでブログを公開したら、最初に行うべきGA4とGoogle Search Consoleの導入手順をまとめます。今回はNetlifyにデプロイしていることを前提で説明します。

## Google Analytics (GA4) の設定

まずは、サイトに誰が来ているかを計測するためのGA4を設定します。

### 測定IDの取得
1. [Googleアナリティクス](https://analytics.google.com/analytics/web/)でプロパティを作成
2. 「データストリーム」からウェブを選択し、サイトURLを登録
3. 発行された 「測定ID（G-XXXXXXXXXX）」 をコピー

### Hugoへ組み込み
GA4のテンプレートが内蔵されているテーマの場合は、設定ファイルとヘッドタグの2箇所を編集するだけで完了します。

hugo.toml（または config.toml）のルート階層にIDを追記します。

```toml
googleAnalytics = 'G-XXXXXXXXXX'
```

layouts/partials/head.html内に、以下の1行を追加します。

```html
{{- template "_internal/google_analytics.html" . -}}
```

テーマによっては、すでに「google_analytics.html」を含むコードがあるのでそれは削除してください。

## Google Search Console の設定
次に、Google検索結果でのパフォーマンスを管理するSearch Consoleを設定します。読者がどんなキーワードで自分のサイトに辿り着いたかがわかるため、検索意図に合わせて記事をブラッシュアップするのに役立ちます。

### ドメインプロパティの登録

1. [Search Console](https://search.google.com/search-console/about?authuser=1)で「ドメイン」を選択し、設定したいURLのドメインを入力
2. 所有権確認用の TXTレコード（google-site-verification=...）をコピー

### DNSの設定（Netlifyの場合）
Netlifyでドメインを管理している場合は、DNSパネルからレコードを追加します。

1. Netlifyの Domain management > DNS panel を開く
2. Add new record をクリックし、以下を入力
- Type: TXT
- Name: @
- Value: 先ほどSearch ConsoleでコピーしたTXTレコード
3. 保存後、Search Consoleで「確認」ボタンを押して完了

## 反映を待つ
設定がすべて完了しても、データが反映されるまでには少し時間がかかります。

### Google Analytics
設定直後から「リアルタイムレポート」で自分のアクセスは確認できますが、全体の統計データが表示されるまでには24〜48時間ほどかかります。

### Search Console
こちらはデータの収集が始まるまで**数日（2〜3日）**かかります。最初は「データを処理しています。数日後にもう一度ご確認ください」と表示されますが、所有権の確認さえ済んでいれば、あとは放置して通知（メール）が来るのを待つだけでOKです。