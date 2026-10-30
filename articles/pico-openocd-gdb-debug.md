---
title: "OpenOCDとGDBでRaspberry Pi Picoをデバッグする"
emoji: "🔧"
type: "tech"
topics: ["raspberrypi", "pico", "openocd", "gdb"]
published: false
---

# はじめに

『Interface 2023年7月号』の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、OpenOCDとGDBを使った実機デバッグを行っています。

OpenOCDとGDBは組込み開発で便利なツールですが、しばらく使わないと細かいコマンドや操作手順を忘れてしまい、使うたびに調べ直すことがあります。

そこで、両者の役割を整理するとともに、基本的なデバッグ操作を備忘録としてまとめました。

本記事では、Windows 11上のPowerShellを使用し、Raspberry Pi Pico W（RP2040）をコマンドラインからデバッグする方法を紹介します。

開発環境の構築方法については、以下の記事を参照してください。

[「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)

本記事では環境構築の説明を省略し、OpenOCDとGDBの役割や、実際のデバッグ操作に焦点を当てます。

# OpenOCDとGDBの役割

OpenOCDとGDBは、どちらもデバッグに使用するツールですが、役割が異なります。

- **GDB（GNU Debugger）**：ブレークポイントの設定、ステップ実行、変数やレジスタの参照など、デバッグ操作を行う。
- **OpenOCD（Open On-Chip Debugger）**：GDBからの指示を受け、デバッグプローブを介してマイコンを制御する。

今回のデバッグ環境は、次の構成です。

```text
Windows 11
┌─────────────────────────────────────────┐
│                                         │
│  GDB                                    │
│   │                                     │
│   │ GDB Remote Serial Protocol          │
│   │ (TCP/IP localhost:3333)             │
│   ▼                                     │
│  OpenOCD                                │
│   │                                     │
└───┼─────────────────────────────────────┘
    │ USB（CMSIS-DAP）
    ▼
 Debug Probe（Pico W）
    │
    │ SWD
    ▼
 Raspberry Pi Pico W（RP2040）
```

GDBとOpenOCDは、Windows上でそれぞれ独立したプロセスとして動作します。

GDBはTCP/IPでOpenOCDのGDBサーバに接続します。OpenOCDはUSB経由でDebug Probeと通信し、SWD（Serial Wire Debug）を使ってRP2040にアクセスします。

つまり、**GDBはデバッグ操作を行うためのインタフェース、OpenOCDはマイコンとの通信を仲介する役割**です。

VSCodeなどのGUIからデバッグする場合も基本的な仕組みは同じですが、本記事ではそれぞれの役割を理解するため、OpenOCDとGDBを個別に起動します。

---

# OpenOCDの起動

PowerShellを開き、次のコマンドを実行します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

各オプションの意味は次のとおりです。

| 指定 | 内容 |
|---|---|
| `-f interface/cmsis-dap.cfg` | デバッグプローブのインタフェースをCMSIS-DAPに指定 |
| `-c "adapter speed 1000"` | SWDの通信速度を1000 kHz（1 MHz）に指定 |
| `-f target/rp2040.cfg` | デバッグ対象をRP2040に指定 |

## 設定ファイル（cfg）の役割

OpenOCDには、さまざまなデバッグプローブやマイコンに対応する設定ファイル（`.cfg`）が標準で用意されています。

今回 `-f` オプションで指定している2つの設定ファイルも、OpenOCDに付属するもので、自分で作成する必要はありません。

- `interface/cmsis-dap.cfg`：CMSIS-DAP対応のデバッグプローブを使用するための設定ファイル
- `target/rp2040.cfg`：RP2040をデバッグするための設定ファイル

OpenOCDでは、デバッグプローブ側の設定と、デバッグ対象のマイコン側の設定を分けて管理しています。

そのため、デバッグプローブやマイコンを変更する場合も、対応する設定ファイルを選択することで構成を切り替えられます。

### 設定ファイルのカスタマイズ

OpenOCDに付属する設定ファイルは、通常、そのまま使用します。

通信速度などの調整は、`-c` オプションで指定できます。今回の `-c "adapter speed 1000"` もその一例です。

ボード固有の設定や、複数の設定をまとめたい場合は、独自の `.cfg` ファイルを作成することもできます。

例えば、次のような `pico-debug.cfg` を用意します。

```tcl
source [find interface/cmsis-dap.cfg]
adapter speed 1000
source [find target/rp2040.cfg]
```

このファイルをカレントディレクトリに保存すれば、PowerShellから次のように起動できます。

```powershell
openocd -f .\pico-debug.cfg
```

標準の設定ファイルを直接変更せず、必要な設定だけを追加する方法です。

今回のTry Kernelでは、独自の設定ファイルを作る必要はなく、最初に示した起動コマンドを使用しています。

## 設定ファイルの保存先を確認する

今回使用しているxPack OpenOCD 0.12.0-7.1では、実行ファイルと設定ファイルは次の場所に配置されています。

```text
.content/
├── bin/
│   └── openocd.exe
└── openocd/
    └── scripts/
        ├── interface/
        │   └── cmsis-dap.cfg
        └── target/
            └── rp2040.cfg
```

実行ファイルの保存先は、PowerShellから次のコマンドで確認できます。

```powershell
Get-Command openocd | Select-Object Source
```

私の環境では、次の場所にインストールされていました。

```text
C:\Users\yamao\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.12.0-7.1\.content\bin\openocd.exe
```

また、OpenOCDが設定ファイルをどこから検索しているかは、次のコマンドで確認できます。

```powershell
openocd -c "echo [find interface/cmsis-dap.cfg]; echo [find target/rp2040.cfg]; shutdown"
```

実行結果は次のとおりです（起動時のバージョン表示などは省略）。

```text
C:/Users/yamao/AppData/Roaming/xPacks/@xpack-dev-tools/openocd/0.12.0-7.1/.content/bin/../openocd/scripts/interface/cmsis-dap.cfg
C:/Users/yamao/AppData/Roaming/xPacks/@xpack-dev-tools/openocd/0.12.0-7.1/.content/bin/../openocd/scripts/target/rp2040.cfg
shutdown command invoked
```

`bin/..` は、`bin` の1つ上のディレクトリを意味します。したがって、2つの設定ファイルは、実際には次のディレクトリ以下に配置されています。

```text
C:/Users/yamao/AppData/Roaming/xPacks/@xpack-dev-tools/openocd/0.12.0-7.1/.content/openocd/scripts/
```

ここで、OpenOCDをどのディレクトリからでも実行できる理由も分かります。

- `openocd.exe` は、Windowsの環境変数 `PATH` から検索される。
- `-f` で指定した `.cfg` は、OpenOCD自身のスクリプト検索パスから検索される。

つまり、**実行ファイルと設定ファイルは、それぞれ別の仕組みで検索されます。**

そのため、カレントディレクトリがTry Kernelのソースコードの場所であっても、OpenOCDを起動できます。

## 起動後の動作

正常に起動すると、OpenOCDはCMSIS-DAP対応のDebug Probeを認識し、SWD経由でRP2040に接続します。

RP2040にはCortex-M0+が2コア搭載されているため、OpenOCDのログには `rp2040.core0` と `rp2040.core1` が表示されます。

また、OpenOCDはGDBからの接続を受け付けるためのGDBサーバを起動します。標準ではTCPポート3333を使用します。

```text
[rp2040.core0] Cortex-M0+ r0p1 processor detected
[rp2040.core1] Cortex-M0+ r0p1 processor detected
[rp2040.core0] starting gdb server on 3333
```

GDBからは、次のコマンドでOpenOCDに接続します。

```gdb
target remote localhost:3333
```

`localhost` は自分自身のPCを表し、`3333` はOpenOCDが待ち受けているTCPポート番号です。

今回はOpenOCDとGDBを同じWindows PC上で実行するため、`localhost` を指定しています。

なお、ポート番号はOpenOCDの設定で変更できますが、本記事では標準の3333を使用します。

OpenOCDは起動したままにして、別のPowerShellからGDBを実行します。

---

# GDBの起動と接続

別のPowerShellを開き、デバッグ対象のELFファイルを指定してGDBを起動します。

```powershell
arm-none-eabi-gdb .\kernel.elf
```

`kernel.elf` は説明用のファイル名です。実際にはTry Kernelのビルドで生成したELFファイルを指定します。

ELFファイルには、実行コードに加えて、ビルド時の設定によってはデバッグ用のシンボル情報などが含まれます。ソースコードと対応させてデバッグするには、デバッグ情報を含むELFファイルを使用します。

GDBが起動したら、OpenOCDへ接続します。

```gdb
target remote localhost:3333
```

ここで指定した `3333` は、OpenOCDが待ち受けているGDBサーバのポート番号です。

続いて、RP2040をリセットして停止します。

```gdb
monitor reset halt
```

`monitor` は、GDBからOpenOCD固有のコマンドを実行するための仕組みです。

つまり、`reset halt` はGDB自身のコマンドではなく、GDBを経由してOpenOCDに渡されるコマンドです。

# 基本的なデバッグ操作

接続後は、GDBからプログラムの実行を制御できます。

| コマンド | 内容 |
|---|---|
| `monitor reset halt` | マイコンをリセットして停止 |
| `break main` | `main` 関数にブレークポイントを設定 |
| `continue` | 実行を再開 |
| `Ctrl+C` | 実行中のプログラムを停止 |
| `next` | 次のソース行へ進む（関数内部には入らない） |
| `step` | 次のソース行へ進む（関数内部にも入る） |
| `print 変数名` | 変数の値を表示 |
| `info registers` | CPUレジスタを表示 |
| `x/16wx アドレス` | 指定アドレスから32ビット値を16個表示 |
| `quit` | GDBを終了 |

## ブレークポイントを設定する

例えば、`main` 関数で停止させる場合は、次のように操作します。

```gdb
monitor reset halt
break main
continue
```

プログラムが `main` に到達すると停止します。

なお、対象のELFファイルにシンボル情報が含まれている必要があります。また、すでに `main` を通過している場合は、リセットなどで再実行する必要があります。

## ステップ実行する

停止した状態から、次のソース行へ進めるには `next` を使用します。

```gdb
next
```

関数の内部に入って処理を追いかけたい場合は、`step` を使用します。

```gdb
step
```

## レジスタやメモリを確認する

CPUレジスタを表示するには、次のコマンドを使用します。

```gdb
info registers
```

メモリの内容を確認するには、`x` コマンドを使用します。

```gdb
x/16wx 0x20000000
```

この例では、SRAMの先頭アドレスから32ビット単位で16個の値を16進数表示します。

`x/16wx` の意味は次のとおりです。

- `16`：表示する個数
- `w`：1個あたり4バイト（32ビット）
- `x`：16進数で表示

# OpenOCDとGDBの終了

GDBを終了するには、GDBのプロンプトから次のコマンドを実行します。

```gdb
quit
```

OpenOCDはGDBとは別プロセスとして動作しているため、GDBを終了しても通常は起動したままです。

OpenOCDを終了するには、OpenOCDを実行しているPowerShellで `Ctrl+C` を入力します。

# おわりに

OpenOCDとGDBは、それぞれ独立したツールですが、組み合わせることでマイコンの実機デバッグを行えます。

OpenOCDがデバッグプローブを介したハードウェア制御を担当し、GDBがブレークポイントやステップ実行などのデバッグ操作を担当します。

今回はコマンドラインから操作しましたが、VSCodeなどのGUIを利用する場合も、基本的な仕組みは変わりません。

次回は、今回のコマンドライン操作がGUI上でどのように実現されるのかを確認したいと思います。
