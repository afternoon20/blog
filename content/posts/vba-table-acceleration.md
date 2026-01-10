+++
date = '2026-01-06T11:34:22+09:00'
draft = false
title = '【VBA】VBAでテーブル操作を高速化する | 180秒から1秒未満への改善'
description = 'Excel VBAでテーブル（ListObject）を扱う際、書き込み処理が遅いと感じる原因の多くは、セルへのアクセス方法とExcelの環境設定にあります。実際に800行のデータ処理で180秒かかっていた処理を、1秒未満まで短縮する手法を解説します。'
tags = ['VBA']
+++

Excel VBAでテーブル（ListObject）を扱う際、書き込み処理が遅いと感じる原因の多くは、セルへのアクセス方法とExcelの環境設定にあります。実際に800行のデータ処理で180秒かかっていた処理を、1秒未満まで短縮する手法を解説します。

## . 処理時間の推移
- 改善前（180秒）: 1行ずつ .ListRows.Add し、セルごとに値を代入。
- 中間（40秒）: 画面更新を停止し、配列で一括代入。
- 最終（1秒未満）: さらに自動計算を停止し、テーブルのリサイズ処理を最適化。

## 改善前の状況と問題点

元のコード（処理時間：約180秒）
1行追加するごとにuserインスタンスの中身を25列分セルへ値を書き込む方式です。

```vb
With allTable
    .Range(.Range.Rows.Count, 1) = user.idFlag
    .Range(.Range.Rows.Count, 2) = user.name
    ' ...（中略：25列分繰り返す）...
    .ListRows.Add
End With
```

以下、問題点になります。

### セルアクセスの回数
800行 × 25列 ＝ 20,000回のセル書き込みが発生しています。Excelとの通信回数が多すぎるため、動作が非常に重くなります。

### 行追加のオーバーヘッド
```.ListRows.Add```を実行するたびに、テーブル範囲の再定義が発生しています。

## 第1段階の改善：配列の導入（処理時間：約40秒）

画面更新を停止し、一度メモリ上の配列に格納してから書き込む手法です。
Application.ScreenUpdating = False の導入と配列一括処理をすることで180秒から40秒まで短縮することができました。依然として40秒かかるのは、テーブル範囲を拡張する際の自動再計算やテーブルのResize命令が負荷となっているためです。

## 最終的な最適化（処理時間：1秒未満）
さらに「自動計算の停止」と「書き込み範囲の最適化」を加えます。

```vb
Sub FinalOptimizedWrite()
    Dim userDict As Object: Set userDict = GetUserDictionary
    Dim rowCount As Long: rowCount = userDict.Count
    If rowCount = 0 Then Exit Sub

    ' 1. 環境制御（自動計算も停止）
    Dim prevCalc As Long: prevCalc = Application.Calculation
    Application.Calculation = xlCalculationManual
    Application.ScreenUpdating = False

    ' 2. 配列へデータ格納
    Dim tableArray() As Variant
    ReDim tableArray(1 To rowCount, 1 To 25)

    Dim x As Long: x = 1
    Dim u As User, k As Variant
    For Each k In userDict.Keys
        Set u = userDict(k)
        tableArray(x, 1) = u.IdFlag
        tableArray(x, 2) = u.Name
        ' ... (中略：必要なプロパティを格納) ...
        tableArray(x, 25) = u.Adjustment
        x = x + 1
    Next

    ' 3. 書き込み（Resizeではなく直接範囲指定）
    Dim targetTable As ListObject
    Set targetTable = ThisWorkbook.Worksheets("Sheet1").ListObjects("MyTable")

    With targetTable
        If Not .DataBodyRange Is Nothing Then .DataBodyRange.Delete
        
        ' 開始セルから配列サイズ分を一度に指定して代入
        .HeaderRowRange.Offset(1, 0).Resize(rowCount, 25).Value = tableArray
    End With

    ' 4. 環境の復元
    Application.Calculation = prevCalc
    Application.ScreenUpdating = True
End Sub
```

## 高速化のポイントまとめ
- 配列の一括代入: セルアクセスの回数を20,000回から1回に削減します。
- 自動計算の停止: 書き込み時の再計算を止め、処理完了後にまとめて計算させます。
- Resize命令の最適化: テーブル自体の構造変更命令（Resize）を減らし、値の代入による自動拡張を利用します。

800行程度の処理であれば、これらの対策をすべて適用することで、人間の目には一瞬で完了するように見えます。