# MusBatch3

MusBatch3 は、Windows 上のマウス操作とキーボード操作を記録し、`SndInput.exe` を呼び出すバッチファイルとして保存・再生するプログラムです。

ユーザー向けの説明は `GUI/doc/Readme.txt` にあります。この README は、プログラムを読む開発者向けの構成メモです。

## リポジトリ構成

```text
CUI/        記録・再生用の C 実装
GUI/        WinForms GUI 本体
Installer/  WinForms インストーラ
```

主なファイルは次の通りです。

| パス | 役割 |
| --- | --- |
| `CUI/src/RecInput.c` | マウスカーソル位置と仮想キー状態を記録し、中間形式の記録ファイルへ出力する CUI プログラム |
| `CUI/src/SndInput.c` | コマンドライン引数に従って `SendInput` でマウス・キーボード入力を送出する CUI プログラム |
| `GUI/HLTForm/HLTForm/Program.cs` | GUI 本体のエントリポイント。多重起動抑止、設定読み込み、メインフォーム起動を行う |
| `GUI/HLTForm/HLTForm/MainWin.cs` | 記録開始/停止、再生開始/中断、繰り返し再生など GUI の主要操作 |
| `GUI/HLTForm/HLTForm/BatchConv.cs` | `RecInput.exe` の記録ファイルを `SndInput.exe` 呼び出しの `.bat` に変換する処理 |
| `GUI/HLTForm/HLTForm/Ground.cs` | GUI の永続設定、実行中プロセス、状態管理 |
| `GUI/doc/Readme.txt` | 配布物に含まれるユーザー向け Readme |
| `GUI/doc/WRec.conf` | GUI の追加設定ファイルのサンプル/配布用ファイル |
| `Installer/HLTForm/HLTForm/MainWin.cs` | インストール/アンインストール処理 |
| `Installer/HLTForm/HLTForm/Consts.cs` | インストーラが扱うアプリ名、クラスタファイル名、既インストール判定ファイル名など |
| `MakeInstaller.bat` | CUI/GUI/Installer のリリース成果物をまとめて `C:\temp\Release\MusBatch` を作成するスクリプト |
| `MakeDistribution.bat` | `C:\Dev\Factory\bat\Dev\MakeDistribution.bat` を呼び出すラッパー |

## 実行時の処理概要

GUI 本体は `Program.Main` から起動します。起動時に `BootTools.OnBoot()` が実行ファイルのあるディレクトリをカレントディレクトリに設定し、`APP_IDENT` と `APP_TITLE` を使って多重起動を抑止します。

記録開始時、`MainWin.Btn記録開始_Click` は `RecInput.exe` を `/R` 付きで起動します。通常の配置では `..\CUI\RecInput.exe` を探し、見つからない場合は開発時の相対パス `..\..\..\..\..\CUI\src\RecInput.exe` を使います。

記録停止時、GUI は `Consts.REC_INPUT_STOP_EVENT` と同じ GUID の名前付きイベントをセットして `RecInput.exe` に停止を伝えます。停止後、`BatchConv.Perform()` が記録ファイルを読み取り、保存先として選択された `.bat` に変換します。

再生時、GUI は選択された `.bat` をプロセスとして実行します。設定によっては右 Control キー押下で再生プロセスを強制終了します。

## CUI プログラム

### RecInput.exe

`CUI/src/RecInput.c` は、10 ミリ秒ごとに次の状態を取得します。

- `GetCursorPos` によるカーソル座標
- `GetAsyncKeyState` による 0x00 から 0xff までの仮想キー状態

出力される中間形式は、行ごとに次のトークンを持ちます。

| トークン | 意味 |
| --- | --- |
| `M X Y` | マウスカーソル座標 |
| `D VK` | 仮想キー `VK` の押下 |
| `U VK` | 仮想キー `VK` の解放 |

`RecInput.exe /R REC-FILE R-CTRL-AND-END` で記録を開始します。`R-CTRL-AND-END` が 0 以外の場合、右 Control キー押下でも記録を終了します。

ソース冒頭のコメントには `/R REC-FILE SAMPLING-MILLIS R-CTRL-AND-END` とありますが、現在の実装では `SamplingMillis` は読み取られておらず、GUI からも `/R "記録ファイル" 0|1` の形で起動されています。

`RecInput.exe /S` は停止イベントをセットします。ただし GUI 本体は `/S` を使わず、同じ名前付きイベントを C# 側からセットしています。

### SndInput.exe

`CUI/src/SndInput.c` は Windows API の `SendInput` を使います。複数コマンドを 1 回の起動引数に連続して指定できます。

| コマンド | 意味 |
| --- | --- |
| `MC X Y` | マウスカーソルを画面座標 `(X, Y)` へ移動 |
| `MB Btn Dwn` | マウスボタン操作。`Btn`: 1=左, 2=中央, 3=右。`Dwn`: 1=押下, 0=解放 |
| `MW Lvl` | マウスホイール操作。負数は手前方向、正数は奥方向 |
| `KB Vk Dwn` | 仮想キー操作。`Dwn`: 1=押下, 0=解放 |
| `T Millis` | 指定ミリ秒待機 |
| `RR X-RndRng Y-RndRng` | 以降の `MC` 座標に指定範囲内のランダム誤差を加える |

## バッチ変換

`BatchConv` は `RecInput.exe` の中間形式を読み込み、次の順で変換します。

1. 必要に応じて Shift/Control/Alt の仮想キー 16, 17, 18 を除外する。
2. GUI 設定に従ってキーボード操作またはマウス操作を除外する。
3. 停止ボタン押下由来とみなされる終端付近の左クリックを除外する。
4. 連続する待機時間を結合する。
5. 短い押下/解放をクリック、ダブルクリック、キーストロークとしてまとめる。
6. 中間コマンドを `SndInput.exe` の `MC` / `MB` / `KB` / `T` コマンド列へ変換する。
7. 設定により複数操作を 1 行にまとめる。

出力 `.bat` は Shift_JIS で保存されます。`SndInput.exe` の呼び出しは、設定 `SndInputBatLocal` が有効ならローカル名、無効なら GUI から見た `..\CUI\SndInput.exe` のフルパスになります。

## GUI の設定ファイル

GUI 本体は実行ファイルと同じディレクトリを基準に、次のファイルを読み書きします。

| ファイル | 文字コード | 用途 |
| --- | --- | --- |
| `GUI.conf` | Shift_JIS | 手動編集向けの追加設定。存在しない、または読み込み失敗時は既定値を使う |
| `GUI.dat` | UTF-8 | ウィンドウ位置、記録/再生設定、最後の出力ディレクトリなどの永続データ |
| `HLTForm.log` | 実装上は `Utils.WriteLog` 経由 | 例外や一部の状態ログ |

`GUI/doc/WRec.conf` は配布用の `GUI.conf` として読める内容です。`Ground.LoadConf()` は空行と `;` で始まるコメント行を除外した上で、先頭の有効項目数と実際の行数が一致することを確認します。

## インストーラ

インストーラは .NET Framework 4.8 の WinForms アプリです。`Installer/HLTForm/HLTForm/Consts.cs` によると、インストール時に扱うクラスタファイルは次の 2 つです。

- `GUI.cmp-gz`
- `CUI.cmp-gz`

`MainWin.DoInstallWkTh()` は、各クラスタファイルと対応する `.hash` ファイルの存在を確認し、SHA-512 ハッシュが一致することを検証してから展開します。既定のインストール先は `%LOCALAPPDATA%\MusBatch` です。ショートカット作成が有効な場合は、デスクトップに `MusBatch.lnk` を作成します。

アンインストール処理は、指定されたインストール先ディレクトリを一度移動できるか確認してから削除します。ショートカット削除は確認ダイアログで選択されます。

## ビルドとリリース

確認できるプロジェクト設定は次の通りです。

- `GUI/HLTForm/HLTForm/HLTForm.csproj`: .NET Framework 4.8、WinExe、x86
- `Installer/HLTForm/HLTForm/HLTForm.csproj`: .NET Framework 4.8、WinExe、x86、`IWshRuntimeLibrary` COM 参照あり
- C ソースは `C:\Dev\Factory\Common\all.h` を直接 include している

各 `Release.bat` は `C:\Dev\Factory` 配下のバッチやツールに依存します。

| スクリプト | 内容 |
| --- | --- |
| `CUI/Release.bat` | `C:\Dev\Factory\SetEnv.bat` を呼び、`Cx **` で C ソースをビルドした後、`src\*.exe` を `out\CUI` へコピーする |
| `GUI/Release.bat` | `C:\Dev\Factory\bat\Dev\Release\CommonReleaseProject.bat` を呼ぶ |
| `Installer/Release.bat` | `C:\Dev\Factory\bat\Dev\Release\CommonReleaseProject.bat` を呼ぶ |
| `MakeInstaller.bat` | `runsub Release` 後、`CUI/out/CUI` と `GUI/out/GUI` を圧縮し、SHA-512 ハッシュを作成し、`Installer/out/Installer` と合わせて `C:\temp\Release\MusBatch` を作る |
| `MakeDistribution.bat` | `C:\Dev\Factory\bat\Dev\MakeDistribution.bat` を呼ぶ |

`MakeInstaller.bat` は `C:\apps\Compress\Compress.exe`、`sha512`、`SFCP`、`z7`、`RDMD`、`runsub` などの外部コマンドに依存します。

## 開発時の注意

- C ソースと `.txt` / `.bat` は `.gitattributes` で CP932 + CRLF に固定されています。
- C# ソースは UTF-8 + CRLF に固定されています。
- `.gitignore` では `bin/`、`obj/`、`.vs/`、`out/`、`tmp/`、`*.exe`、`*.obj` が除外対象です。
- GUI は実行パスに Shift_JIS へ変換できない文字が含まれる場合、およびネットワークフォルダから実行された場合に終了します。
- GUI の再生対象 `.bat` も、ネットワークパスや Shift_JIS で表現できないパスは拒否されます。
- 記録上限は `RecInput.c` の `REC_MAX` が 360000、サンプリング間隔が 10 ミリ秒なので、実装上は約 1 時間です。
- `CUI/src/sub/RecMon.c` と `CUI/src/sub/WheelMon.c` は補助/調査用のプログラムです。`WheelMon.c` には、ホイール操作の検出に使えなかった旨のコメントがあります。

## 調査時点で確認しなかったこと

以下はこの README では断定していません。

- `C:\Dev\Factory` 配下の共通ビルドスクリプトや `CommonReleaseProject.bat` の内部動作
- リリース工程での最終的な GUI 実行ファイル名のリネーム規則
- `CUI/src` 配下の `*.exe` がどの C ソースからどの名前で生成されるかの全ルール
- 実機でのビルド可否とインストーラ実行結果
