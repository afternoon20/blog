+++
date = '2025-12-28T11:13:41+09:00'
draft = false
title = '【React】条件付きProps展開方法とundefinedを渡さずキー自体を除外する方法'
description = 'Reactで値が空のときにPropsのキー自体を渡さない方法を解説します。undefinedを渡した場合との挙動の違い、TypeScriptでの型定義、複数Props・aria属性・disabled属性への応用例を紹介します。'
tags = ['React']
+++

Reactでコンポーネントを設計している際、値が空のときはPropsのキー自体を渡したくない（undefinedを渡したくない）場面がありました。
例えば、HTML属性に余計な属性を付与したくない場合や、子コンポーネント側で`defaultProps`を適用させたい場合です。

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

### undefinedを渡した場合との挙動の違い

以下のコードで挙動の違いを確認できます。

```javascript
<Child user_id={undefined} /> //①
<Child {...(false && { user_id: "12345" })} /> //②
```

①の場合、子コンポーネントには`user_id`というキーが存在しますが値は`undefined`です。
Reactの`defaultProps`はキーが存在しない場合にのみ適用されるため、①ではデフォルト値が適用されません。

②の場合、条件が偽のときはスプレッド展開が行われないため、子コンポーネントには`user_id`というキー自体が存在しません。
そのためデフォルト値が正しく適用されます。

```javascript
const Child = ({ user_id = "default_id" }) => {
  return <div>{user_id}</div>;
};
// ①の結果: undefined（デフォルト値が適用されない）
// ②の結果: "default_id"（デフォルト値が適用される）
```

## 複数のPropsをまとめて条件付き展開する

条件が同じ複数のPropsをまとめて展開することもできます。

```javascript
const Parent = () => {
  const isAdmin = true;

  return (
    <Child
      {...(isAdmin && {
        role: "admin",
        canEdit: true,
        canDelete: true,
      })}
    />
  );
};
```

個別に書く場合と比べてコードがスッキリします。

```javascript
<Child
  role={isAdmin ? "admin" : undefined}
  canEdit={isAdmin ? true : undefined}
  canDelete={isAdmin ? true : undefined}
/>
```

## TypeScriptでの書き方

TypeScriptを使う場合、受け取るコンポーネント側でPropsの型定義をオプショナルにしておく必要があります。

```typescript
type ChildProps = {
  user_id?: string;
  role?: string;
  canEdit?: boolean;
};

const Child = ({ user_id = "default_id", role, canEdit }: ChildProps) => {
  return (
    <div>
      <p>{user_id}</p>
      <p>{role}</p>
      <p>{canEdit ? "編集可能" : "編集不可"}</p>
    </div>
  );
};
```

## クラスコンポーネントでの記述

```javascript
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

## よくあるユースケース

### disabled属性を条件付きで付与する

ボタンの活性・非活性をPropsで制御する場合

```javascript
const Button = ({ isLoading, onClick, children }) => {
  return (
    <button
      onClick={onClick}
      {...(isLoading && { disabled: true })}
    >
      {children}
    </button>
  );
};
```

### aria属性を条件付きで付与する

アクセシビリティ対応でaria属性を動的に付与する場合

```javascript
const Modal = ({ isOpen, children }) => {
  return (
    <div
      {...(isOpen && { "aria-modal": true, role: "dialog" })}
    >
      {children}
    </div>
  );
};
```

## この記事のまとめ

`{...(条件 && { キー名: 値 })}`の構文を使うことで、条件が偽のときはPropsのキー自体を渡さないようにできます。

- `undefined`を渡すのとは異なり、デフォルト値が正しく機能する
- 複数のPropsをまとめて条件付き展開できる
- `disabled`や`aria`属性など、HTML属性の動的付与にも活用できる
- TypeScriptではPropsの型定義を`?`でオプショナルにする

UIコンポーネントをラップする際にコードをスッキリ保つのに役立つテクニックです。