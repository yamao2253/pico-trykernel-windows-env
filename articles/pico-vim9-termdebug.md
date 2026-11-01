---
title: "Vim 9のtermdebugでRaspberry Pi Picoをデバッグする"
emoji: "🔧"
type: "tech"
topics: ["vim", "gdb", "openocd", "raspberrypi"]
published: false
---

# はじめに

前回の記事では、Windows 11環境でOpenOCDとGDBを使用し、Raspberry Pi Pico上で動作するTry Kernelをコマンドラインからデバッグしました。

今回は、Vim 9の標準機能である `termdebug` を使用して、Vim上からGDBを操作してみます。

`termdebug` を使うと、GDBのコマンドライン操作に加えて、ソースコード上でのブレークポイント設定やステップ実行、現在の実行位置の表示などが可能になります。

追加のVimプラグインを導入せず、実機でどこまでデバッグできるか確認します。

# 使用環境

今回使用した環境は次のとおりです。

| 項目 | 環境 |
|---|---|
| OS | Windows 11 |
| エディタ | GVim 9.0（64bit版） |
| デバッグ対象 | Raspberry Pi Pico W（RP2040） |
| デバッグプローブ | Pico Wを使用したCMSIS-DAP Debug Probe |
| GDB | xPack GNU Arm Embedded GDB 15.2.90 |
| OpenOCD | xPack OpenOCD 0.12.0 |
| 対象プログラム | Try Kernel（`part_5`） |

OpenOCDとGDBのインストール、デバッグプローブの準備、配線については、前回までの記事で紹介しています。

- [Windows 11でRaspberry Pi PicoのTry Kernel開発環境を構築する](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

今回は、これらの環境が構築済みであることを前提とします。

# Vimのtermdebugとは

`termdebug` はVim 9に標準で付属するプラグインで、追加のインストールは不要です。

`termdebug` はGDBを置き換えるものではなく、**GDBのデバッグ機能とVimのソースコード表示・操作機能を連携させる仕組み**です。

Vim内にGDBのターミナルを開くだけでなく、GDBの停止位置に合わせてソースコードを表示したり、ソースコード上からブレークポイントを操作したりできます。

デバッグ操作は、**GDBのターミナルからコマンドを入力する方法**と、**Vimのソースコード表示画面から `:Break` や `:Step` などのコマンドを実行する方法**の両方が使えます。

以降では、操作する場所を区別して説明します。

# コマンドラインのGDBとの違いは？

前回の記事では、GDBのコマンドラインからブレークポイントの設定やステップ実行を行いました。これだけでも実機デバッグは可能ですが、ソースコードを確認しながら操作するには、GDBとエディタを行き来する必要があります。

Vimの `termdebug` を使うと、GDBのデバッグ機能をそのまま利用しながら、ソースコードと連動した操作が可能になります。

| 操作 | コマンドラインのGDB | Vimのtermdebug |
|---|---|---|
| ソースコードの確認 | GDBのコマンドや別のエディタで確認 | 停止位置のソースを自動表示 |
| 実行位置の確認 | ファイル名と行番号で確認 | 該当行を強調表示 |
| ブレークポイント設定 | `break main`、`break ファイル名:行番号` | ソースコード上で `:Break` |
| ブレークポイント解除 | `delete 番号` | ソースコード上で `:Clear` |
| ステップ実行 | `step`、`next`、`finish` | ソースコード表示画面で `:Step`、`:Over`、`:Finish` |
| 変数の確認 | `print`、`display` | ソースコード表示画面で `:Evaluate`（継続的な監視にはGDBターミナルで `display` を実行） |

今回、特に便利だと感じたのは、ブレークポイントで停止すると、その場所のソースコードが自動的に表示されることです。

また、ブレークポイントを設定する際に、関数名や行番号をGDBへ入力する必要がなく、ソースコード上のブレークポイントを設定したい行にカーソルを置いて :Break を実行できます。

---

# termdebugを使うための事前設定

Vimの `termdebug` は標準プラグインですが、使用するには `packadd` による読み込みが必要です。

また、`termdebug` が使用するGDBを指定するため、変数 `g:termdebugger` に組込み開発用の `arm-none-eabi-gdb` を設定します。

GVimで次のコマンドを実行します。

```vim
:packadd termdebug
:let g:termdebugger = 'arm-none-eabi-gdb'
```

続いて、設定を確認します。

```vim
:echo exists(':Termdebug')
:echo g:termdebugger
```

>それぞれ 2、arm-none-eabi-gdb と表示されれば、設定は成功です。2 は、:Termdebug コマンドが利用可能であることを示しています。

## 設定の自動化

上記の設定はGVimを起動するたびに必要ですが、Vimの設定ファイルに記述しておけば自動的に設定されます。

Windowsの場合は、通常ホームディレクトリにある `_vimrc` に以下を追加します。

```vim
packadd termdebug
let g:termdebugger = 'arm-none-eabi-gdb'
```

なお、`_vimrc` に記述する場合は、先頭の `:` は不要です。

設定ファイルの場所は、GVimで `:echo $MYVIMRC` と入力すると確認できます。

---

# Try Kernelをデバッグする

ここからは、実際にRaspberry Pi Pico上で動作するプログラムを使って、`termdebug` によるデバッグを試してみます。

今回のデバッグ対象は、Interface誌の特集「ラズパイPicoで1500行 ゼロから作るOS」で紹介されている自作OS「Try Kernel」です。

Try Kernelの開発環境の構築方法や、OpenOCDとGDBを使ったコマンドラインでのデバッグ方法については、以下の記事で紹介しています。

- [Windows 11でRaspberry Pi PicoのTry Kernel開発環境を構築する](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [Raspberry Pi PicoをOpenOCDとGDBでデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

今回は、この環境を利用して、Vimのソースコード表示画面とGDBを連携させたデバッグ操作を確認します。

## OpenOCDを起動する

まず、PowerShellでOpenOCDを起動します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

ここでは、デバッグプローブ（CMSIS-DAP）とデバッグ対象（RP2040）の設定ファイルを指定しています。また、`adapter speed 1000` でSWDの通信速度を1000kHzに設定しています。

OpenOCDはGDBサーバとして動作し、GDBからの接続を待ち受けます。

OpenOCDを起動したPowerShellは、そのまま開いておきます。

## termdebugからGDBを起動する

GVimで次のコマンドを実行します。

```vim
:Termdebug
```

GVimの画面が分割され、上側にGDBターミナル、下側にソースコード表示画面が開きます。

<img src="/images/pico-vim9-termdebug/termdebug-start.png"
     alt="termdebug起動直後のGVim画面"
     width="700">

上側のGDBターミナルには `No symbol table is loaded. Use the "file" command.` と表示されています。これは、まだデバッグ対象のELFファイルを読み込んでいないためです。

次に、GDBターミナルからELFファイルを読み込み、OpenOCDへ接続します。

---

## ELFファイルを読み込む

ここからは、GVim内に開いた**GDBターミナル**で操作します。

`gdb>` プロンプトで、Try KernelのELFファイルを読み込みます。

```gdb
file C:/home/<ユーザ名>/repos/src/pico/interface_trykernel/part_5/build/trykernel.elf
```

実際のELFファイルの保存場所に合わせてパスを指定してください。

今回の環境では、次のようにシンボル情報が読み込まれました。

```text
Reading symbols from C:/home/.../part_5/build/trykernel.elf...
gdb>
```

## OpenOCDに接続する

引き続き、GDBターミナルでOpenOCDに接続します。

```gdb
target remote localhost:3333
```

接続後、RP2040をリセットして停止します。

```gdb
monitor reset halt
```

接続に成功すると、GVim側に停止位置のソースコードが表示されます。

今回の環境では、`kernel/dispatch.S` が自動的に開かれ、現在の実行位置が強調表示されました。

ここで、**GDBの実行位置とVimのソースコード表示が連動している**ことが確認できました。

## ソースコード上からデバッグする

ここまでの操作で、デバッグを開始する準備が整いました。

これ以降は、GDBターミナルだけでなく、**Vimのソースコード表示画面からもデバッグ操作ができます。**

例えば、ブレークポイントを設定したい行にカーソルを移動し、次のコマンドを実行します。

```vim
:Break
```

また、ステップ実行には次のコマンドが使えます。

```vim
:Step
```

これらの操作は、GDBターミナルではなく、**Vimのソースコード表示画面を選択した状態で実行します。**

具体的な操作方法は、次の節で説明します。

# ブレークポイントとステップ実行

## ブレークポイントを設定する

まず、GDBコンソールから `main` 関数にブレークポイントを設定します。

```gdb
break main
continue
```

`main` 関数で停止すると、GVimは自動的に `kernel/inittsk.c` を表示しました。

停止している行が強調表示され、左端にはブレークポイント番号が表示されます。

## ソースコード上からブレークポイントを操作する

`termdebug` では、GDBコンソールにコマンドを入力しなくても、ソースコード上からブレークポイントを操作できます。

設定したい行にカーソルを移動し、次を実行します。

```vim
:Break
```

解除する場合は、ブレークポイントが設定されている行にカーソルを移動し、次を実行します。

```vim
:Clear
```

今回の環境では、どちらも正常に動作しました。

## ステップ実行する

ソースコードのウィンドウで、次のVimコマンドを実行できます。

| 操作 | Vimコマンド | GDBコマンド |
|---|---|---|
| ステップイン | `:Step` | `step` |
| ステップオーバ | `:Over` | `next` |
| 関数から戻る | `:Finish` | `finish` |
| 実行再開 | `:Continue` | `continue` |

`:Step`、`:Over`、`:Finish` を実際に試したところ、ソースコードの表示が実行位置に追従しました。

GVimのメニューやツールバーからも、一部のデバッグ操作を実行できます。

# 変数の値を確認する

## Evaluateで値を確認する

ソースコード上で変数名にカーソルを置き、次のコマンドを実行します。

```vim
:Evaluate
```

GDBで評価した変数の値を確認できます。

## displayで変数を監視する

GDBには、プログラムが停止するたびに式の値を表示する `display` コマンドがあります。

これはVimのコマンドではなく、**GDBコンソールで実行するコマンド**です。

例えば、変数 `err` を監視する場合は次のように入力します。

```gdb
display err
```

登録済みの式を確認するには、次を実行します。

```gdb
info display
```

監視を解除するには、表示番号を指定します。

```gdb
undisplay 1
```

なお、GDBの `watch` は、変数の値を常時表示する機能ではなく、値が変更されたときにプログラムを停止させるウォッチポイントです。

GUIデバッガのWatchウィンドウとは異なるため、区別しておく必要があります。

# VimとGDBのコマンドを使い分ける

今回の操作で少し戸惑ったのが、VimのコマンドとGDBのコマンドの違いです。

例えば、次のように使い分けます。

| 操作場所 | コマンド例 |
|---|---|
| Vimのソースウィンドウ | `:Break`、`:Clear`、`:Step`、`:Over` |
| GDBコンソール | `break main`、`print err`、`display err`、`info registers` |

Vimのコマンドは `:` から入力し、GDBのコマンドは `gdb>` プロンプトに入力します。

`display` はVimにも同名のコマンドがあるため、`:display err` と入力してもGDBの変数監視にはなりません。

# おわりに

Vim 9の標準機能である `termdebug` を使用し、Windows 11上のGVimからRaspberry Pi Pico Wの実機デバッグを行えました。

今回確認できたのは、OpenOCDへの接続、ソースコードへの実行位置の追従、ブレークポイントの設定と解除、ステップ実行などです。

特に、追加のVimプラグインをインストールせずに、ソースコードと連動したデバッグができる点が印象的でした。

変数の継続監視などはGDBコンソールでの操作が中心になりますが、普段のVimから組込みプログラムをデバッグするための基本的な機能は揃っています。

今後は、Neovimでも同様のデバッグ環境を構築し、使い勝手の違いを確認してみたいと思います。
