+++
date = '2025-12-28T12:02:20+09:00'
draft = false
title = '【VBA】配列の代わりにDictionaryオブジェクトを使う'
description = 'VBAの配列操作に限界を感じている方へ。実務で役立つDictionaryオブジェクトの活用術と、配列との明確な使い分けを解説します。'
tags = ['VBA']
+++

VBAの動的配列で```ReDim Preserve```を繰り返して可読性が落ちたり、処理速度が落ちて困ったことがありました。
そこで私は大量のデータを名寄せする際、配列のループ処理に限界を感じて```Dictionaryオブジェクト```に乗り換えました。
結果、コードが劇的にスッキリし、保守性も上がりました。そこでよく使っている実装方法をまとめます。

## Dictionaryオブジェクトについて
VBAで使える連想配列のオブジェクトです。キーと値をセットとして要素の追加、削除、存在チェックなど基本的なメソッドが使えます。

## 変数宣言と設定方法
Dictionaryには2種類の設定方法があります。

### 1. 実行時バインディング（推奨）
実務では、自分一人が使うツールなら「参照設定」で良いのですが、社内の他部署に配布する場合は、参照設定のズレで動かなくなるトラブルが多発します。
そのため、私は配布用ツールでは必ず「実行時バインディング（CreateObject）」を使うようにしています。

```vb
Dim dictionary As Object: Set dictionary = CreateObject("Scripting.Dictionary")
```

### 2. 参照設定（Early Binding）
VBEのメニューから [ツール] ＞ [参照設定] ＞ [Microsoft Scripting Runtime] にチェックを入れます。入力候補が表示されるようになり、開発効率が上がります。

```vb
Dim dictionary As New Scripting.Dictionary
```

**注意点：Mac環境での利用について**
Scripting.Dictionary はWindows専用のライブラリ（scrrun.dll）を使用しているため、Mac版のExcelでは動作しません。 Mac環境を含むマルチプラットフォームで開発する場合は、VBA標準の Collection オブジェクトや、自作のクラスで代替する必要があります。

## 使い方例
### 要素の追加

```vb
Call dictionary.Add("1", "value")
dictionary("2") = "value2"
Debug.Print dictionary("1")
```

キーは必須で数値型でも指定できます。(Collection型は文字列のみ)

### 要素の存在確認
```vb
If dictionary.Exists("1") Then
    Debug.Print "存在します"
End If
```

### 配列のループ
```vb
Dim key As Variant
For Each key In dictionary
    Debug.Print key & ":" & dictionary(key)
Next
```

### 要素の削除
```vb
Call dictionary.Remove("1")
Call dictionary.RemoveAll()
```

### 擬似的にプロパティを連想配列に定義
```vb
Dim user As Object: Set user = CreateObject("Scripting.Dictionary")
user("ID") = 101
user("Name") = "田中"
user("Role") = "Admin"

Dim k As Variant
For Each k In user
    Debug.Print k & "は" & user(k)
Next
```

### リストから重複を除いたユニークなIDだけを抽出する
```vb
Dim cell As Range
For Each cell In Range("A2:A100")
    If Not dictionary.Exists(cell.Value) Then
        dictionary.Add cell.Value, True
    End If
Next
```

## 配列との比較
| 機能 | 配列 | Dictionary |
| :--- | :--- | :--- |
| **検索** | ループが必要 | `Exists`で一瞬 |
| **サイズ変更** | `ReDim`が必要 | 自動で拡張 |
| **要素の削除** | 困難 | `Remove`で容易 |
| **Mac環境** | 利用可能 | 利用不可 |

Dictionaryオブジェクトを推奨してきましたが、要素数が固定のときは単純な配列を使ったほうが使いやすいこともあります。

## 参考サイト
- [Dictionary オブジェクト](https://learn.microsoft.com/ja-jp/office/vba/language/reference/user-interface-help/dictionary-object)