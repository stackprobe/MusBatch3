============
MusBatch CUI
============


----
概要

このフォルダには、マウス・キーボード操作の記録と再生に使用する
CUI プログラムが含まれています。

	RecInput.exe
	SndInput.exe

RecInput.exe は、マウスカーソル位置とキーボード・マウスボタンの状態を
一定間隔で記録し、中間形式の記録ファイルへ出力します。

SndInput.exe は、コマンドライン引数に従って Windows API の SendInput を呼び出し、
マウス・キーボード入力を送出します。

通常は GUI プログラムから呼び出されますが、コマンドプロンプトや
バッチファイルから直接実行することもできます。


----
動作環境

Windows


----
収録ファイル

	RecInput.exe	入力操作を記録します。
	SndInput.exe	入力操作を再生します。
	Readme.txt	このファイルです。


----
RecInput.exe

入力操作を記録し、指定されたファイルへ保存します。

	RecInput.exe /R REC-FILE R-CTRL-AND-END

		REC-FILE       ... 記録ファイル
		R-CTRL-AND-END ... 右 Control キー押下で記録を停止するか
		                   0 = 停止しない
		                   0 以外 = 停止する

例)

	RecInput.exe /R rec.tmp 1

上記の例では rec.tmp に入力操作を記録します。
記録中に右 Control キーを押すと記録を終了します。


記録を停止するには、別のコマンドプロンプトなどから次のように実行します。

	RecInput.exe /S

/S は停止イベントを通知します。
GUI プログラムから記録している場合は、GUI 側から同じ停止イベントが通知されます。


記録ファイルは、GUI プログラムがバッチファイルへ変換するための中間形式です。
主なトークンは次の通りです。

	M X Y		マウスカーソル位置を記録します。
	D VK		仮想キー VK が押されたことを記録します。
	U VK		仮想キー VK が放されたことを記録します。

例)

	M 123 456
	 D 1
	 U 1


----
SndInput.exe

指定されたコマンドを順に実行し、マウス・キーボード入力を送出します。

	SndInput.exe COMMAND ...

複数のコマンドを 1 回の起動引数に続けて指定できます。

例)

	SndInput.exe KB 32 1 T 30 KB 32 0

上記の例では、スペースキーを押し、30 ミリ秒待ってから放します。


----
SndInput.exe のコマンド

	SndInput.exe MC X-Pos Y-Pos

		マウスカーソルを移動します。

		X-Pos ... X 座標
		Y-Pos ... Y 座標

		例) SndInput.exe MC 123 456
		    マウスカーソルを (123,456) に移動します。


	SndInput.exe MB Btn Dwn

		マウスボタンを操作します。

		Btn ... 1 = 左ボタン, 2 = 中央ボタン, 3 = 右ボタン
		Dwn ... 1 = 押す, 0 = 放す

		例) SndInput.exe MB 1 1
		    左ボタンを押します。


	SndInput.exe MW Lvl

		マウスホイールを操作します。

		Lvl ... 回転量を整数で指定します。
		        マイナスは手前方向、プラスは奥方向です。

		例) SndInput.exe MW -3
		    マウスホイールを手前方向に 3 単位回します。

		例) SndInput.exe MW 5
		    マウスホイールを奥方向に 5 単位回します。


	SndInput.exe KB Vk Dwn

		キーボードを操作します。

		Vk  ... 仮想キーコード
		Dwn ... 1 = 押す, 0 = 放す

		例) SndInput.exe KB 32 1
		    スペースキーを押します。


	SndInput.exe T Millis

		指定時間ウェイトします。

		Millis ... ミリ秒

		例) SndInput.exe T 100
		    100 ミリ秒ウェイトします。


	SndInput.exe RR X-RndRng Y-RndRng

		以降の MC コマンドの座標にランダムな誤差を加えます。
		誤差は、その都度 -最大誤差 から +最大誤差 までの範囲で生成されます。

		X-RndRng ... X 座標の最大誤差
		Y-RndRng ... Y 座標の最大誤差

		例) SndInput.exe RR 10 5 MC 400 300
		    マウスカーソルを (390〜410,295〜305) のどこかに移動します。


----
複数コマンドの指定

SndInput.exe のコマンドは複数指定できます。

例えば、次の 3 行は、

	SndInput.exe KB 32 1
	SndInput.exe T 30
	SndInput.exe KB 32 0

次のように 1 行にまとめられます。

	SndInput.exe KB 32 1 T 30 KB 32 0


----
注意

SndInput.exe を実行すると、実際にマウス・キーボード入力が送出されます。
実行前に、操作内容と実行対象のウィンドウを確認してください。

再生中はマウス・キーボード操作が難しくなり、バッチファイルを中断しにくい
状況になる場合があります。
GUI プログラムから再生する場合は、「右 Control 押下でも中断する」を
有効にしておくことをお勧めします。

RecInput.exe の記録間隔は 10 ミリ秒です。
記録件数の上限に達するとエラーになり、記録中の操作データは保存されません。

RecInput.exe の記録ファイルは中間形式です。
通常の再生には、GUI プログラムで変換したバッチファイルを使用してください。


----
補足

仮想キーコードは Windows の Virtual-Key Codes に従います。

マウスボタンは、記録時の仮想キーコードと再生時の Btn 指定が異なります。
SndInput.exe の MB コマンドでは、1 が左ボタン、2 が中央ボタン、
3 が右ボタンです。

