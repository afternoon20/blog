+++
date = '2026-01-03T12:18:34+09:00'
draft = false
title = '【PHP】ノンフレームワークでセキュアなコードを書く'
description = 'PHP標準関数やPDOを活用し、外部ライブラリに頼らずにSQLインジェクションやXSS、CSRFなどの主要な脆弱性を防ぐ実装方法を体系的に解説します。'
tags = ['PHP']
+++

PHPでフレームワークを使わずにWebアプリケーションを構築する場合、セキュリティ対策をすべて開発者自身が制御する必要があります。
本記事では、以下6つの対策について解説します。

1. XSS（クロスサイト・スクリプティング）
2. CSRF（クロスサイト・リクエスト・フォージェリ）
3. セッションハイジャック対策
4. SQLインジェクション
5. OSコマンドインジェクション
6. ディレクトリトラバーサル

## XSS対策：htmlspecialcharsによるサニタイズ

ユーザーの入力をブラウザに出力する際、HTMLタグやJavaScriptとして実行されるのを防ぎます。

```php
$username = $_POST['username'] ?? '';

echo "こんにちは、" . htmlspecialchars($username, ENT_QUOTES, 'UTF-8') . "さん";
```

htmlspecialchars関数は、特定の文字（<, >, &, ", '）をHTMLエンティティ（&lt;など）に変換します。引数のENT_QUOTESは、シングルクォートとダブルクォートの両方を変換対象にするための指定です。UTF-8の引数は文字コードを明示的に指定し、マルチバイト文字による誤作動を防ぎます。

## CSRF対策：ワンタイムトークンの発行と検証

利用者の意図しないリクエスト（設定変更や投稿など）を強制される攻撃を防ぐため、正当な画面からの遷移であることを確認します。

```php
session_start();

if ($_SERVER['REQUEST_METHOD'] === 'GET') {
    $token = bin2hex(random_bytes(32));
    $_SESSION['csrf_token'] = $token;
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $token = $_POST['csrf_token'] ?? '';
    if (!isset($_SESSION['csrf_token']) || !hash_equals($_SESSION['csrf_token'], $token)) {
        http_response_code(403);
        exit('Invalid CSRF token');
    }
}
```

暗号論的に安全なランダムトークンを生成し、```hash_equals```で比較します。

## セッションハイジャック対策
セッションIDが第三者に盗用されるリスクを低減するため、Cookieの設定とセッション管理を厳格に行います。

```php
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
ini_set('session.use_only_cookies', 1);

session_start();

session_regenerate_id(true);
```

1~3行目でそれぞれJavaScriptからのCookie読み取り禁止、HTTPS接続時のみCookieを送信許可、URLのパラメーターからのセッション受け渡しを禁止しています。
session_regenerate_id関数はログイン時などの重要なタイミングでセッションIDを更新し、セッション固定攻撃を防ぎます。

## SQLインジェクション対策：PDOプリペアドステートメントの利用

ユーザーが入力した文字が、意図せずSQLの命令（コマンド）の一部として実行されてしまう事を防ぎます。

```php
$dsn = 'mysql:host=localhost;dbname=testdb;charset=utf8mb4';
$pdo = new PDO($dsn, 'user', 'pass', [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_EMULATE_PREPARES => false,
]);

$stmt = $pdo->prepare("SELECT id, name FROM users WHERE email = :email");
$stmt->bindValue(':email', $_POST['email'], PDO::PARAM_STR);
$stmt->execute();
```

```prepare```と```bindValue```を使用し、値を命令から分離します。

## OSコマンドインジェクション

システムコマンドを実行する際、外部入力をコマンドの一部として扱わないようにします。

```php
$mode = $_GET['mode'] ?? 'default';
$allowed_commands = [
    'backup' => '/usr/bin/backup_script',
    'status' => '/usr/bin/status_check'
];

if (!isset($allowed_commands[$mode])) {
    exit('Invalid request');
}

$cmd = $allowed_commands[$mode];
exec($cmd, $output);
```

ユーザー入力を直接コマンドライン引数に渡さず、あらかじめ定義したホワイトリストから選択させます。

## ディレクトリトラバーサル

ファイル操作において、想定外のディレクトリへのアクセスを制限します。

```php
$base_dir = '/var/www/html/storage/';
$requested_file = $_GET['file'] ?? '';

$filename = basename($requested_file);
$target_path = $base_dir . $filename;

if (strpos(realpath($target_path), realpath($base_dir)) !== 0) {
    exit('Access denied');
}

if (file_exists($target_path)) {
    readfile($target_path);
}
```

```basename()```でディレクトリ要素を除去し、```realpath()```で最終的なパスが許可範囲内にあるかを確認します。

## まとめ

ノンフレームワークでの開発では、標準関数を適切に組み合わせてデータの安全性を確保する必要があります。具体的には、入力値が期待する形式に合致するかを検証し、不正な値は処理を行わずに遮断します。また、SQLやOSコマンドを実行する際は、プリペアドステートメント等を用いてデータと命令を物理的に分離し、入力値がプログラムのロジックとして解釈されるのを防ぎます。さらに、最終的な出力時には表示先の仕様に合わせて特殊文字をエスケープし、意図しないスクリプトの実行を防止する手続きを各工程で徹底することが不可欠です。

フレームワークの仕組みを理解するいい機会にもなります。