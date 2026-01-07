+++
date = '2026-01-07T14:45:52+09:00'
draft = true
title = '【MySQL】SSH経由でのCSV出力とローカルへのファイル転送方法'
description = 'リモートサーバー上のMySQLからデータを抽出し、ローカル環境へ安全に転送する手順の備忘録です。INTO OUTFILE命令による権限エラーを回避し、データ内の改行を保持したまま出力する方法をまとめます。'
tags = ['MySQL', 'SSH']
+++

リモートサーバー上のMySQLからデータを抽出し、ローカル環境へ安全に転送する手順の備忘録です。INTO OUTFILE命令による権限エラーを回避し、データ内の改行を保持したまま出力する方法をまとめます。

## 出力対象のテーブル例

以下のusersテーブルから条件に合うデータを抽出します。

| カラム名 | データ型 | 論理名 | 備考 |
| :--- | :--- | :--- | :--- |
| id | INT | ユーザーID | 主キー |
| name | VARCHAR | 氏名 | |
| workplace | TINYINT | 勤務地フラグ | 1:北海道, 2:関東, 3:その他 |
| introduction | TEXT | 自己紹介 | 改行や引用符が含まれる可能性あり |
| created_at | DATETIME | 作成日時 | yyyy-mm-dd hh:mm:ss 形式 |

## サーバー側でのCSV出力コマンド
SSHログイン後、シェルから以下のコマンドを実行します。標準出力をリダイレクトすることで、MySQL側のディレクトリ書き込み制限を回避します。

```bash
mysql -h ホスト名 -u ユーザー名 -p -D DB名 -e "
SELECT
    id 'ID',
    name '氏名',
    CASE workplace
        WHEN 1 THEN '北海道'
        WHEN 2 THEN '関東'
        WHEN 3 THEN 'その他'
        ELSE '不明'
    END '勤務地',
    CONCAT('\"', REPLACE(IFNULL(introduction, ''), '\"', '\"\"'), '\"') '自己紹介',
    DATE_FORMAT(created_at, '%Y-%m-%d %H:%i:%s') '作成日'
FROM users
WHERE created_at < '2026-01-07'
ORDER BY created_at DESC;
" | sed 's/\t/,/g' > ~/users_list.csv
```

### コマンドの解説

#### mysql -e "SQL"
対話モードに入らずに SQL を実行します。結果は標準出力として返されます。

#### DATE_FORMAT()
日時型をyyyy-mm-dd HH:mm:ss形式に整形します。

#### | sed 's/\t/,/g'
mysql -e の出力結果（タブ区切り）をカンマ区切りに置換します。

#### > ~/users_list.csv
ログインユーザーのホームディレクトリに書き出します。

## データ内の改行・引用符への対応

introduction（自己紹介）のように、改行やカンマが含まれるデータは、そのまま出力すると CSV の行や列が崩れてしまいます。これを防ぐために、以下のエスケープ処理を施しています。

```sql
CONCAT('"', REPLACE(IFNULL(column, ''), '"', '""'), '"')
```

- IFNULL: NULL値を空文字に置換し、予期せぬ文字列出力を防ぐ
- REPLACE: 項目内の " を "" に置換（RFC 4180準拠）
- CONCAT: 項目全体を " で囲むことで、データ内の改行を「セル内改行」として保持

## ローカルPCへのファイル転送
ファイルの生成完了後、ローカルPCのターミナルからscpコマンドで取得します。

```bash
scp ユーザー名@サーバー名:~/users_list.csv ~/Documents/
```

SSH経由でファイルをコピーするコマンドで、```ユーザー名@ホスト名:~/path```でリモート側のファイルパスを指定します。```~/Documents/```でローカル PC 側の保存先ディレクトリを指定します。

## まとめ
この方法を使えば、データベース側の細かい権限設定を変更することなく、安全かつ正確（改行崩れなし）にデータをエクスポートできます。定期的なレポート作成やデータ分析の際に非常に有効です。