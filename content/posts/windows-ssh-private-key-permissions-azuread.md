+++
date = '2026-10-23T23:39:57+09:00'
draft = false
title = '【Windows】ユーザー移行後のSSH「UNPROTECTED PRIVATE KEY FILE」エラーの直し方（Azure AD対応）'
description = 'Windowsでユーザー移行後に秘密鍵をコピーすると、SSHで「UNPROTECTED PRIVATE KEY FILE」「bad permissions」エラーが出ます。Azure ADアカウントでGUIから所有者を変更できない場合の、icaclsコマンドでの直し方を解説します。'
tags = ['Windows', 'SSH']
+++

PCのログインユーザーを変更した（変更後のユーザーはAzure AD（現Microsoft Entra ID）＝職場・学校アカウント）ことに伴って発生したSSH接続エラーと、その対応をまとめます。

## やったこと
ユーザーが変わったため、`C:\Users\元のユーザー\.ssh`フォルダーの中身を、`C:\Users\変更後のユーザー\.ssh`へそのままコピーしました。その後、sshを実行しました。

## 発生したエラー
秘密鍵ファイルの権限エラーが出ました（以下は秘密鍵が`id_rsa`の場合の例です）。
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions for 'id_rsa' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "id_rsa": bad permissions
```
あわせて、鍵が使われなかったことによる認証エラーも出ていました。

## 原因
コピーした秘密鍵ファイルのアクセス権（所有者・閲覧できるユーザー）が、元のユーザーのまま、または他のユーザーも読める状態になっていました。
Windows版OpenSSHは、秘密鍵を変更後のユーザー（とSYSTEM / Administrators）以外が読めると、安全でないとみなして鍵を使いません。

- `config`ファイルも、権限が緩いと「Bad owner or permissions on ...config」という別のエラーになることがあります。その場合も、下記の手順で`<秘密鍵ファイル>`を`config`に置き換えれば直せます。

## 試したがうまくいかなかったこと
エクスプローラーで秘密鍵ファイルを右クリック→「プロパティ」→「セキュリティ」タブ→「詳細設定」→所有者の「変更」から、所有者を変更しようとしました。しかし、「ユーザーの選択」画面でユーザー名・メールアドレスを入力しても「見つからない」「場所にない」と表示されました。

この画面ではAzure ADのアカウントを検索できないためです（場所にローカルPCしか表示されません）。同じ現象は以下でも報告されています。

- [How to add file permissions (or anything else) for AzureAD users on AzureAD-joined Win10 machine?（Microsoftの旧フォーラムのアーカイブ）](https://learn.microsoft.com/en-us/archive/msdn-technet-forums/28951ca9-50a7-4432-af61-fb4f2c207b82)
- [How to create a network share in an AAD only environment（Mike Terrill氏のブログ）](https://miketerrill.net/2019/07/04/how-to-create-a-network-share-in-an-aad-only-environment/)

## 対応（コマンドで解決）
コマンドプロンプト（またはPowerShell）で以下を実行します。`<秘密鍵ファイル>`は、対象のファイル名（`id_rsa`、`id_ed25519`など）に置き換えてください。

### 1. 自分のSIDを調べる
```
whoami /user
```
`S-1-12-1-`で始まる文字列（SID）をコピーします。

### 2. 所有者と権限を直す（1行ずつ実行）
`S-1-12-1-...`の部分を自分のSIDに置き換えます。先頭の`*`と末尾の`:F`は残してください。

コマンドプロンプトの場合：
```
icacls "%USERPROFILE%\.ssh\<秘密鍵ファイル>" /setowner *S-1-12-1-...
icacls "%USERPROFILE%\.ssh\<秘密鍵ファイル>" /inheritance:r
icacls "%USERPROFILE%\.ssh\<秘密鍵ファイル>" /grant:r *S-1-12-1-...:F
```

PowerShellの場合：
```powershell
icacls "$env:USERPROFILE\.ssh\<秘密鍵ファイル>" /setowner "*S-1-12-1-..."
icacls "$env:USERPROFILE\.ssh\<秘密鍵ファイル>" /inheritance:r
icacls "$env:USERPROFILE\.ssh\<秘密鍵ファイル>" /grant:r "*S-1-12-1-...:F"
```
※PowerShellでは、`*`や`:`を含む部分を`"`で囲んでおくと確実です。

| 行 | 内容 |
|---|---|
| 1行目 | 所有者を変更後のユーザーにします |
| 2行目 | フォルダーから受け継いだ権限を外します |
| 3行目 | 変更後のユーザーにフルコントロールを付けます |

いずれも`Successfully processed 1 files; Failed processing 0 files`と表示されれば成功です。

### 3. 確認

コマンドプロンプトの場合：
```
icacls "%USERPROFILE%\.ssh\<秘密鍵ファイル>"
```
PowerShellの場合：
```powershell
icacls "$env:USERPROFILE\.ssh\<秘密鍵ファイル>"
```
変更後のユーザーだけが表示されていれば完了です。

OKの例（`id_rsa`の場合。変更後のユーザーだけが表示されている）：
```
C:\Users\変更後のユーザー\.ssh\id_rsa AzureAD\変更後のユーザー:(F)

Successfully processed 1 files; Failed processing 0 files
```
- `AzureAD\変更後のユーザー`の部分に変更後のユーザー名が表示されます（SIDで指定しても、表示は名前になります）。
- `(F)`はフルコントロールの意味です。
- `NT AUTHORITY\SYSTEM`や`BUILTIN\Administrators`が一緒に表示されていても問題ありません。

NGの例（`id_rsa`の場合。他のユーザーが残っている）：
```
C:\Users\変更後のユーザー\.ssh\id_rsa AzureAD\変更後のユーザー:(F)
                                    PC名\元のユーザー:(F)
                                    BUILTIN\Users:(RX)

Successfully processed 1 files; Failed processing 0 files
```
- 元のユーザーや`Users`、`Everyone`、`Authenticated Users`が表示される場合は、表示された名前をそのまま指定して削除します。
  ```
  icacls "%USERPROFILE%\.ssh\<秘密鍵ファイル>" /remove "PC名\元のユーザー" "BUILTIN\Users"
  ```
  （PowerShellの場合は`%USERPROFILE%`を`$env:USERPROFILE`にしてください）

## 補足
- `config`内の`IdentityFile`に元のユーザーのパスが書かれていたら、変更後のユーザーのパスに書き換えてください。
- Azure ADではないローカルアカウントなどの場合は、SIDの代わりにユーザー名（`*`なし）を指定しても実行できます。コマンドプロンプトでは`%USERNAME%`、PowerShellでは`$env:USERNAME`で変更後のユーザー名を指定できます。