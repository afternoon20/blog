+++
date = '2026-01-04T11:32:22+09:00'
draft = false
title = 'Alpine.jsで作るタグ入力（Tag Input）UIの実装'
description = 'ユーザーが入力したキーワードを視覚的な「チップ」として表示するタグ入力UIです。。本記事では、Alpine.jsを使用して、EnterやSpaceキーで動的にタグを生成する実装方法を解説します。'
tags = ['JavaScript']
+++

タグ入力UIは、検索フィルターやブログ記事のカテゴリ付け、スキル入力フォームなど、様々な場面で使われるUIパターンです。この記事では、Alpine.jsを使用してEnterやSpaceキーで動的にタグを生成する実装方法を解説します。

## Alpine.jsとは

Alpine.jsは、VueやReactのような本格的なフレームワークを導入するほどではない、小規模なインタラクションに適した軽量JavaScriptライブラリです。
HTMLに`x-data`などの属性を直接記述するだけで動作するため、既存のHTMLに手軽に組み込めます。

| ライブラリ | 特徴 | 向いているケース |
| :--- | :--- | :--- |
| Alpine.js | 軽量・HTML属性で記述 | 小規模なUI部品 |
| Vue.js | コンポーネント設計 | 中規模SPA |
| React | 大規模アプリ向け | 大規模SPA |

CDNから1行読み込むだけで使えるため、WordPressやHugoなどの静的サイトとの相性が特に良いです。

## 実装コード
### html

```html
<script src="https://unpkg.com/alpinejs@3.x.x/dist/cdn.min.js" defer></script>

<div x-data="tagComponent()" class="container">
    <div class="tag-input-wrapper">
        <template x-for="(tag, index) in tags" :key="index">
            <span class="chip">
                <span x-text="tag"></span>
                <button type="button" @click="removeTag(index)">&times;</button>
            </span>
        </template>

        <input 
            type="text" 
            x-model="newTag" 
            @keydown.enter.prevent="addTag"
            @keydown.space.prevent="addTag"
            @keydown.backspace="handleBackspace"
            placeholder="タグを入力..." 
        >
    </div>
</div>
```

### css

```css
.container {
    padding: 20px;
    max-width: 400px;
    margin: 0 auto;
}

.tag-input-wrapper {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    padding: 8px;
    border: 1px solid #ccc;
    border-radius: 6px;
    background: #fff;
}

.tag-input-wrapper:focus-within {
    border-color: #007bff;
    box-shadow: 0 0 0 2px rgba(0,123,255,0.25);
}

.chip {
    display: flex;
    align-items: center;
    background: #e1ecf4;
    color: #39739d;
    padding: 4px 8px;
    border-radius: 4px;
    font-size: 14px;
}

.chip button {
    background: none;
    border: none;
    margin-left: 6px;
    cursor: pointer;
    font-weight: bold;
}

input {
    border: none;
    outline: none;
    flex: 1;
    min-width: 100px;
}
```

### JavaScript

```js
function tagComponent() {
    return {
        tags: ['Alpine.js', 'UI'],
        newTag: '',
        addTag() {
            const val = this.newTag.trim();
            if (val && !this.tags.includes(val)) {
                this.tags.push(val);
            }
            this.newTag = '';
        },
        removeTag(index) {
            this.tags.splice(index, 1);
        },
        handleBackspace() {
            if (this.newTag === '' && this.tags.length > 0) {
                this.tags.pop();
            }
        }
    }
}
```

## 仕様のポイント

### 入力トリガー
EnterおよびSpaceキーでタグを確定させます。`.prevent`により、フォーム送信や入力欄への不要な空白挿入を防いでいます。

### 削除操作
チップ右側の「×」ボタンによる削除に加え、入力欄が空の状態でBackspaceを押すと末尾のタグが削除される仕様です。直感的な操作感でUXを向上させています。

### バリデーション
以下の2点をチェックしています。
- 空文字の追加を禁止
- 既存タグとの重複を禁止（`tags.includes(val)`で判定）

### 日本語入力（IME）との相性について

日本語をタグとして入力する場合、IMEによる変換確定時にもEnterキーイベントが発火するため、
意図せずタグが追加されてしまうことがあります。

これを防ぐには`compositionstart`と`compositionend`イベントを使用して、
IME変換中かどうかを判定する処理を追加します。

```js
function tagComponent() {
    return {
        tags: ['Alpine.js', 'UI'],
        newTag: '',
        isComposing: false, 
        addTag() {
            if (this.isComposing) return;
            const val = this.newTag.trim();
            if (val && !this.tags.includes(val)) {
                this.tags.push(val);
            }
            this.newTag = '';
        },
        removeTag(index) {
            this.tags.splice(index, 1);
        },
        handleBackspace() {
            if (this.newTag === '' && this.tags.length > 0) {
                this.tags.pop();
            }
        }
    }
}
```

inputタグにも以下のイベントを追加します。

```html
<input 
    type="text" 
    x-model="newTag" 
    @keydown.enter.prevent="addTag"
    @keydown.space.prevent="addTag"
    @keydown.backspace="handleBackspace"
    @compositionstart="isComposing = true"
    @compositionend="isComposing = false"
    placeholder="タグを入力..." 
>
```

### フォーカス時のスタイル変化
`:focus-within`を使用することで、入力欄にフォーカスが当たったときにラッパー全体のボーダーが青くなります。ネイティブの`<input>`のフォーカススタイルに近い自然なUXを実現しています。

## 実際のコード

{{< codepen id="ByzombM" >}}