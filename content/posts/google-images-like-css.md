+++
date = '2025-12-31T06:32:18+09:00'
draft = false
title = '【CSS】Google画像検索風レイアウトで画像を並べる'
description = "Google画像検索のようなレイアウトで画像を並べるCSSを紹介します。いわゆるタイル状に並べるスタイルです。"
tags = ['CSS']
+++

Google画像検索のようなレイアウトで画像を並べるCSSをいくつか紹介します。

※今回のCSSでは実装の容易さを優先したため、画像はトリミングが発生します。

## 基本のhtml
以下のhtmlを元にそれぞれCSSを適応させます。実際のレイアウトはCodepenで確認とします。
画像については[私のリポジトリー](https://github.com/afternoon20/test-images)から適当に指定しています。

```html
<div class="gallery">
    <div class="item"><img src="https://afternoon20.github.io/test-images/img01.jpg" alt="img01"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img02.jpg" alt="img02"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img03.jpg" alt="img03"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img04.jpg" alt="img04"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img05.jpg" alt="img05"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img06.jpg" alt="img06"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img07.jpg" alt="img07"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img08.jpg" alt="img08"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img09.jpg" alt="img09"></div>
    <div class="item"><img src="https://afternoon20.github.io/test-images/img10.jpg" alt="img10"></div>
</div>
```

## flexboxを使った実装
画像の比率を維持したまま、行ごとに高さを揃えるレイアウトです。Google画像検索に最も近い挙動をします。
行の右端まで隙間なく埋まり、各行で画像の枚数が可変になります。
※```object-fit: cover```を使用するため、指定した高さに合わせる際に画像の一部がトリミング（見切れ）されます。

```css
.gallery {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.item {
  height: 200px;
  flex-grow: 1;
}

.item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

{{< codepen id="yyJNgQO" >}}

## grid: masonryを使う
高さがバラバラな画像を、隙間を埋めるようにレンガ状に配置します。ただし、2025年現在、主要ブラウザの標準設定では動作しないので、次に紹介するJavaScriptライブラリを使った方法で実現可能です。

```css
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  grid-template-rows: masonry;
  gap: 10px;
}
```

**※2025年現在、主要ブラウザの標準設定では動作しません**

ブラウザの対応状況は以下の通りです。

| ブラウザ | 対応状況 |
| :--- | :--- |
| Chrome | 未対応（フラグでも不可） |
| Firefox | `layout.css.grid-template-masonry-value.enabled`を有効にすることで試験的に使用可能 |
| Safari | Technology Preview版で試験的に対応 |

将来的には標準対応が期待されていますが、現時点では本番環境での使用は推奨できません。
試してみたい場合はFirefoxで`about:config`を開き、上記フラグを`true`に変更することで動作確認ができます。

{{< codepen id="emzNgwv" >}}

## Masonry.jsを使ったレンガ状レイアウト
JavaScriptライブラリを使用して、画像の高さを維持したまま隙間なく並べます。画像を一切トリミングせず、パズルを埋めるように配置できます。

```css
.gallery {
  width: 100%;
}

.item {
  width: 19%;
  margin-bottom: 10px;
}

.item img {
  width: 100%;
  height: auto;
  display: block;
}
```

```js
window.onload = () => {
  new Masonry('.gallery', {
    itemSelector: '.item',
    columnWidth: '.item',
    gutter: 10,
    percentPosition: true
  });
};
```

{{< codepen id="raLVjXw" >}}

## レスポンシブ対応

スマホでの表示を考慮する場合、各手法に以下のメディアクエリを追加します。

### flexboxの場合

```css
@media (max-width: 600px) {
  .item {
    height: 150px;
  }
}
```

### Masonry.jsの場合

```css
@media (max-width: 600px) {
  .item {
    width: 48%;
  }
}
```

Masonry.jsはウィンドウのリサイズ時に自動で再計算されるため、
CSSの列幅を変更するだけでレスポンシブに対応できます。

## Googleのレイアウトを忠実に再現する場合
もし画像をトリミングせずに忠実に再現をするなら、以下のJavaScriptによる計算が必要になります。
- 画像の比率（アスペクト比）の取得: 全ての画像の元サイズを把握する
- 1行に入る画像の比率を合計し、画面幅に合わせてその行の最適な高さを数ピクセル単位で算出・上書き
- ウィンドウ幅が変わるたびに、再計算して画像の並び順や高さを入れ替える

CSSのFlexboxを使えば、数行のコードでGoogle画像検索風の隙間のないレイアウトが実現できます。

## この記事のまとめ

| 手法 | トリミング | ブラウザ対応 | 難易度 |
| :--- | :--- | :--- | :--- |
| **flexbox** | あり | 全ブラウザ対応 | 低 |
| **grid: masonry** | なし | 未対応（試験的） | 低 |
| **Masonry.js** | なし | 全ブラウザ対応 | 中 |

- 手軽にタイル状に並べたい → **flexbox**
- 画像の形を変えたくない → **Masonry.js**
- 将来的な標準仕様を試したい → **grid: masonry**（Firefox限定）

用途に合わせて使い分けてみてください。