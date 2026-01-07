+++
date = '2026-01-06T10:38:27+09:00'
draft = true
title = '【VBA】テーブル（ListObject）をループ処理する方法｜並び替えからデータ抽出まで'
description = 'ExcelのListObjectをVBAで操作する基本的な方法を解説します。本記事ではループ処理を実際のコードを例に説明します。'
tags = ['VBA']
+++


Excelのテーブル機能（ListObject）をVBAで操作すると、データの増減に柔軟に対応できる保守性の高いプログラムが構築できます。

本記事では、テーブル内のデータを特定の列で並び替えた後、1行ずつループ処理してデータを取り出す基本パターンを解説します。

## 実装の前提条件

今回の解説では、以下の環境を想定しています。
- シート名: DataSheet
- テーブル名: MainTable
- 処理内容: 「日付」列を基準に昇順ソートし、各行の3列目の値を抽出

## サンプルコード
VBAでテーブルを操作する標準的なコードです。

```vb
Option Explicit

Sub ProcessTableRows()
    Dim targetTable As ListObject
    Set targetTable = ActiveWorkbook.Worksheets("DataSheet").ListObjects("MainTable")

    targetTable.Range.AutoFilter Field:=3
    targetTable.Sort.SortFields.Clear
    targetTable.Sort.SortFields.Add2 Key:= _
        Range("MainTable[[#All],[日付]]"), SortOn:=xlSortOnValues, Order:=xlAscending, _
        DataOption:=xlSortNormal

    With targetTable.Sort
        .Header = xlYes
        .MatchCase = False
        .Orientation = xlTopToBottom
        .SortMethod = xlPinYin
        .Apply
    End With

    Dim row As Excel.ListRow
    For Each row In targetTable.ListRows
        If row.Range.Cells(3) = "" Then
            Exit For
        End If
        Debug.Print row.Range.Cells(3).Value
    Next
End Sub
```

## コードの解説と技術的なポイント
### ListObjectによるテーブルの定義

```Worksheets("シート名").ListObjects("テーブル名") ```を使用してテーブルをオブジェクト変数に格納します。通常の```Range```指定と異なり、データ行が追加・削除されても範囲を自動的に認識するため、最終行を取得する処理（End(xlUp) など）を記述する必要がありません。

### 構造化参照を用いたソート処理

ソートのキー指定に ```Range("MainTable[[#All],[日付]]") ```という構造化参照を用いています。これにより、シート上での列の位置（列番号）が変更されても、「日付」という見出し名さえ合致していれば、コードを修正することなく正確に並び替えを実行できます。

### ListRowsコレクションによる行の反復処理

テーブル内の各行を操作するには、ListRowsコレクションを```For Each```文でループさせるのが効率的です。

#### row.Range
ループ中の「現在の行」全体をRangeオブジェクトとして扱います。

#### Cells(3)
その行内の左から3番目のセルを参照します。

## ListObjectを使うメリット

通常のセル範囲（Range）ではなく、テーブルとして扱うことで以下の利点があります。

### 動的な範囲参照
データが増えてもListRowsが自動的に範囲を更新します。

### 可読性の向上
テーブル名や列名を利用することで、何に対する処理なのかが明確になります。

### 保守性
列の入れ替えや挿入といった、運用中によくあるレイアウト変更に対してエラーが起きにくくなります。

## まとめ
ListObjectを活用したループ処理を習得すると、Excelのシート構成の変化に強い、堅牢なツールを作成できるようになります。大量のデータを扱う実務においては、標準的なRange操作よりもテーブル操作を優先して利用することを推奨します。