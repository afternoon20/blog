+++
date = '2025-12-28T11:13:41+09:00'
draft = false
title = '【React】条件付きProps展開'
description = 'Reactで条件付きでPropsを展開する際の設定方法を解説します。'
tags = ['React']
+++

値が存在する場合のみ、Propsのキーごと子コンポーネントへ渡す記述方法を解説します。
## 構文

```JavaScript
{...(条件 && { キー名: 値 })}
```

## コード例
### 関数コンポーネント
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

## クラスコンポーネント

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

## 仕様
条件が真の場合: { user_id: "12345" } が展開され、Propsに user_id が含まれます。
条件が偽の場合: false が展開され、Propsに user_id は含まれません（undefined が渡るのではなく、キー自体が存在しない）。
