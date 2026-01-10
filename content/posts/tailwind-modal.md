+++
date = '2026-01-01T00:00:14+09:00'
draft = false
title = 'TailwindとJavaScriptで作るアニメーション付きモーダル'
description = '外部ライブラリを使わず、Tailwind CSSとバニラJSだけで軽量なモーダルを実装。複数設置への対応やアニメーション、アクセシビリティ（Escapeキー対応）のポイントを解説します。'
tags = ['TailWind CSS', 'JavaScript']
+++

Webサイト制作においてモーダルは必須パーツですが、ライブラリを導入するとプロジェクトが重くなったり、カスタマイズが制限されたりすることがあります。
今回は、Tailwind CSSのユーティリティクラスと最小限のJavaScript（バニラJS）のみで、複数設置にも対応した軽量なアニメーション付きモーダルの実装方法を紹介します。

## この実装のメリット
- ライブラリ不要でJSの記述も最小限で軽量
- Tailwind CSSのユーティリティクラスのみで実現
- アニメーション付きモーダル
- JavaScriptで開閉処理をイベントリスナーで分離
- ボタン、クリック、キーボード操作でモーダルの開閉が可能で実用的なUX

## 実際のコード（複数モーダル対応）

### HTMLの実装

data-modal-openとdata-modal-idを紐付けることで、1つのスクリプトで複数のモーダルを個別に制御できます。

```html
<div class="p-10 space-x-4">
  <button data-modal-open="modal-1" class="px-4 py-2 bg-blue-600 text-white rounded shadow hover:bg-blue-700 transition">
    モーダル1を開く
  </button>
  <button data-modal-open="modal-2" class="px-4 py-2 bg-green-600 text-white rounded shadow hover:bg-green-700 transition">
    モーダル2を開く
  </button>
</div>

<div id="modal-1" data-modal-id="modal-1" class="fixed inset-0 z-50 flex items-center justify-center invisible transition-all duration-300 pointer-events-none">
  <div class="absolute inset-0 bg-black/50 opacity-0 transition-opacity duration-300" data-modal-close></div>
  <div class="relative w-full max-w-md p-6 bg-white rounded-lg shadow-xl transform -translate-y-10 opacity-0 transition-all duration-300">
    <h2 class="text-xl font-bold mb-4">モーダル 1</h2>
    <p class="text-gray-600 mb-6">これは1つ目のモーダルです。共通のロジックで動作しています。</p>
    <button data-modal-close class="px-4 py-2 bg-gray-200 rounded hover:bg-gray-300 transition">閉じる</button>
  </div>
</div>

<div id="modal-2" data-modal-id="modal-2" class="fixed inset-0 z-50 flex items-center justify-center invisible transition-all duration-300 pointer-events-none">
  <div class="absolute inset-0 bg-black/50 opacity-0 transition-opacity duration-300" data-modal-close></div>
  <div class="relative w-full max-w-md p-6 bg-white rounded-lg shadow-xl transform -translate-y-10 opacity-0 transition-all duration-300">
    <h2 class="text-xl font-bold mb-4">モーダル 2</h2>
    <p class="text-gray-600 mb-6">別のコンテンツを持つ2つ目のモーダルです。</p>
    <button data-modal-close class="px-4 py-2 bg-gray-200 rounded hover:bg-gray-300 transition">閉じる</button>
  </div>
</div>
```

※このコードはTailwind CSS v3 (JITモード) を前提としています。```bg-black/50```などのクラスが動かない場合は、最新のCDNを読み込んでください。

### JavaScriptの実装

requestAnimationFrameを使用して、DOMの表示切り替え直後にアニメーションクラスを適用させるのがポイントです。

```js
document.addEventListener('DOMContentLoaded', () => {
  const openButtons = document.querySelectorAll('[data-modal-open]');
  const closeElements = document.querySelectorAll('[data-modal-close]');

  const openModal = (modalId) => {
    const modal = document.getElementById(modalId);
    if (!modal) return;

    const content = modal.querySelector('.transform');
    const overlay = modal.querySelector('.bg-black\\/50');

    modal.classList.remove('invisible');
    modal.classList.add('pointer-events-auto');

    requestAnimationFrame(() => {
      content.classList.remove('-translate-y-10', 'opacity-0');
      content.classList.add('translate-y-0', 'opacity-100');
      overlay.classList.remove('opacity-0');
      overlay.classList.add('opacity-100');
    });
  };

  const closeModal = (modal) => {
    const content = modal.querySelector('.transform');
    const overlay = modal.querySelector('.bg-black\\/50');

    content.classList.remove('translate-y-0', 'opacity-100');
    content.classList.add('-translate-y-10', 'opacity-0');
    overlay.classList.remove('opacity-100');
    overlay.classList.add('opacity-0');

    setTimeout(() => {
      modal.classList.add('invisible');
      modal.classList.remove('pointer-events-auto');
    }, 300);
  };

  openButtons.forEach(btn => {
    btn.addEventListener('click', () => {
      const targetId = btn.getAttribute('data-modal-open');
      openModal(targetId);
    });
  });

  closeElements.forEach(el => {
    el.addEventListener('click', (e) => {
      const modal = e.target.closest('[data-modal-id]');
      if (modal) closeModal(modal);
    });
  });

  document.addEventListener('keydown', (e) => {
    if (e.key === 'Escape') {
      const visibleModals = document.querySelectorAll('[data-modal-id]:not(.invisible)');
      visibleModals.forEach(modal => closeModal(modal));
    }
  });
});
```

## 実装の概要とポイント

### モーダルを開く
```data-modal-open```を付与したボタンがクリックされると、以下のステップでアニメーションを実行します。

1. ```const openModal```を実行```modalId```から対象DOMを特定
2. 対象DOMから```invisible```を削除、```pointer-events-auto```付与
3. requestAnimationFrame()を実行
4. ```translate-y-0, opacity-100```のクラス追加、300msのフェード＆スライドインアニメーション追加

### モーダルを閉じる
```data-modal-close```要素、モーダル外のクリックまたはEscapeキー入力をトリガーに実行します。

1. ```const closeModal```の実行、表示用クラスを削除
2. ```opacity-0 -translate-y-10``` のクラス付与、上方向へのフェードアウト開始
3. 300ms（アニメーション時間）待機後、```invisible```クラスを追加
4. ```pointer-events-none```で背面干渉を防止し完全に閉じる

## 実装のポイント
### アニメーションの仕組み
Tailwindの```transition-all```と```duration-300```を使用しています。invisible（displayに相当する制御）を外した直後にクラスを切り替えるため、```requestAnimationFrame```を使用して確実にブラウザに描画更新を認識させています。

### イベントの分離
```data-modal-close```を、閉じるボタンだけでなく背景の黒い領域にも付与しています。これにより、共通のクリックイベントで「外側クリック」も処理できます。

### ポインターイベント
```invisible```時は```pointer-events-none```にすることで、背後のボタンなどがクリックできない問題を回避し、表示時のみ```pointer-events-auto```に切り替えています。

## 実際の動作

ボタンを押すことでそれぞれのモーダルが表示されます。

{{< codepen id="QwEbeWb" >}}

以上です、仕組みを理解すれば使いやすいと思うのでぜひ活用してください。