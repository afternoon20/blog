+++
date = '2025-12-28T22:19:54+09:00'
draft = false
title = 'Cloudinaryで動的なOGP画像を生成する'
description = 'Cloudinaryを使用して記事タイトル入りのOGP画像を自動生成する方法を解説。日本語フォントの適用設定やURLエンコードの仕様、React・Hugoでの実装例をまとめました。'
tags = ['Cloudinary']
+++

ブログのSNSシェア時に表示されるOGP画像を、記事タイトルに合わせて動的に生成する仕組みを構築しました。
Cloudinaryを利用すると、サーバー側の処理を記述することなく、URLパラメータのみで画像加工が可能です。
日本語フォントを適用するための権限設定や、特殊なURLエンコードの仕様など、実装に必要な具体的な手順を解説します。

## Cloudinaryのアカウント作成と初期設定
まずは以下からサインアップ、ログインします。

[https://cloudinary.com/](https://cloudinary.com/)

## Cloudinaryでの設定手順

### Cloud name（クラウド名）の確認
Dashboardに表示されている「Cloud name」を使用します。これは生成URLのホスト部分に含まれる識別子です。

### 日本語フォントを有効化するための権限設定
デフォルト以外のフォント（Googleフォント等）を読み込むには、認証済みアップロード（Authenticated）の設定が必要です。

1. Settings > Upload を開く
2. Add upload preset をクリック
3. Upload preset nameを```custom_fonts```等にする（任意の名前にする）
4. 左メニューのOptimize and Delivery > Delivery typeをAuthenticatedに変更
5. それ以外は変更しないでsaveを押す
6. Settings > Uploadに戻り、Manage DefaultsからMedia LibraryのRawを作成したpresetに変更して保存

### 素材のアップロード
#### 背景画像
Assetからアップロードして画像を選択します。右にメニューが出るのでPublic IDを設定したい値に変更します。（例：ogp_base）
#### 日本語フォント
Googleフォントから[NotoSansJP](https://fonts.google.com/noto/specimen/Noto+Sans+JP)などをダウンロードします。その後、画像と同様にAssetから任意のttfファイルをアップロードします。PublicIDには「NotoSansJP-Bold.ttf」など拡張子込みの名前に変更します。

## 実装コード例

### URLエンコードの仕様
Cloudinaryで日本語テキストを重ねる場合、通常のURLエンコードに加え、記号```%```を```%25```に置換する処理が必要です。
これを行わない場合、マルチバイト文字が正しく認識されません。

### Reactでの実装例

```js
const getOgpUrl = (title) => {
  const cloudName = "your_cloud_name";
  const baseId = "ogp_base";
  
  const encodedTitle = encodeURIComponent(title).replace(/%/g, '%25');
  
  const params = [
    `l_text:NotoSansJP-Bold.ttf_60_bold:${encodedTitle}`,
    'w_850',         // テキストの最大横幅
    'co_rgb:000000', // 文字色
    'c_fit',         // 指定幅での自動改行
    'g_center'       // 配置の基準点
  ].join(',');

  return `https://res.cloudinary.com/${cloudName}/image/upload/${params}/v1/${baseId}.jpg`;
};
```

定数paramsはそれぞれ、使用するフォント・サイズ・太さ、テキストを表示する最大横幅、文字色、枠内に収めるための改行設定、配置の基準位置を設定しています。

### Hugoでの実装例

```html
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

### URLパラメータの構成

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

## まとめ
Cloudinaryを活用することで、運用コストを抑えつつ記事ごとに最適化されたOGP画像を配信できます。
Zennなどの技術系メディアでも採用されている手法であり、個人開発のブログにおいても有用な機能です。