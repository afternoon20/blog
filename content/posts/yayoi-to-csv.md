+++
date = '2026-01-01T10:10:40+09:00'
draft = false
title = '【Python】弥生会計のインポートクラスの作成と使用例'
description = 'Pythonで弥生会計の仕訳インポート用CSVを自動生成する方法。Excelからデータを読み込み、複雑な「識別フラグ」を自動判定して出力する実用的なクラスの実装例を解説します。'
tags = ['Python', '弥生会計']
+++

## はじめに
手入力での会計処理はミスが起きやすく、件数が多いと膨大な時間がかかります。
Pythonを活用して、Excelの取引データから弥生会計のインポート用CSVを自動生成する仕組みを構築しました。
特に設定が煩雑な識別フラグの自動計算や、弥生会計特有の文字コード制約に対応した汎用的なインポートクラスの実装例を紹介します。

※実際にインポートする際は、事前にバックアップを取ることをお勧めします。

## 今回実現する仕組み
- 弥生会計のインポート形式（25項目）を構造化した「Yayoiクラス」の定義
- openpyxlによるExcelデータの抽出と整形
- 複数行伝票に対応した識別フラグの自動付与
- 弥生会計が認識可能なShift-JIS（cp932）形式での出力

## 前提とするExcelデータ構造

以下のように、1行ごとに借方または貸方の情報が入力されているリスト形式を想定します。

| A:取引日 | B:借方科目 | C:借方金額 | D:貸方科目 | E:貸方金額 | F:摘要 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 2025/12/01 | 消耗品費 | 1500 | | | 文房具代 |
| 2025/12/01 | | | 現金 | 1500 | 文房具代 |
| 2025/12/01 | 旅費交通費 | 840 | | | 電車代 |
| 2025/12/01 | | | 現金 | 840 | 電車代 |
| 2025/12/03 | 接待交際費 | 5500 | | | 会食代 |
| 2025/12/03 | | | 未払金 | 5500 | 会食代 |
| 2025/12/04 | 通信費 | 11000 | | | ネット代 |
| 2025/12/04 | | | 普通預金 | 11000 | ネット代 |

## インポート形式について

形式について、[弥生会計の製品サポート](https://support.yayoi-kk.co.jp/subcontents.html?page_id=18545)から確認できます。
特に注意するのは識別フラグで、正しい識別フラグを記入しないとインポートエラーになります。こちらも自動化で正確に設定するようにします。

## 事前準備（ライブラリのインストール）
PythonでExcelファイルを操作するために、openpyxlというライブラリを使用します。インストールしていない場合は、以下のコマンドで準備します。
```sh
pip install openpyxl
```

## 弥生クラスの定義
弥生会計のインポート形式は25項目と決まっています。これをクラス化することで、項目の並び順やデフォルト値の管理を容易にします。

```py
class Yayoi:
    def __init__(self):
        self.id_flag = None
        self.slip_number = None
        self.settlement_slip = None
        self.slip_day = None
        self.debit_name = None
        self.debit_sub_name = None
        self.debit_department = None
        self.debit_tax_type = '対象外'
        self.debit_amount = 0
        self.debit_tax = 0
        self.credit_name = None
        self.credit_sub_name = None
        self.credit_department = None
        self.credit_tax_type = '対象外'
        self.credit_amount = 0
        self.credit_tax = 0
        self.summary = None
        self.bill_number = None
        self.due_date = None
        self.slip_type = 3
        self.system_source = None
        self.memo = None
        self.tag1 = None
        self.tag2 = None
        self.adjustment = 'no'

    def to_list(self):
        return [
            self.id_flag,
            self.slip_number,
            self.settlement_slip,
            self.slip_day,
            self.debit_name,
            self.debit_sub_name,
            self.debit_department,
            self.debit_tax_type,
            self.debit_amount,
            self.debit_tax,
            self.credit_name,
            self.credit_sub_name,
            self.credit_department,
            self.credit_tax_type,
            self.credit_amount,
            self.credit_tax,
            self.summary,
            self.bill_number,
            self.due_date,
            self.slip_type,
            self.system_source,
            self.memo,
            self.tag1,
            self.tag2,
            self.adjustment
        ]
```

## エクセルの読み込みと弥生クラスへの値設定
openpyxlを使用してExcelファイルからデータを取得し、Yayoiクラスのインスタンスに値を設定します。
取引日はA列から取得し、弥生会計が認識できる「YYYY/MM/DD」形式の文字列に変換して保持します。

```py
import openpyxl

def load_excel_to_yayoi(filepath):
    wb = openpyxl.load_workbook(filepath)
    ws = wb.active
    
    yayoi_instances = []
    
    for row in ws.iter_rows(min_row=2, values_only=True):
        # A:取引日, B:借方科目, C:借方金額, D:貸方科目, E:貸方金額, F:摘要
        s_day, d_name, d_amount, c_name, c_amount, summary = row[0:6]
        
        # A列（取引日）が空白になった時点で終了
        if s_day is None:
            break
            
        item = Yayoi()
        item.summary = summary
        
        # 取引日のフォーマット変換
        item.slip_day = s_day.strftime('%Y/%m/%d') if hasattr(s_day, 'strftime') else str(s_day)
        
        # 借方・貸方の各項目をそのまま設定
        item.debit_name = d_name
        item.debit_amount = d_amount if d_amount else 0
        item.credit_name = c_name
        item.credit_amount = c_amount if c_amount else 0
            
        yayoi_instances.append(item)

    # 読み込んだ後に日付で並び替え
    yayoi_instances.sort(key=lambda x: x.slip_day)

    # 日付ごとの連想配列に整理
    yayoi_dict = {}
    for item in yayoi_instances:
        key = item.slip_day
        if key not in yayoi_dict:
            yayoi_dict[key] = []
        yayoi_dict[key].append(item)
        
    return yayoi_dict
```

今回は1行ごとに借方行と貸方行が分かれていますが、金額が1つでプラス/マイナスで貸借を判定する資料もあると思うので、そこらへんはロジックで対応できます。（今回は割愛）

### 並び替えによる伝票の再構成
すべての行を取り込んだ後に並び替えを行うことで、バラバラになっている可能性のある伝票をひとつにまとめます。
弥生会計のインポートでは1つの伝票は連続したデータではないといけないので並び替えは必須です。
今回は日付をキーにしていますが、伝票番号などより伝票を特定できるものがあればそちらでまとめたほうがいいです。

### 連想配列に整形
キーごとの連想配列に整形しています。後述する識別フラグの設定がしやすくなります。今回は割愛しますが、貸借一致の判定も可能になります。

## 識別フラグの設定
連想配列に整理された各グループ（伝票）に対して、行の位置に応じた識別フラグを設定します。
```py
def set_id_flags(yayoi_dict):
    for key in yayoi_dict:
        items = yayoi_dict[key]
        count = len(items)
        
        for i in range(count):
            if i == 0:
                # 伝票の開始行
                items[i].id_flag = 2110
            elif i == count - 1:
                # 伝票の終了行
                items[i].id_flag = 2101
            else:
                # 3行以上の伝票における中間行
                items[i].id_flag = 2100
```

そして、関数を呼び出す処理を```return```の手前に追記します。

```py
set_id_flags(yayoi_dict) # ここに追記

return yayoi_dict
```

設定している数字はインポート形式に則って設定しています。今回は2000、2111に該当する伝票はないので処理に含めていません。

>
仕訳の識別番号を半角数字で記述。
>
伝票以外の仕訳データ   ：2000
1行の伝票データ	   ：2111
複数行の伝票データ
	   1行目		   ：2110
	   間の行		   ：2100
	   最終行		   ：2101


## 弥生会計へのインポートファイル出力
以下のメソッドで整形した連想配列の中身をCSVに出力します。文字コードは弥生会計の仕様に合わせて cp932 (Shift-JIS) を指定します。

```
import csv

def export_to_yayoi_csv(yayoi_dict, output_path):
    with open(output_path, 'w', encoding='cp932', newline='') as f:
        writer = csv.writer(f)
        
        # 整理済みの連想配列を順にループして書き出す
        for day_items in yayoi_dict.values():
            for item in day_items:
                writer.writerow(item.to_list())
```

```item.to_list()```では、クラスで定義した弥生会計の規定している項目を順番に出力しています。

以下のように任意の処理内で呼びます。

```py
export_to_yayoi_csv(data_dict, 'generate/yayoi_import.csv')
```

※フォルダがないとエラーになるので事前に作成するか、フォルダ作成処理は随時入れてください。

## 出力結果

```csv
2110,,2025/12/01,消耗品費,,,対象外,1500,0,,,,,対象外,0,0,文房具代,,,3,,,no
2100,,2025/12/01,,,,対象外,0,0,現金,,,対象外,1500,0,文房具代,,,3,,,no
2100,,2025/12/01,旅費交通費,,,対象外,840,0,,,,,対象外,0,0,電車代,,,3,,,no
2101,,2025/12/01,,,,対象外,0,0,現金,,,対象外,840,0,電車代,,,3,,,no
2110,,2025/12/03,接待交際費,,,対象外,5500,0,,,,,対象外,0,0,会食代,,,3,,,no
2101,,2025/12/03,,,,対象外,0,0,未払金,,,対象外,5500,0,会食代,,,3,,,no
2110,,2025/12/05,消耗品費,,,対象外,2000,0,,,,,対象外,0,0,備品代,,,3,,,no
2101,,2025/12/05,,,,対象外,0,0,現金,,,対象外,2000,0,備品代,,,3,,,no
```

今回は基本的な処理フローのみ解説しましたが、消費税の計算なども自動化できるので次回以降の記事にしたいと思います。