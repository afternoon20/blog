+++
date = '2025-12-28T22:19:54+09:00'
draft = false
title = 'Cloudinaryで動的なOGP画像を生成する'
description = 'Cloudinaryで動的なOGP画像を生成する方法とReact及びHugoでの実装例を解説します。日本語フォントの外部読み込みも合わせて解説します。'
tags = ['Cloudinary']
+++

ZennのOGP画像のように画像のテンプレートを用意して、そこに動的にブログ記事のタイトルを挿入する方法を解説します。
今回はCloudinaryというサービスを使用し、Hugo、Reactでの実装例も紹介します。

## ログイン
まずは以下からサインアップ、ログインします。

[https://cloudinary.com/](https://cloudinary.com/)

## Cloudinaryでの設定手順

### Cloud name（クラウド名）の確認
Home > Dashboardのうち、Product Environmentの「Cloud name」がクラウド名になります。

### 日本語フォントを有効化するための権限設定
デフォルトのArialが微妙な場合は、Googleフォントなどをアップロードして読み込ませることができます。そのための必須設定です。
1. Settings > Upload を開く
2. Uploadから「Add upload preset」ボタンを押す
3. Upload preset nameは任意入力（例：custom_fonts）
4. 左メニューの「Optimize and Delivery」> Delivery typeを「Authenticated」
5. それ以外は変更しないでsaveを押す
6. Settings > Uploadに戻り、「Manage Defaults」からMedia LibraryのRawを作成したpresetに変更して保存

### 素材のアップロード
#### 背景画像
Assetからアップロードして画像を選択します。右にメニューが出るのでPublic IDを設定したい値に変更します。（例：ogp_base）
#### 日本語フォント
Googleフォントから[NotoSansJP](https://fonts.google.com/noto/specimen/Noto+Sans+JP)などをダウンロードします。その後、画像と同様にAssetから任意のttfファイルをアップロードします。PublicIDには「NotoSansJP-Bold.ttf」など拡張子込みの名前に変更します。

## 実装コード例
### Hugo

### React

```js
const getOgpUrl = (title) => {
  const cloudName = "your_cloud_name";
  const baseId = "ogp_base";
  const encodedTitle = encodeURIComponent(title).replace(/%/g, '%25');
  
  const params = [
    `l_text:NotoSansJP-Bold.ttf_60_bold:${encodedTitle}`,
    'w_850',
    'co_rgb:000000',
    'c_fit',
    'g_center'
  ].join(',');

  return `https://res.cloudinary.com/${cloudName}/image/upload/${params}/v1/${baseId}.jpg`;
};
```

定数paramsはそれぞれ、使用するフォント・サイズ・太さ、テキストを表示する最大横幅、文字色、枠内に収めるための改行設定、配置の基準位置を設定しています。

### Hugo

```html
{{- /* Cloudinary */}}
{{- $cloudName := "your_cloud_name" -}}
{{- $baseId := "ogp_base" -}}
{{- $text := .Title -}}
{{- if $text -}}
  {{- $encodeText := urlquery $text -}}
  {{- $doubleEncodedText := replace $encodeText "%" "%25" -}}
  {{- $ogUrl := printf "https://res.cloudinary.com/%s/image/upload/l_text:NotoSansJP-Bold.ttf_60_bold:%s,w_850,co_rgb:000000,c_fit,g_center/v1/%s.jpg" $cloudName $doubleEncodedText $baseId -}}
  <meta property="og:image" content="{{ $ogUrl }}">
{{- end -}}
```

### 生成されるURLの構造

以下のようなURLが生成されます。
```
https://res.cloudinary.com/[cloudName]/image/upload/[params]/[version]/[baseId].jpg
```

| 項目 | 意味・役割 |
| :--- | :--- |
| **cloudName** | アカウント固有のID、Dashboardで確認できる名前 |
| **image/upload** | リソースタイプとデリバリータイプ、画像を動的に生成・配信することを示す |
| **params** | 最重要項目。カンマ区切りで複数の加工指示を並べる |
| **version** | キャッシュ制御用の数値（例: v1）、画像を更新した際にブラウザへ変更を知らせる役割 |
| **baseId** | 土台となる背景画像のPublic ID |

この記事で実際に生成されたURLは以下になります。
```
https://res.cloudinary.com/dl79ugpcw/image/upload/l_text:NotoSansJP-Bold.ttf_60_bold:Cloudinary%25E3%2581%25A7%25E5%258B%2595%25E7%259A%2584%25E3%2581%25AAOGP%25E7%2594%25BB%25E5%2583%258F%25E3%2582%2592%25E7%2594%259F%25E6%2588%2590%25E3%2581%2599%25E3%2582%258B,w_850,co_rgb:000000,c_fit,g_center/v1/ogp_base.jpg
```

## 補足
実際ZennでもCloudinaryを使用しているようです。（検証ツールから確認）