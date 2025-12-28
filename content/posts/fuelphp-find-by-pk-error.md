+++
date = '2023-12-28T15:47:25+09:00'
draft = true
title = '【FuelPHP】find_by_pkで「id」カラムエラーが出る原因と解決策'
description = "FuelPHPで「Unknown column 'id'」エラーが出る原因と解決策を解説。主キー名がid以外の場合に、モデル側で設定すべき$_primary_keyの使い方をまとめました。"
tags = ['PHP', 'FuelPHP']
+++

FuelPHPの Model_Crud を使用して、find_by_pk() などのメソッドを実行した際、以下のようなエラーが発生することがあります。

```
SQLSTATE[42S22]: Column not found: 1054 Unknown column 'id' in 'where clause'
```

これは、モデル側で主キーを指定していない場合、FuelPHPがデフォルトの主キー名として id を参照しようとするためです。

## 解決策：モデルクラスに主キーを明示する
テーブルの主キーが```id```以外の名称（例：```user_id```や```entry_id```）である場合は、モデルの ```$_primary_key```プロパティを設定することで解決します。

### 修正後のコード例

```php
class Model_User extends \Model_Crud
{
    protected static $_table_name = 'user';
    protected static $_primary_key = 'user_id';
}
```

このように設定することでfind_by_pk()やdelete()を実行した際、正しくWHERE user_id = ...というSQLが生成されるようになります。

### 補足：Model_Crudで設定可能な主要パラメーター
主キー以外にも、よく使われる設定値は以下のとおりです。これらを活用することで、開発効率を高めることができます。

| パラメーター名 | 説明 | コード例 |
| :--- | :--- | :--- |
| **`$_rules`** | バリデーションルールの設定 | `'age' => 'required'` |
| **`$_labels`** | バリデーションエラー時のラベル名 | `'email' => 'メールアドレス'` |
| **`$_created_at`** | 作成日時を保存するフィールド名 | `'user_created_at'` |
| **`$_updated_at`** | 更新日時を保存するフィールド名 | `'user_updated_at'` |

### 注意点：複合主キーについて
Model_Crud は複合主キー（複数のカラムを組み合わせた主キー）に対応していません。もし複合主キーのテーブルを扱う必要がある場合は、Model_CrudではなくOrm\Modelへの移行を検討してください。

## Laravel（Eloquent）でも考え方は同じ
LaravelのEloquent ORMにおいても、デフォルトの主キーは id と想定されています。主キー名が異なる場合は、モデル内で以下のように指定が必要です。

```php
protected $primaryKey = 'user_id';
```