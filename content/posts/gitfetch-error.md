+++
date = '2026-08-09T10:25:56+09:00'
draft = false
title = 'git fetch エラー「index.lock 許可がありません」、「The requested URL returned error: 503」が発生した時の対応法'
description = "git fetchを実行した際に発生した「index.lock 許可がありません」、「The requested URL returned error: 503」のエラーについて対応したことをまとめました。"
tags = ['Git']
+++

git fetchを実行した際に発生した「index.lock 許可がありません」、「The requested URL returned error: 503」のエラーについて対応したことをまとめました。権限の変更とconfigファイル内のURLの修正が必要でした。

## 問題①「許可がありません」のエラー

git fetch実行時に以下のエラーが発生しました。

```bash
fatal: Unable to create '/プロジェクト/.git/index.lock': 許可がありません
```

### グループに対して権限を付与
.gitディレクトリの権限を確認したところ、グループに適切な権限が付与されていませんでした。グループに読み書き実行権限（setgid含む）を付与するなど、正しい権限に変更したことでこの問題は解決しました。

```bash
# .git以下のファイル・ディレクトリすべてにグループの読み書き実行権限を付与
$chmod -R g+rwX .git
# ディレクトリにsetgidを付与（新規作成ファイルのグループを自動継承させる）
$find .git -type d -exec chmod g+s {} \;
```

setgidを付与することで、ディレクトリ内にファイルやディレクトリが作成された場合に、それらがそのディレクトリと同じグループで作成されます。

※補足（権限の見方）
```
rw- rws r--
│ │ │ └─ その他 (Others)
│ │ └──── グループ (Group)
│ └──────── 所有者 (Owner)
└────────── 通常ファイル
```

| 対象 | 権限 | 意味 |
| --- | --- | --- |
| 所有者 | rw- | 読み取り・書き込み可、実行不可 |
| グループ | rws | 読み取り・書き込み・実行可 + setgid |
| その他 | r-- | 読み取りのみ |

## 問題②「The requested URL returned error: 503」のエラー

問題①解決後、続けてgit fetchを実行すると今度は以下のエラーが発生しました。

```bash
fatal: unable to access 'リポジトリ': The requested URL returned error: 503
```

### 実際に解決できた対応

.git/configのremote URL に埋め込まれていた「ユーザー名:パスワード」を削除し、認証を都度入力方式に変更したところ、このエラーが解消しました。

```bash
[remote "origin"]
	url = https://ユーザー名:パスワード@ドメイン/リポジトリ名.git
	fetch = +refs/heads/*:refs/remotes/origin/*
↓ユーザー名:パスワード@を削除
[remote "origin"]
	url = https://ドメイン/リポジトリ名.git
	fetch = +refs/heads/*:refs/remotes/origin/*
```

おそらく、埋め込み認証情報を使って何らかのバックグラウンド処理（credential helper／IDEの自動同期など）が認証エラーを引き起こしており、それが503エラーの原因だったと考えられます。認証情報を外したことでこの処理による競合が解消されました。

### 試したが効果がなかった対応

**所有者の変更**

.gitが入っている一つ上のフォルダの所有者がグループ外のユーザーだったため、グループ内のメンバーに所有者を変更したが、エラーは改善されませんでした。

**git config --add safe.directory <パス> の実行**

実行者とフォルダの所有者が不一致の場合にgit操作が制御されるのを回避するためのコマンドだが、これも効果はありませんでした。

## まとめ
- 「index.lock 許可がありません」エラーは、.gitディレクトリのグループ権限不足が原因でした。グループへの読み書き実行権限とsetgidの付与で解消できました。
- 「The requested URL returned error: 503」エラーは、.git/configのremote URLに埋め込まれた認証情報（ユーザー名:パスワード）が原因だった可能性が高く、これを削除し都度認証方式に変更することで解消しました。
- 所有者の変更やsafe.directoryの設定は今回のエラーには効果がなく、権限とURLの両方を正しく見直すことが解決につながりました。