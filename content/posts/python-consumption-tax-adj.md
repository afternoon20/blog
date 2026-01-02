+++
date = '2026-01-02T00:28:38+09:00'
draft = false
title = '【Python】弥生会計のインポートクラスの応用｜本体行と消費税行を合算'
description = '以前の記事で作成した弥生会計のインポートクラスを使用して、消費税を含めた伝票データを作成する方法を解説します。'
tags = ['Python', '弥生会計']
+++

[Pythonで弥生会計のインポートファイルを作成する記事](/posts/yayoi-to-csv/)の続きです。今回は同じ伝票内にある消費税行を本体行に含める処理を紹介します。インポートファイルの形式などは前回の記事を参照してください。

※実際にインポートする際は、事前にバックアップを取ることをお勧めします。

## 合算処理を行う仕訳データのサンプル

以下のデータを、前回作成した弥生クラスの配列に設定されていることとします。

| A:取引日 | B:借方科目 | C:借方金額 | D:貸方科目 | E:貸方金額 | F:摘要 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 2025/12/01 | 消耗品費 | 1,500 | | | 文房具代 |
| 2025/12/01 | 仮払消費税 | 150 | | | 文房具代 |
| 2025/12/01 | 通信費 | 80 | | | 切手代 |
| 2025/12/01 | | | 現金 | 1,730 | 文房具店 |
| 2025/12/02 | 交際費 | 2,000 | | | お昼代 |
| 2025/12/02 | 仮払消費税 | 160 | | | お昼代 |
| 2025/12/02 | | | 現金 | 2,160 | コンビニエンスストア |
| 2025/12/03 | 未収入金 | 11,000 | | | コンサル売上 |
| 2025/12/03 | | | 売上 | 10,000 | コンサル売上 |
| 2025/12/03 | | | 仮受消費税 | 1,000 | コンサル売上 |

## Pythonによる消費税行の特定と合算ロジック

- 伝票単位でのループ処理
- 消費税行の特定
- 税率（10%または8%切り捨て）が一致する本体行を特定し、以下を行う
    - 借方（貸方）税金額に消費税行の金額を設定
    - 税率によって借方（貸方）税区分を設定
    - 消費税行(クラスインスタンス)を配列から削除

具体例を挙げると以下の取引データがある場合、

```csv
2110,,2025/12/01,消耗品費,,,対象外,1500,0,,,,,対象外,0,0,文房具代,,,3,,,no
2100,,2025/12/01,仮払消費税,,,対象外,150,0,,,,,対象外,0,0,文房具代,,,3,,,no
2101,,2025/12/01,,,,対象外,0,0,未払金,,,対象外,1650,0,文房具代,,,3,,,no
```

以下のように整形されます。

- 借方金額1,500円 → 1,650円
- 借方税金額 → 150円
- 借方税区分 → 課税対応仕入10%
- 仮払消費税の行を削除（２行目）

```csv
2110,,2025/12/01,消耗品費,,,対象外,1650,150,課税対応仕入10%,,,,対象外,0,0,文房具代,,,3,,,no
2101,,2025/12/01,,,,対象外,0,0,未払金,,,対象外,1650,0,文房具代,,,3,,,no
```

## 【実装コード】消費税行を本体行へ合算する関数

実際のコードは以下のとおりです。

```py
def merge_tax_rows(yayoi_dict):
    tax_account_names = ['仮払消費税', '仮受消費税']
    
    for key in yayoi_dict:
        items = yayoi_dict[key]
        tax_rows = [i for i in items if i.debit_name in tax_account_names or i.credit_name in tax_account_names]
        
        for tax_item in tax_rows:
            target_found = False
            
            for target_item in items:
                if target_item in tax_rows:
                    continue
                
                if tax_item.debit_name == '仮払消費税' and tax_item.debit_amount > 0:
                    if target_item.debit_amount > 0:
                        base = target_item.debit_amount
                        tax = tax_item.debit_amount
                        if int(base * 0.1) == tax:
                            target_item.debit_amount += tax
                            target_item.debit_tax = tax
                            target_item.debit_tax_type = '課税対応仕入10%'
                            target_found = True
                        elif int(base * 0.08) == tax:
                            target_item.debit_amount += tax
                            target_item.debit_tax = tax
                            target_item.debit_tax_type = '課税対応仕入8%(軽)'
                            target_found = True
                            
                elif tax_item.credit_name == '仮受消費税' and tax_item.credit_amount > 0:
                    if target_item.credit_amount > 0:
                        base = target_item.credit_amount
                        tax = tax_item.credit_amount
                        if int(base * 0.1) == tax:
                            target_item.credit_amount += tax
                            target_item.credit_tax = tax
                            target_item.credit_tax_type = '課税売上10%'
                            target_found = True

                if target_found:
                    items.remove(tax_item)
                    break
```

関数の呼び出しは、前回コード内の「# 日付ごとの連想配列に整理」の次に以下の形で追記します。

```py
merge_tax_rows(yayoi_dict)
```

## 数値計算による正確な紐付けロジックの解説

### 消費税行の特定

伝票データの中から、科目名が「仮払消費税」または「仮受消費税」となっている行を抽出し、合算処理のための対象リスト（tax_rows）に格納します。

### 税額に一致する本体行の特定

抽出した消費税行の金額が、同じ伝票内にある他の行（本体行）の金額に対して、10%または8%を掛けた値と一致するかを判定します。これにより、摘要などの文字列に頼らず数値ベースで正確に本体行を特定します。

### 税額を本体行に設定

紐付けが成功した場合、本体行の金額に消費税額を加算して税込金額に更新します。同時に、弥生会計の仕様に基づき「税金額」プロパティに消費税額を、また「税区分」には判定された税率に応じて「課税対応仕入10%」「課税対応仕入8%(軽)」「課税売上10%」のいずれかを設定します。

### 消費税行の削除

本体行への合算処理が完了した消費税行は、items.remove() を使用して元のリストから削除します。これにより、インポート用CSVには消費税が外出しされていない、税込形式の仕訳行のみが出力されるようになります。

### 出力されるCSV

```csv
2110,,2025/12/01,消耗品費,,,課税対応仕入10%,1650,150,,,,,対象外,0,0,文房具代,,,3,,,no
2100,,2025/12/01,通信費,,,対象外,80,0,,,,,対象外,0,0,切手代,,,3,,,no
2101,,2025/12/01,,,,対象外,0,0,現金,,,対象外,1730,0,文房具店,,,3,,,no
2110,,2025/12/02,交際費,,,課税対応仕入8%(軽),2160,160,,,,,対象外,0,0,お昼代,,,3,,,no
2101,,2025/12/02,,,,対象外,0,0,現金,,,対象外,2160,0,コンビニエンスストア,,,3,,,no
2110,,2025/12/03,未収入金,,,対象外,11000,0,,,,,対象外,0,0,コンサル売上,,,3,,,no
2101,,2025/12/03,,,,対象外,0,0,売上,,,課税売上10%,11000,1000,コンサル売上,,,3,,,no
```

今回のロジックでは、消費税額から逆算して金額比が一致した行のみを「本体」として処理します。 そのため、上記の前提データにある「通信費 80円（切手代）」のように、税率計算が一致しない行は合算されず、そのまま独立した行として出力されます。

## 補足：科目名が不明な場合の金額比率による推定合算

科目名が「仮払消費税」等になっておらず、科目名だけで消費税行を特定できない場合は、金額の比率から「消費税らしき行」を推測して合算する方法が有効です。

コードは以下の通りです。

```py
def merge_tax_by_rate(yayoi_dict):
    for key in yayoi_dict:
        items = yayoi_dict[key]
        
        for tax_item in items[:]:
            if tax_item not in items: continue
            
            for target_item in items:
                if tax_item == target_item: continue
                
                is_match = False
                
                # 借方側の判定
                if tax_item.debit_amount > 0 and target_item.debit_amount > 0:
                    base = target_item.debit_amount
                    tax = tax_item.debit_amount
                    if int(base * 0.1) == tax:
                        target_item.debit_amount += tax
                        target_item.debit_tax = tax
                        target_item.debit_tax_type = '課税対応仕入10%'
                        is_match = True
                    elif int(base * 0.08) == tax:
                        target_item.debit_amount += tax
                        target_item.debit_tax = tax
                        target_item.debit_tax_type = '課税対応仕入8%(軽)'
                        is_match = True
                
                # 貸方側の判定
                elif tax_item.credit_amount > 0 and target_item.credit_amount > 0:
                    base = target_item.credit_amount
                    tax = tax_item.credit_amount
                    if int(base * 0.1) == tax:
                        target_item.credit_amount += tax
                        target_item.credit_tax = tax
                        target_item.credit_tax_type = '課税売上10%'
                        is_match = True
                
                if is_match:
                    items.remove(tax_item)
                    break
```

ただし、たまたま金額比が一致するだけの無関係な本体行（例：1,000円の消耗品費と10,000円の旅費交通費が並んでいる場合など）があると、誤って合算されてしまうリスクがあるため注意が必要です。

