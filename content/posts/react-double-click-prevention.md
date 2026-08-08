+++
date = '2026-08-08T21:09:30+09:00'
draft = false
title = '【React】ボタンの二重クリックを制御する'
description = 'Reactでボタンの二重クリックを制御する方法です。React RouterとプレーンなReactの②パターンでサンプルコードを用意して解説します。'
tags = ['React']
+++

実務でよく使うダブルクリック制御をまとめます。二重送信によるデータの重複登録が懸念される案件が多かったので、設計時に仕様に含めることをおすすめします。特に、決済の二重登録は重大なインシデントになります。
今回は、React RouterとシンプルなReactを使った2パターンをまとめました。


## React Routerを使ったパターン（useNavigationまたはuseFetcherを使用）

### サンプルコード
```tsx
import { Form, useNavigation } from "react-router";

function SubmitForm() {
    const navigation = useNavigation();
    const isSubmitting = navigation.state !== "idle";

    return (
        <Form method="post">
            <input name="title" />
            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? "送信中..." : "送信"}
            </button>
        </Form>
    );
}
```

### サンプルコードの解説

React Routerは、useNavigationというフックが使えるのでこちらで状態を管理してボタンの制御を行います。このフックはグローバルなナビゲーション状態（フルページ遷移のsubmit）を追跡できます。
ボタンを押して送信中になるととstateプロパティがidleからsubmittingに変化するので、ボタンが非活性に変化します。stateプロパティには以下のようなステータスがあります。

| ステータス   | タイミング                                                                 |
|---------|------------------------------------------------------------------------|
| idle       | 通信・遷移が発生しておらず、何も実行していない状態                                  |
| submitting | フォームが送信され、action（データの書き込み処理）が実行されている状態              |
| loading    | action完了後、ルートのloaderが実行されている状態 |

いいねボタンなど画面遷移を伴わない場合は、useFetcherというフックも使用できます。こちらもReact Routerの機能で、サーバーサイドの処理が完了して結果が返るまで自動的にstate がsubmitting→loading→idleと遷移するので、自前でフラグ管理する必要がありません。

```tsx
import { useFetcher } from "react-router";

function LikeButton({ postId }: { postId: string }) {
    const fetcher = useFetcher();
    const isSubmitting = fetcher.state !== "idle";

    return (
        <fetcher.Form method="post" action='/'>
            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? "処理中..." : "いいね"}
            </button>
        </fetcher.Form>
    );
}

```

## プレーンなReactで実装する場合（useRefとuseState）

### サンプルコード

```tsx
import { useRef, useState } from "react";

function SubmitButton() {
    const isSubmittingRef = useRef(false);
    const [isSubmitting, setIsSubmitting] = useState(false);

    const handleClick = async () => {
        if (isSubmittingRef.current) return;
        isSubmittingRef.current = true;
        setIsSubmitting(true);

        try {
            const res = await fetch("/api/posts", { method: "POST" });
            if (!res.ok) throw new Error("failed");
        } catch (e) {
            console.error(e);
        } finally {
            isSubmittingRef.current = false;
            setIsSubmitting(false);
        }
    };

    return (
        <button onClick={handleClick} disabled={isSubmitting}>
            {isSubmitting ? "送信中..." : "送信"}
        </button>
    );
}

```

### サンプルコードの解説
まず、定数isSubmittingRefとisSubmittingを用意します。関数useRefを使って、定数isSubmittingRefにフラグの初期状態を設定します。useRefは再レンダリングしても値を保持する関数です。送信中も非活性状態を保持するために使用します。
isSubmittingはボタンのdisabled属性に反映し、非活性化するために使用します。
ボタンをクリックすると関数handleClickが実行されます。以下の順番で処理を行います。
- isSubmittingRef.currentがtrue（＝既に送信中）なら即returnし、以降の処理を一切実行しない
- まだ送信中でなければ、定数isSubmittingRefとisSubmittingを両方trueにして送信中としつつ、axiosでサーバーサイドと通信を行います。
- catchで通信エラー時のハンドリングを行い、finallyで成功・失敗どちらの場合も必ずisSubmittingRefとisSubmittingをfalseに戻します。

## まとめ
- React RouterとプレーンなReactのどちらのパターンでも、ボタンを非活性にすることで二重クリックを防ぐ基本的な考え方は同じです。
- React Routerを使う場合はuseNavigationやuseFetcherが送信状態を自動で管理してくれるため、自前でフラグを持つ必要がありません。
- 決済など重複登録が深刻な影響を与える処理では、設計の段階から二重クリック制御を仕様に含めることをおすすめします。