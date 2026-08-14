+++
date = '2026-09-20T20:19:35+09:00'
draft = false
title = 'Laravelで多階層リレーションを安全かつ高速に扱う方法'
description = 'Laravelで$user->purchases->itemsのような多階層リレーションを安全に扱うための方法を解説します。N+1問題とNullエラーの対策を、具体的なコード例とともにご紹介します。'
tags = ['Laravel', 'PHP']
+++

Laravelでは、ユーザー → 購入履歴 → 商品のように、リレーションが2〜3階層以上にわたるデータ構造を扱う場面が多くあります。この構造に何も考えずアクセスすると、N+1問題によるパフォーマンス低下や、Nullアクセスによる実行時エラーが発生しやすくなります。本記事では、その対策として `with()` によるEager Loadingと、`data_get()` / Null Safe演算子によるNull安全なアクセス方法をまとめます。

## 前提となるモデル構成

- ユーザー（User）は複数の購入履歴（Purchase）を持つ
- 各購入履歴（Purchase）は複数の商品（Item）を持つ

**User.php**
```php
class User extends Model
{
    public function purchases()
    {
        return $this->hasMany(Purchase::class);
    }
}
```

**Purchase.php**
```php
class Purchase extends Model
{
    public function items()
    {
        return $this->hasMany(Item::class);
    }
}
```

**Item.php**
```php
class Item extends Model
{
    public function purchase()
    {
        return $this->belongsTo(Purchase::class);
    }
}
```

## 問題点1：N+1問題

ループ内で関連リレーションを都度呼び出すと、データ件数に比例してクエリが発行されます。たとえば100人のユーザーそれぞれに5件の購入履歴があれば、最悪の場合500回以上のクエリが実行されることになります。開発環境ではデータ数が少なく気づきにくいですが、本番でアクセスが増えると表示速度の低下として顕在化します。

**NGなコード例**

```php
$users = User::all();

foreach ($users as $user) {
    foreach ($user->purchases as $purchase) {
        foreach ($purchase->items as $item) {
            echo $item->name;
        }
    }
}
```

`User::all()` の1回に加えて、ユーザーの数だけ `purchases` へのクエリが、さらに購入履歴の数だけ `items` へのクエリが発行されます。ユーザー数・購入履歴数が増えるほど、クエリ数は加速度的に増加します。

**対策：Eager Loading**

```php
$users = User::with('purchases.items')->get();
```

ドット記法で関連先を指定することで、必要なデータを一括取得できます。

## 問題点2：Nullアクセスによるエラー

購入履歴が0件のユーザーや、商品が紐付いていない購入データが存在する場合、以下のようなコードはエラーになります。

**NGなコード例**

```php
$firstItemName = $user->purchases->first()->items->first()->name;
```

`purchases` が空だと `first()` が `null` を返し、続く `->items` で以下のエラーが発生します。

```
Attempt to read property "items" on null
```

このエラーはテストデータが揃っている環境では発生せず、本番で欠損データを持つレコードにアクセスした際に初めて表面化することが多いです。

## 対策：data_get() と Null Safe 演算子

### data_get()

`data_get()` はドット記法で深い階層までアクセスでき、途中でデータが欠けていてもエラーにならず `null` を返します。

```php
$items = data_get($user, 'purchases.0.items');
```

ワイルドカード（`*`）を使うと、全購入履歴に紐づく商品をまとめて取得できます。

```php
$allItems = data_get($user, 'purchases.*.items');
```

### Null Safe演算子（PHP 8.0以上）

単一データへのアクセスには `?->` が簡潔です。

```php
$firstItemName = $user->purchases->first()?->items->first()?->name;
```

途中のいずれかが `null` になっても例外は発生せず、最終結果が `null` になるのみで処理は継続します。

## 使い分け

| 手法 | 用途 | 特徴 |
|---|---|---|
| `with()` | データ取得時 | N+1問題を防止 |
| `data_get()` | 値の取り出し（複数件・配列混在） | 階層が深くてもエラーにならない |
| `?->`（Null Safe） | 単一データへのアクセス | コードが短い |

基本パターンは、`with()` でまとめて取得し、`data_get()` や `?->` で値を取り出す組み合わせです。

## 補足：よくある疑問

**`with()` を使っているのにクエリが減らない場合**

指定した階層が足りていない可能性があります。`purchases.items.category` のように、必要な階層をすべて列挙する必要があります。

**`data_get()` と `pluck()` の違い**

`pluck()` はコレクションの存在が確実な場合の値抽出に向いています。途中の階層がNullになりうる場合は `data_get()` の方が安全です。

**Null Safe演算子の制約**

`?->` はオブジェクトのプロパティ・メソッドチェーンには使えますが、配列アクセス（`[]`）には使えません。配列を含む構造には `data_get()` を使います。

## まとめ

- 取得時は `with()` でEager Loadingし、N+1問題を防ぎます
- アクセス時は `data_get()` または `?->` でNullアクセスによるエラーを防ぎます