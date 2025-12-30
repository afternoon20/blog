+++
date = '2025-12-30T21:11:57+09:00'
draft = false
title = 'HugoでCodePenを埋め込む'
description = 'Hugoのショートコードを使って、Codepenのコードをiframeで埋め込み表示する方法を紹介します。'
tags = ['Hugo']
+++

CodePenの埋め込みコードを、iframeを利用して表示するショートコードで作成します。

## ショートコードの作成
layouts/shortcodes/codepen.htmlを作成して以下を記述します。ユーザー名は任意の名前に書き換えます。

```html
<div style="margin: 1rem 0;">
    <iframe height="300" style="width: 100%;" scrolling="no" title="CodePen Embed" 
    src="https://codepen.io/ユーザー名/embed/{{ .Get "id" }}?default-tab=result" 
    frameborder="no" loading="lazy" allowtransparency="true" allowfullscreen="true">
    </iframe>
</div>
```

### 表示速度の最適化
loading="lazy"を指定することで、スクロールして表示領域に入るまで読み込みを遅延させ、ページ全体の表示速度を向上させています。

### 外部スクリプトの排除
公式の埋め込みタグに含まれる ei.js を読み込みません。外部のJavascriptの読み込みを省略できます。

## 記事（Markdown）での呼び出し方

記事内では、以下のようにコードのIDを渡すだけで埋め込みが完了します。

```
{{</* codepen id="raLVjXw" */>}}
```

実際のコードは以下になります。

{{< codepen id="ExPpWvE" >}}

## 補足： カスタマイズ例
特定の記事だけ高さを変えたい、あるいは他者のコードを紹介したい場合は引数を追加して対応できます。

```html
<div style="margin: 1rem 0;">
    <iframe height="{{ .Get "height" | default "300" }}" 
            style="width: 100%;" 
            scrolling="no" 
            title="CodePen Embed" 
            src="https://codepen.io/{{ .Get "user" | default "ユーザー名" }}/embed/{{ .Get "id" }}?default-tab=result" 
            frameborder="no" 
            loading="lazy" 
            allowtransparency="true" 
            allowfullscreen="true">
    </iframe>
</div>
```

記事（Markdown）での呼び出し方は以下のようになります。

```md
{{</* codepen id="raLVjXw" user="他のユーザー名" height="500" */>}}
```