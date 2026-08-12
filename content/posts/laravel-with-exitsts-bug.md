+++
date = '2026-08-12T22:30:24+09:00'
draft = false
title = '【Laravel】Laravel Eloquent::withExistsで追加したカラムが0やbooleanになる問題の解決'
description = "LaravelのwithExistsで追加した動的カラムが0やbooleanになる問題の原因と解決策を解説します。アクセサの上書きや自動型変換、select句によるカラム消失など3つの原因と、castsによる解決方法を紹介します。"
tags = ['Laravel', 'PHP']
+++

Laravel の`withExists`や`withCount`を利用して「お気に入り済みフラグ」などの動的カラムを実装する際、期待した値(1 / 0)が返らず、`true`/`false`になったり、データがあるはずなのに`0`に固定される問題が発生しました。

---

## 発生した問題

`posts`(投稿)テーブルに対して、ログインユーザーがその投稿を`favorites`(お気に入り)テーブルに登録済みかどうかを`post_is_favorite`という動的カラムで一緒に取得したいというのが、今回のシチュエーションです。Xや掲示板でよく見られるいいねやお気に入りの情報を投稿の一覧とともに取得します。

Eloquentの`withExists()`を使うと、実際には以下のようなSQLが発行されます。
 
```sql
select
  `posts`.*,
  exists(
    select 1
    from `favorites`
    where `favorites`.`target_id` = `posts`.`id`
      and `favorites`.`user_id` = ?
  ) as `post_is_favorite`
from `posts`
```
 
このSQLをDB上で直接実行すると、対象のお気に入りレコードは存在しており、`post_is_favorite`は`1`が返ってきます。ところが、同じSQLが発行されているはずなのに、APIのレスポンスでは矛盾した結果になってしまいました。
 
修正前は`post_is_favorite: 0`となり、存在するはずのデータが存在しないことになってしまいます。アクセサ(Accessor)を調整しても解決せず、今度は`true`(boolean)で返ってしまうこともあり、フロントエンド(React/TypeScript)側の`number`型定義と噛み合わなくなってしまいました。
 
SQL単体を実行すれば正しい値が返るのに、API経由だと値がおかしくなってしまうというのが、この問題の厄介なところです。原因はSQLそのものではなく、SQLが実行された後のLaravel/Eloquent内部の処理にありました。
 
---
 
## 主な原因
 
原因は大きく分けて3つありました。
 
### アクセサの上書き
動的に追加したカラム(`post_is_favorite`)に対してモデル側でアクセサを定義している場合、引数`$value`に正しく値が渡らない、あるいは`null`が渡ってしまうケースがあります。その結果、内部ロジック`return $value ? 1 : 0;`が常に`0`を返してしまい、SQLの結果を上書きしてしまっていました。
 
### Laravelによる自動型変換
Laravelのクエリビルダ経由でexists句を実行すると、DB側が1を返していても、Laravelが自動的にPHPのboolean型(true/false)へ変換してしまいます。
アクセサが定義されている場合、$castsよりもアクセサが優先して評価されるため、$castsでintegerを指定していても、アクセサ側で$valueが正しく受け取れず
0を返し続けてしまい、$castsの変換が意味をなさなくなってしまいます。
 
### select句による上書き
クエリ構築の過程で`addSelect`を使用しても、その後に`select()`や、内部で`select()`を実行するメソッドが呼ばれると、それまでの選択カラムがリセットされ、動的カラムが消失してしまいます。
 
## 修正前の実装
 
修正前の実装と、先述した3つの原因を確認します。
クエリ側では、`withExists`を使ってリレーションの存在チェックを行っていました。
 
```php
$query = Post::query()
    ->withExists(['favorites as post_is_favorite' => function ($q) use ($userId) {
        $q->where('user_id', $userId);
    }]);
```
 
さらにこの後で、一覧取得用の`select()`が別途呼ばれていました。
 
```php
$query->withExists([...]);
 
// ここでカラムを絞り込む処理が後から入っていた
$query->select([
    'id',
    'title',
    'body',
    // ...
]);
```
 
このため、せっかく追加した`post_is_favorite`が`select()`のタイミングでリセットされて消えてしまっていました。
モデル側のアクセサは、以下のような実装でした。
 
```php
public function getPostIsFavoriteAttribute($value)
{
    return $value ? 1 : 0;
}
```
 
一見正しそうな実装ですが、動的に追加したカラムのため`$value`にモデルの生の属性がうまく渡らず、`$value ? 1 : 0`が常にelse側の`0`に落ちてしまっていました。

## 対応策　castsによる型固定
`withExists`自体はそのまま維持し、アクセサを削除して`$casts`で数値型に変換する方法をとりました。
以下のように`getPostIsFavoriteAttribute`メソッドを削除して、`$casts`を定義します。
 
```php
// このアクセサを削除
public function getPostIsFavoriteAttribute($value)
{
    return $value ? 1 : 0;
}

// こちらを定義
protected $casts = [
    'post_is_favorite' => 'integer',
];
```
 
クエリ側は特に変更していません。
 
```php
// そのまま
$query = Post::query()
    ->withExists(['favorites as post_is_favorite' => function ($q) use ($userId) {
        $q->where('user_id', $userId);
    }]);
```

`(int) true`は`1`、`(int) false`は`0`になるため、`withExists`が返す`boolean`値も`$casts`の`integer`指定で`1`/`0`に変換されます。今回の不具合の本質は、`withExists`がboolean値を返すこと自体ではなく、その手前でアクセサが`$value`を正しく受け取れず`0`を返し続けていた点にありました。アクセサを取り除けば、`$casts`のキャストだけで意図通りの値になります。

## 本記事まとめ
- 型変換はモデル側の`$casts`に任せる。アクセサで独自に`1`/`0`へ変換しようとすると、動的カラムの値がうまく渡らず不具合の原因になる
- `withExists`のboolean値も`$casts`の`integer`指定で`1`/`0`に変換できるため、クエリ側を変更する必要はない

