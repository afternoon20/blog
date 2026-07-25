+++
date = '2026-07-25T17:45:41+09:00'
draft =  false
title = 'Laravelプロジェクトを複数人でgit操作できるようにするパーミッション設定'
description = 'サーバー上の共有ディレクトリに配置したLaravelプロジェクトを、特定の1ユーザーだけでなく同じグループに所属する複数人でgit操作できるようにするための手順です。'
tags = ['Git', 'Linux', 'Laravel']
+++

サーバー上の共有ディレクトリに配置したLaravelプロジェクトを、特定の1ユーザーだけでなく同じグループに所属する複数人でgit操作できるようにするための手順です

## 前提条件

- 作業実行ユーザー（以下、user）は、対象プロジェクトの共有グループ（以下、group）に既に所属している
- プロジェクトの初期の所有者・グループは user:group
- 対象パスは/srv/laravel-appとする

## 手順

### 変更前の状態を確認

```bash
ls -la /srv/laravel-app
```

作業前の所有者・パーミッションを記録しておきます。後で差分を確認するために必須です。

### Gitを複数人共有モードにする

```bash
cd /srv/laravel-app
git config --global --add safe.directory /srv/laravel-app
git config core.sharedRepository group
cat .git/config
```

- safe.directory: リポジトリの所有者と実行ユーザーが異なる場合にgitが出す「dubious ownership」エラーを回避するための設定です。この設定は--globalのためコマンドを実行したユーザー個人にしか効きません。他のユーザーも触る場合は、そのユーザーごとに個別に実行するか、--system（要sudo）で全ユーザーに一括適用します。
- core.sharedRepository = group: 以後gitが新規作成するオブジェクト（コミット、blobなど）のパーミッションを、実行ユーザーのumaskに関係なく常にグループ書き込み可能にする設定です。.git/configの [core] セクションに sharedRepository = groupが入っていることを確認します。

### 所有者・グループを揃える

Webサーバーが書き込む前提のディレクトリ（vendor、storage配下の一部）は除外して chownします。

```bash
cd /srv
sudo find laravel-app \
 -path "laravel-app/vendor" -prune -o \
 -path "laravel-app/storage/app/private" -prune -o \
 -path "laravel-app/storage/debugbar" -prune -o \
 -path "laravel-app/storage/framework" -prune -o \
 -path "laravel-app/storage/logs" -prune -o \
 -exec chown user:group {} +
```

findの起点引数がlaravel-app（.ではない）のため、-pathの比較対象パスも laravel-app/...から始まります。ここが ./vendorのような相対表記になっていると除外条件が一致せず、除外したいディレクトリまでchownされてしまうので注意してください。実行前に -exec chown ... {} +を-printに置き換えて、除外対象がヒットしていないかドライランで確認してから本実行するとよいでしょう。

### グループに読み書き権限を付与

```bash
sudo chmod -R g+rwxs /srv/laravel-app
```

- ディレクトリに setgid (s) を付けることで、以後配下に新規作成されるファイル・ディレクトリのグループ所有者が自動的にgroupに揃います。
- core.sharedRepository=groupと組み合わせることで、今後user以外のグループメンバーがpull/pushしても、新規に作られるファイルのパーミッションが崩れません。

### .git配下のファイルからsetgidだけ外す

setgidはディレクトリには意味がありますが、通常ファイルに付いていても実質的な効果はありません。グループに読み書き権限を付与で.git配下のファイルにも一律setgidが付くため、ファイルのみ除去します。

```bash
find /srv/laravel-app/.git -type f -exec sudo chmod g-s {} +
```

owner・otherのビットは変更されないため、例えば.git/HEADや.git/configは -rw-rwxr--（owner: rw-, group: rwx, other: r--）になります。

### Gitフックの実行権限を確認

post-checkout / post-mergeなどカスタムフックを使っている場合、実行ビットが落ちていないか確認します。

```bash
cd /srv/laravel-app/.git/hooks
sudo chmod 775 post-checkout post-merge
ls -la
```

## 新しいメンバーが増えたとき

上記の所有者・パーミッション変更は一度行えばリポジトリ側に残るため、後から別のメンバー（例: user2）が加わっても手順3〜6をやり直す必要はありません。必要なのは以下の2点のみです。

1. user2をgroupに追加する（OS側のグループ設定。パーミッション上の実際の読み書き権限はここで初めて付与されます）
2. user2自身のアカウントでgit config --global --add safe.directory /srv/laravel-appを実行する（安全性チェックの回避は個人設定のため、各ユーザーが自分の環境で設定する必要があります）