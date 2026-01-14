+++
date = '2023-12-28T15:47:25+09:00'
draft = false
title = '【FuelPHP】find_by_pkで「id」カラムエラーが出る原因と解決策'
description = "FuelPHPのModel_Crudで「Unknown column 'id'」エラーが出る原因と解決策を解説。主キー名がid以外の場合に必要な$_primary_keyの設定方法を解説します。"
tags = ['PHP', 'FuelPHP']
+++

FuelPHPの Model_Crud を使用して、find_by_pk() などのメソッドを実行した際、以下のようなエラーが発生することがあります。

```
SQLSTATE[42S22]: Column not found: 1054 Unknown column 'id' in 'where clause'
```

これは、テーブルの主キー名がid以外（例：user_idなど）であるにもかかわらず、プログラム側でその指定が漏れている場合に発生します。

## エラーが発生する原因
FuelPHPのModel_Crudは、テーブルの主キー名はデフォルトでidであるというルール（規約）に基づいて動作します。
そのため、モデル側で何も指定せずに```find_by_pk(10)```などを実行すると、内部的には```WHERE id = 10```というSQLを生成しようとします。ここで実際のテーブルにidというカラムが存在しないため、前述のエラーが投げられます。

## 解決策：モデルクラスに主キーを明示する
解決方法は非常にシンプルです。モデルクラスのプロパティ```$_primary_key```を定義し、実際のテーブルで使用している主キー名を指定するだけです。

### 修正後のコード例
例えば、主キー名が```user_id```の場合は以下のように記述します。

```php
class Model_User extends \Model_Crud
{
    protected static $_table_name = 'user';
    protected static $_primary_key = 'user_id';
}
```

このように設定することでfind_by_pk()やdelete()を実行した際、正しくWHERE user_id = ...というSQLが生成されるようになります。

### 補足：Model_Crudで設定可能な主要パラメーター
主キー以外にも、実務でよく使われる設定値を紹介します。これらを適切に設定することで、バリデーションやタイムスタンプの管理を自動化できます。

| パラメーター名 | 説明 | コード例 |
| :--- | :--- | :--- |
| **`$_rules`** | バリデーションルールの設定 | `'age' => 'required'` |
| **`$_labels`** | バリデーションエラー時のラベル名 | `'email' => 'メールアドレス'` |
| **`$_created_at`** | 作成日時を保存するフィールド名 | `'user_created_at'` |
| **`$_updated_at`** | 更新日時を保存するフィールド名 | `'user_updated_at'` |

### 注意点：複合主キーについて
Model_Crud は複合主キー（複数のカラムを組み合わせた主キー）に対応していません。もし複合主キーを持つレガシーなテーブルなどを扱う必要がある場合は、Model_CrudではなくOrm\Modelへの移行を検討してください。

## Laravel（Eloquent）でも考え方は同じ
LaravelのEloquent ORMにおいても、デフォルトの主キーは id と想定されています。主キー名が異なる場合は、モデル内で以下のように指定が必要です。

```php
protected $primaryKey = 'user_id';
```