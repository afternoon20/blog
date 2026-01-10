+++
date = '2025-12-28T11:13:41+09:00'
draft = false
title = '【React】条件付きProps展開'
description = 'Reactで特定の値がある場合のみPropsを渡すテクニック。undefinedを渡さず、キー自体を存在させないスプレッド構文の使い方を解説します。'
tags = ['React']
+++

Reactでコンポーネントを設計している際、値が空のときはPropsのキー自体を渡したくない（undefinedを渡したくない）場面がありました。
例えば、HTML属性に余計な属性を付与したくない場合や、子コンポーネント側で```defaultProps```を適用させたい場合です。

論理演算子とスプレッド構文を組み合わせることで、簡潔に条件付き展開が可能です。

## 基本構文

```JavaScript
{...(条件 && { キー名: 値 })}
```

## 実践的なコード例
### 関数コンポーネント

```user_id```が存在する場合のみ、子コンポーネントへ渡します。

```JavaScript
const Parent = () => {
  const user_id = "12345";

  return (
    <Child 
      {...(user_id && { user_id })} 
    />
  );
};
```

### 子コンポーネントでの受け取り方

```JavaScript
const Child = ({ user_id = "default_id" }) => {
  return <div>{user_id}</div>;
};
```

**なぜ```user_id={user_id}```ではいけないのか**

通常、```user_id={user_id}```と記述すると、user_id が空の場合は```user_id={undefined}```として渡ります。
しかし、このスプレッド構文の手法を使うと、条件が偽のときはオブジェクトの展開が行われないため、子コンポーネント側には```user_id```というキー自体が渡りません。
これにより、子コンポーネント側で設定しているデフォルト値（Default Parameters）を正しく機能させることができます。


## クラスコンポーネントでの記述

```JavaScript
class Parent extends React.Component {
  render() {
    const { user_id } = this.props;

    return (
      <Child 
        {...(user_id && { user_id })} 
      />
    );
  }
}
```

## まとめ
この記述方法は、複数のオプション引数を持つUIコンポーネント（ボタンや入力フォームなど）をラップする際に、コードをスッキリ保つのに役立ちます。
