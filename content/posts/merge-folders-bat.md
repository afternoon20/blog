+++
date = '2026-09-02T09:12:44+09:00'
draft =  false
title = 'フォルダの差分を比較・出力するbatファイルの作成と使い方'
description = '2つのフォルダを比較して、片方にしかないファイルと内容が違うファイルを一覧表示するバッチファイルmerge_folders.batを作成したので、その解説と使い方をまとめました。'
tags = ['その他']
+++

git diffのように「どちらかにしかないファイル」と「両方にあるが中身が違うファイル」を一覧表示してくれるツールが欲しかったので、バッチファイルで作りました。
Windows標準の機能だけで、2つのフォルダの中身を比較して差分を確認したいという限られた環境で使うことを想定しています。

## できること
 
- 2つのフォルダをサブフォルダ含めて再帰的に比較
- Aにしかないファイルを`Only in A: パス`として表示
- Bにしかないファイルを`Only in B: パス`として表示
- 両方にあるテキストファイルは`fc /n`で行単位の差分を表示
- 画像・exe・zipなどのバイナリファイルは内容比較せず、存在チェックのみ
## 使い方
 
```
merge_folders A B
```
 
コンソールに直接結果が表示されます。ファイルに保存したい場合はリダイレクトを使います。
 
```
merge_folders A B > result.txt
```
 
## 出力例
 
例えば以下のような構成のフォルダを比較したとします。
 
```
A/
├── common.txt      (中身: hello)
├── onlyA.txt
└── sub/
    └── nested.md   (中身: sub content A)
 
B/
├── common.txt      (中身: world)
├── onlyB.txt
└── sub/
    └── nested.md   (中身: sub content B)
```
 
`merge_folders A B` を実行すると、次のように出力されます。
 
```
==============================================
 Diff : "C:\path\to\A"  vs  "C:\path\to\B"
==============================================
 
---------- Only in A / Only in B ----------
Only in A: onlyA.txt
Only in B: onlyB.txt
 
---------- Content differences (text files only) ----------
 
=== diff: common.txt ===
***** C:\path\to\A\common.txt
1:  hello
***** C:\path\to\B\common.txt
1:  world
*****
 
=== diff: sub\nested.md ===
***** C:\path\to\A\sub\nested.md
1:  sub content A
***** C:\path\to\B\sub\nested.md
1:  sub content B
*****
```
 
`onlyA.txt`・`onlyB.txt`は片方にしかないので存在のみ報告され、`common.txt`・`sub\nested.md`は両方にあるが中身が違うので`fc /n`による差分が表示されます。中身が完全に同じファイルは何も出力されません。
 
## コード全文
 
```bat
@echo off
rem ============================================================
rem  merge_folders.bat
rem  2つのフォルダを比較し、git diff風に差分を出力する
rem
rem  使い方:
rem    merge_folders A B
rem    merge_folders A B > result.txt
rem ============================================================
setlocal enabledelayedexpansion
 
if "%~1"=="" goto usage
if "%~2"=="" goto usage
 
set "FOLDER_A=%~f1"
set "FOLDER_B=%~f2"
 
if not exist "%FOLDER_A%\" (
    echo [ERROR] FolderA が見つかりません: "%FOLDER_A%" 1>&2
    exit /b 1
)
if not exist "%FOLDER_B%\" (
    echo [ERROR] FolderB が見つかりません: "%FOLDER_B%" 1>&2
    exit /b 1
)
 
rem --- 内容比較(fc)の対象とするテキスト系拡張子 ---
rem     ここに無い拡張子は「存在チェックのみ」(Only in A/B判定のみ)になる
set "TEXT_EXTS= .txt .md .log .csv .tsv .ini .cfg .conf .json .xml .html .htm .css .js .ts .jsx .tsx .py .java .c .cpp .h .hpp .cs .go .rb .php .sql .yaml .yml .ps1 .bat .cmd .sh .properties .srt .gitignore .gitattributes .editorconfig "
 
set "TMP_DIR=%TEMP%\merge_folders_%RANDOM%_%RANDOM%"
mkdir "%TMP_DIR%" >nul 2>&1
 
set "LIST_A=%TMP_DIR%\listA.txt"
set "LIST_B=%TMP_DIR%\listB.txt"
 
type nul > "%LIST_A%"
type nul > "%LIST_B%"
 
rem --- Aフォルダ内の相対パス一覧を作成 ---
pushd "%FOLDER_A%"
for /r %%F in (*) do (
    set "full=%%F"
    set "rel=!full:%FOLDER_A%\=!"
    echo(!rel!>>"%LIST_A%"
)
popd
 
rem --- Bフォルダ内の相対パス一覧を作成 ---
pushd "%FOLDER_B%"
for /r %%F in (*) do (
    set "full=%%F"
    set "rel=!full:%FOLDER_B%\=!"
    echo(!rel!>>"%LIST_B%"
)
popd
 
sort "%LIST_A%" /o "%LIST_A%" >nul 2>&1
sort "%LIST_B%" /o "%LIST_B%" >nul 2>&1
 
echo ==============================================
echo  Diff : "%FOLDER_A%"  vs  "%FOLDER_B%"
echo ==============================================
echo.
echo ---------- Only in A / Only in B ----------
 
rem --- Aにしかないファイル ---
for /f "usebackq delims=" %%L in ("%LIST_A%") do (
    findstr /x /c:"%%L" "%LIST_B%" >nul 2>&1
    if errorlevel 1 (
        echo Only in A: %%L
    )
)
 
rem --- Bにしかないファイル ---
for /f "usebackq delims=" %%L in ("%LIST_B%") do (
    findstr /x /c:"%%L" "%LIST_A%" >nul 2>&1
    if errorlevel 1 (
        echo Only in B: %%L
    )
)
 
echo.
echo ---------- Content differences (text files only) ----------
 
rem --- 両方に存在するテキストファイルのみ内容比較 ---
rem     バイナリ(拡張子がTEXT_EXTSに無いもの)は存在チェックのみで内容比較はスキップ
for /f "usebackq delims=" %%L in ("%LIST_A%") do (
    findstr /x /c:"%%L" "%LIST_B%" >nul 2>&1
    if not errorlevel 1 (
        set "ext=%%~xL"
        echo(!TEXT_EXTS!| findstr /i /c:" !ext! " >nul 2>&1
        if not errorlevel 1 (
            fc /n "%FOLDER_A%\%%L" "%FOLDER_B%\%%L" >nul 2>&1
            if errorlevel 1 (
                echo.
                echo === diff: %%L ===
                fc /n "%FOLDER_A%\%%L" "%FOLDER_B%\%%L" 2>nul
            )
        )
    )
)
 
rmdir /s /q "%TMP_DIR%" >nul 2>&1
endlocal
exit /b 0
 
:usage
echo Usage: merge_folders ^<FolderA^> ^<FolderB^> [ ^> result.txt ]
exit /b 1
```
 
## コード解説
 
### 1. 引数チェックとフォルダの存在確認
 
```bat
if "%~1"=="" goto usage
if "%~2"=="" goto usage
 
set "FOLDER_A=%~f1"
set "FOLDER_B=%~f2"
```
 
`%~f1`で相対パス指定でも絶対パスに変換しています。指定フォルダが存在しない場合はエラーメッセージを標準エラー出力（`1>&2`）に出して終了します。標準エラーに出すことで、`> result.txt`で結果をファイルに落としてもエラー内容だけはコンソールに表示されます。
 
### 2. 内容比較の対象拡張子を定義
 
```bat
set "TEXT_EXTS= .txt .md .log .csv ... .editorconfig "
```
 
テキストファイルとみなす拡張子をスペース区切りで一覧管理しています。前後にスペースを入れているのは、後述の`findstr`で部分一致による誤判定（例:`.js`が`.jsx`にヒットしてしまう等）を防ぐためです。ここに無い拡張子（画像・exe・zipなど）は自動的に内容比較なし・存在チェックのみの扱いになります。
 
### 3. 両フォルダのファイル一覧を作成
 
```bat
pushd "%FOLDER_A%"
for /r %%F in (*) do (
    set "full=%%F"
    set "rel=!full:%FOLDER_A%\=!"
    echo(!rel!>>"%LIST_A%"
)
popd
```
 
`for /r`でサブフォルダを含めた全ファイルを再帰的に取得し、`%FOLDER_A%`のプレフィックスを取り除いて相対パスに変換、一時ファイルに書き出します。同じ処理をBフォルダにも行い、最後に`sort`で両リストを整列させます（整列しておくことで後の突き合わせが安定します）。
 
### 4. 片方にしかないファイルを判定
 
```bat
for /f "usebackq delims=" %%L in ("%LIST_A%") do (
    findstr /x /c:"%%L" "%LIST_B%" >nul 2>&1
    if errorlevel 1 (
        echo Only in A: %%L
    )
)
```
 
Aの一覧を1行ずつ読み、`findstr /x`（完全一致）でBの一覧に同じ行があるか検索します。見つからなければ`errorlevel 1`になるので「Only in A」として出力します。Bについても同様の処理を行います。
 
### 5. 共通ファイルのうちテキストファイルだけ内容比較
 
```bat
set "ext=%%~xL"
echo(!TEXT_EXTS!| findstr /i /c:" !ext! " >nul 2>&1
if not errorlevel 1 (
    fc /n "%FOLDER_A%\%%L" "%FOLDER_B%\%%L" >nul 2>&1
    if errorlevel 1 (
        echo === diff: %%L ===
        fc /n "%FOLDER_A%\%%L" "%FOLDER_B%\%%L" 2>nul
    )
)
```
 
`%%~xL`でファイルの拡張子だけを取り出し、`TEXT_EXTS`に含まれているか判定します。含まれていれば`fc /n`（行番号付き比較）を実行し、差分があれば`=== diff: ファイル名 ===`の見出しとともに差分内容を表示します。拡張子がリストに無い場合（バイナリ扱い）は、この内容比較自体をスキップします。
 
## Macで実行する場合
 
Macの場合は`.bat`は使えないので、代わりに`merge_folders.sh`という名前で以下の内容のファイルを作成し、同じように実行します。
 
```bash
#!/bin/bash
# ============================================================
# merge_folders.sh
# 2つのフォルダを比較し、git diff風に差分を出力する (macOS / Linux)
#
# 使い方:
#   ./merge_folders.sh A B
#   ./merge_folders.sh A B > result.txt
# ============================================================
set -u
 
# 内容比較(diff)の対象とするテキスト系拡張子
# ここに無い拡張子は「存在チェックのみ」(Only in A/B判定のみ)になる
TEXT_EXTS=" txt md log csv tsv ini cfg conf json xml html htm css js ts jsx tsx py java c cpp h hpp cs go rb php sql yaml yml ps1 bat cmd sh properties srt gitignore gitattributes editorconfig "
 
if [ $# -ne 2 ]; then
    echo "Usage: $0 <FolderA> <FolderB> [ > result.txt ]" >&2
    exit 1
fi
 
FOLDER_A="$(cd "$1" 2>/dev/null && pwd)"
FOLDER_B="$(cd "$2" 2>/dev/null && pwd)"
 
if [ -z "$FOLDER_A" ]; then
    echo "[ERROR] FolderA が見つかりません: $1" >&2
    exit 1
fi
if [ -z "$FOLDER_B" ]; then
    echo "[ERROR] FolderB が見つかりません: $2" >&2
    exit 1
fi
 
TMP_DIR="$(mktemp -d)"
trap 'rm -rf "$TMP_DIR"' EXIT
 
LIST_A="$TMP_DIR/listA.txt"
LIST_B="$TMP_DIR/listB.txt"
 
# 相対パス一覧を作成(サブフォルダ含めて再帰的に)
( cd "$FOLDER_A" && find . -type f | sed 's|^\./||' | sort ) > "$LIST_A"
( cd "$FOLDER_B" && find . -type f | sed 's|^\./||' | sort ) > "$LIST_B"
 
echo "=============================================="
echo " Diff : \"$FOLDER_A\"  vs  \"$FOLDER_B\""
echo "=============================================="
echo
echo "---------- Only in A / Only in B ----------"
 
# Aにしかないファイル / Bにしかないファイル
comm -23 "$LIST_A" "$LIST_B" | while IFS= read -r rel; do
    echo "Only in A: $rel"
done
comm -13 "$LIST_A" "$LIST_B" | while IFS= read -r rel; do
    echo "Only in B: $rel"
done
 
echo
echo "---------- Content differences (text files only) ----------"
 
# 拡張子がテキスト系か判定する関数
is_text_file() {
    local rel="$1"
    local ext="${rel##*.}"
    # 拡張子が無い場合(パスと同じ=分割できない)はバイナリ扱い
    if [ "$ext" = "$rel" ]; then
        return 1
    fi
    ext="$(echo "$ext" | tr '[:upper:]' '[:lower:]')"
    case "$TEXT_EXTS" in
        *" $ext "*) return 0 ;;
        *) return 1 ;;
    esac
}
 
# 両方に存在するファイルのみ内容比較
comm -12 "$LIST_A" "$LIST_B" | while IFS= read -r rel; do
    if is_text_file "$rel"; then
        if ! diff -q "$FOLDER_A/$rel" "$FOLDER_B/$rel" >/dev/null 2>&1; then
            echo
            echo "=== diff: $rel ==="
            diff -u --label "A/$rel" --label "B/$rel" "$FOLDER_A/$rel" "$FOLDER_B/$rel"
        fi
    fi
done
 
exit 0
```
 
```
./merge_folders.sh A B
./merge_folders.sh A B > result.txt
```
 
`for /r`や`findstr`の代わりに`find`・`comm`・`diff`を使っているだけで、「片方にしかないファイルを`Only in A/B`で表示」「テキスト拡張子のみ内容比較（`diff -u`によるgit diff風の出力）」「バイナリは存在チェックのみ」というロジックはbat版と同じです。
 
## 注意事項
 
- ファイル名に`%`や`&`、`^`などの特殊文字が含まれていると、正しく処理できないことがあります
- テキストファイルの内容比較は`fc`コマンドの仕様に依存するため、文字コードが異なるファイル同士では差分表示が崩れることがあります
- 対象ファイルが数千を超えると、`findstr`による突き合わせ処理がやや遅くなります
- Mac版（`merge_folders.sh`）は、権限がないため実行できない・「Permission denied」と表示される場合があります。その際は`chmod +x merge_folders.sh`で実行権限を付与してください
Windows標準コマンドだけで完結するので、追加のツールをインストールしたくない環境や、簡易的な差分確認をサクッと行いたい場面で使えると思います。
 