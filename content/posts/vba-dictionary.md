+++
date = '2025-12-28T12:02:20+09:00'
draft = false
title = '【VBA】配列の代わりにDictionaryオブジェクトを使う'
description = 'VBAで使える便利なDictionaryオブジェクトについて解説します。'
tags = ['VBA']
+++

VBAの配列型は非常に扱いづらいので、代わりにDictionaryオブジェクトを使っています。その際によく使う実装方法をまとめます。

## Dictionaryオブジェクトについて
VBAで使える連想配列のオブジェクトです。キーと値をセットとして要素の追加、削除、存在チェックなど基本的なメソッドが使えます。

## 変数宣言と設定方法
Dictionaryには2種類の設定方法があります。

### 1. 実行時バインディング（推奨）
事前の設定が不要で、ファイルを他人に配布する場合にトラブルが少ない方法です。

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

## 配列との比較
| 機能 | 配列 | Dictionary |
| :--- | :--- | :--- |
| **検索** | ループが必要 | `Exists`で一瞬 |
| **サイズ変更** | `ReDim`が必要 | 自動で拡張 |
| **要素の削除** | 困難 | `Remove`で容易 |
| **Mac環境** | 利用可能 | 利用不可 |

## 参考サイト
- [Dictionary オブジェクト](https://learn.microsoft.com/ja-jp/office/vba/language/reference/user-interface-help/dictionary-object)