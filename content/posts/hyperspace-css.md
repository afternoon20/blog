+++
date = '2026-10-01T06:32:18+09:00'
draft = false
title = '【CSS】ハイパースペース風アニメーションの実装方法と解説'
description = "Google画像検索のようなレイアウトで画像を並べるCSSを紹介します。いわゆるタイル状に並べるスタイルです。"
tags = ['CSS', 'JavaScript']
+++

光が中心から放射状に動いている、ハイパースペース風のアニメーションをCSSとJavaScriptで実装してみました。

## 仕様と完成形

- 光の線⋯600本、長さ⋯25〜65vmax、中心に半径 8vmax の空洞
- 円の中心から飛び出るようなアニメーション
- マウスオーバーまたは画面を押し続けていると、円の中心が追従する

完成形は以下になります。

{{< codepen id="01a0db4d-e80b-7407-9c3a-af0225df5f66" >}}

## HTMLの全体構成
```html
<div class="warp">          <!-- 画面全体。イベントを受け取る -->
  <div class="field"></div>  <!-- 線の入れ物。transformで動かす -->
</div>
<div class="caption">Hyperspace</div>  <!-- 上に乗せるテキスト -->
```

要素は3つで、線をまとめた```.field```を丸ごと動かすことで消失点（中心）を移動させています。

## CSSの解説
### 中央の円の半径の定義
```css
:root {
  --hole: 8vmax;
}
```
変数holeを定義して、円の半径を指定します。vmaxは画面サイズを基準にした単位で、幅と高さの大きい方の1%となります。よって、今回の実装は、画面の長い辺の8%を円の半径として定義しています。

今回は```:root```に定義していますが、要素の中に実装する場合はその親要素に指定します。

### 背景とはみ出しの設定
```css
html, body {
  margin: 0;
  height: 100%;
  background: #000;
  overflow: hidden;
  -webkit-user-select: none;
  user-select: none;
  -webkit-touch-callout: none;
}
```
```overflow: hidden```は、画面の外へ飛んでいった線によってスクロールバーが出ないようにするための指定です。
下の3行は、スマホで長押ししたときにテキスト選択やメニューが出ないようにしています。

### 線をまとめる要素
```css
.field {
  position: absolute;
  inset: 0;
  will-change: transform;
}
```
中心が動いた際にすべての線が追従するようにしています。```will-change: transform```は、この要素が頻繁に動くことをブラウザに事前に伝えて描画の準備をさせ、動きを滑らかにするための指定です。

[will-change - CSS | MDN](https://developer.mozilla.org/ja/docs/Web/CSS/Reference/Properties/will-change)

### 中心の淡い光
```.field::after```で、疑似要素を作成しています。高さと幅を事前に定義していた変数holeを2倍にして直径にしています。```translate(-50%, -50%)```で自分の大きさの半分だけ戻して、円の中心をぴったり合わせています。

### 光の線1本の定義
光の線1本分のスタイルです。JavaScriptでこの要素を600個作ります。leftとtopで線の出発点を画面の中央にそろえています。

### 線の高さと幅の設定
```width```が線の長さ、```height```が線の太さです。値を```var(--len)```と```var(--t)```にしているのは、JavaScriptで1本ずつ違う値を入れるためです。すべて同じ長さ・太さだと、機械的な模様に見えてしまいます。

### 線にグラデーションをつけるスタイリング
```90deg```で左から右へ色が変わり、左端（中心側）は透明、右端（先端）は白になります。途中の45%と80%に薄い白を置いて、光の尾が長く残るようにしています。```calc(var(--o) * 0.15)```のように明るさの変数を掛けて、線ごとに明るさを変えています。```opacity: 0;```は、初期状態を透明にするために定義しています。

### animationで線のアニメーションを設定
```@keyframes warp```の動きを割り当てます。1回の時間は```--dur```、加速していく速さの変化は```cubic-bezier```、開始位置のずれは```--delay```（マイナス値）、```infinite```で無限に繰り返します。
```cubic-bezier```は曲線グラフのような時間経過を指定できます（```ease-in```や```ease-out```よりも細かい設定が可能）。
細かい設定値については公式ドキュメントを参照してください。

[animation プロパティ (CSS) - CSS | MDN](https://developer.mozilla.org/ja/docs/Web/CSS/Reference/Properties/animation)

```@keyframes warp```で時間経過によるスタイルの指定を行っています。```translateX(150vmax)```は、画面の外まで線を移動させるようにしています。

これらを設定した線1本の動きは以下のようになります。

{{< codepen id="01a0dd40-7733-737b-b04f-8065abb82560" >}}

## JavaScriptの解説
### 線の本数とランダム関数の定義
```js
const COUNT = 600;
const rand = (min, max) => min + Math.random() * (max - min);
```

作成する線の本数と、範囲内の値をランダムで返す関数を事前に定義しておきます。

### メインのループ処理で、線をランダムな値で作成する
```js
for (let i = 0; i < COUNT; i++) {
  const s = document.createElement("span");
  s.className = "streak";
  const dur = rand(1.1, 3.2);
  s.style.setProperty("--a", rand(0, 360) + "deg");
  s.style.setProperty("--len", rand(25, 65) + "vmax");
  s.style.setProperty("--dur", dur + "s");
  s.style.setProperty("--delay", -rand(0, dur) + "s");
  s.style.setProperty("--o", rand(0.35, 1).toFixed(2));
  s.style.setProperty("--t", (Math.random() < 0.85 ? 1 : 2) + "px");
  field.appendChild(s);
}
```

定義したCOUNTの本数分の線を作成していきます。setPropertyで、CSSで定義していた変数にランダムな値を設定することで、角度・長さ・速さ・明るさ・太さを変えることができます。太さは85%の確率で1px、残りを2pxにして、ところどころ太い線を混ぜています。

```--delay```にはマイナスの値を入れています。```animation-delay```にマイナスを指定すると、その時間だけ経過した状態から再生が始まります。これがないと、ページを開いた瞬間に600本が一斉に中心からスタートしてしまうため、開いた時点ですでに線が飛び交っている状態を作っています。

### 中心の位置の定義
```js
const target = { x: 0, y: 0 };
const current = { x: 0, y: 0 };
const EASE = 0.08;
```

中心の位置を、画面の中央から何pxずれているかで管理しています。```target```はマウスや指の操作で決まる目標の位置、```current```は今表示している位置です。

操作があったときにすぐ中心を動かすと、線がカクッと飛んでしまいます。そのため目標だけを変えておき、実際の位置は```EASE```の割合ずつ目標に近づけて、滑らかに移動させています。

### 中心位置を変える関数の定義
中心位置を変える関数は、以下の挙動になるように実装しています。

- setCenter・・・マウスや指の座標を受け取り、画面中央からのずれに変換して目標の位置に設定します。
- resetCenter・・・目標の位置を画面の中央に戻します。
- tick・・・毎フレーム少しずつ動かし、毎回目標との差の8%（定数EASE）ずつ近づけて、その位置を```.field```の```transform```に反映しています。線を1本ずつ動かすのではなく、線の入れ物である```.field```を動かすことで、すべての線が新しい中心に追従します。

tickは```requestAnimationFrame```で、画面が描き直されるタイミングに合わせて繰り返し実行しています。

[Window: requestAnimationFrame() メソッド - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Window/requestAnimationFrame)

### マウスオーバー、タッチ時の制御
ここからは、各イベントの制御を行う処理を実装しています。マウスと指の操作は、Pointer Eventsを使って同じイベントで受け取り、```e.pointerType```でマウスか指かを判定しています。

[ポインターイベント - Web API | MDN](https://developer.mozilla.org/ja/docs/Web/API/Pointer_events)

```js
const MOVE_LIMIT = 10;
let touch = null;
```

```MOVE_LIMIT```は、画面を押している間10pxまでは長押ししているとみなして、中心をその地点にするための値です。指は押しているだけでも少し揺れるので、多少のブレは許しています。

```touch```は、押している指の情報（どの指か、押し始めの位置）を記録する変数です。押していないときは```null```になります。

```js
warp.addEventListener("pointerdown", (e) => {
  if (e.pointerType === "mouse") return;
  touch = { id: e.pointerId, x: e.clientX, y: e.clientY };
  setCenter(e.clientX, e.clientY);
});
```

画面を押した瞬間に、どの指かと押した位置を```touch```に記録し、その座標をsetCenterに渡して中心位置を設定しています。マウスのクリックでは何もしないため、最初に処理を抜けています。

```js
warp.addEventListener("pointermove", (e) => {
  //略
});
```

pointermoveの部分は、指（押し続けている場合）やマウスを動かした時のイベント制御です。マウスの場合はマウスオーバーしている座標を中心に設定します。指の場合は、```touch```に記録した押し始めの位置からの移動距離を計算し、10pxを超えて動いたら長押しをやめたとみなしてresetCenterを実行し、画面の中央を中心に戻します。10px以内の場合は、押した瞬間の位置を中心のまま維持します。

endTouchは、指を離したときや、スクロールなどで操作が中断されたときの制御で、中身は中心を画面の中央に戻す初期化処理です。

上記以降の処理は、マウスが外に出たら初期化したり、スマホで長押ししたときにメニューが出ないように制御をしています。

### この実装のまとめ
今回のアニメーションは、見た目は複雑ですが、やっていることは次の2つです。

- CSSで光の線1本の動きを作る
- JavaScriptでその線を600本複製し、CSS変数で1本ずつ角度や速さを変える

線の動きはすべてCSSのアニメーションに任せ、JavaScriptは線の生成と中心の移動だけを担当しています。また、中心を動かすときは線を1本ずつ動かさず、線の入れ物である```.field```だけを動かすことで、処理を軽くしています。

光の長さを変えたい場合は```--len```の範囲を、本数を変えたい場合は```COUNT```の値を変更するだけで調整できます。ただし、線1本ごとにHTML要素を作る方式のため、本数を増やしすぎるとスマホなどで動きが重くなるので注意してください。