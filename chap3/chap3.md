# 第3章 プロセス管理

仮想記憶を利用しない場合の単純なプロセス管理について。

プロセスの生成には大きく2つの目的がある。

- 同じプログラムの処理を複数のプロセスに分ける：Webサーバによる複数リクエストの受付
- 別のプログラムを生成する：bashを介した各種プログラムの実行

## `fork()`

`fork()`を発行すると、発行したプロセス（親プロセス）をもとに新しいプロセス（子プロセス）を生成する（内部的にはシステムコール`clone()`の発行）。

1. 子プロセス用メモリ領域を作成、親プロセスのメモリをコピーし子プロセスを生成する
1. 子プロセスを実行する
1. 必要に応じ、`fork()`の返り値（0：子プロセス、子のプロセスID（> 1）：親プロセス）を元に処理を分岐する

## `execve()`

`execve()`を発行すると、指定したプログラムで現在のプロセスを置換し実行する。

1. 実行ファイルを読み出し、プロセスのメモリマップに必要な情報を読み出す
1. 呼び出し元プロセスのメモリを新しいプロセスのデータで上書きする
1. 新しいプロセスの最初から実行する

## メモリマップ

実行ファイルには、実行時に利用するコードとデータのほか次のような情報が保持され、この情報に基づいてメモリ上（厳密には仮想記憶上）にマッピングされる。

- コード領域の情報
  - ファイル上オフセット
  - 領域サイズ
  - メモリマップ開始アドレス
- データ領域の情報：コード領域と同様
- エントリポイント：命令実行開始位置のメモリアドレス

## ELF

実行ファイルのバイナリコードはセグメントと呼ばれるブロックにグループ化されており、それぞれが特定のデータを含む。

Linux環境の多くでは、オブジェクトファイルおよび実行ファイルのフォーマットとしてELF (Executable Linkable Format) が採用されている。
ELFファイルは次のようなセグメントを含んでいる。

- text: 実行可能コード
- .bss: 初期値がゼロのデータ
- data: 初期値のあるデータ
- symtab: コード内の識別子（シンボルテーブル）

`readelf`コマンドにより、ELFファイルの各種情報を確認できる。

`sleep`コマンド (/bin/sleep) について、ELFヘッダ (`readelf -h`) およびセクションヘッダ (`readelf -S`) の情報を表示し、メモリマップに必要な情報を確認する。

```sh
$ readelf -h /bin/sleep
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  ...
  Entry point address:               0x1b70
  Start of program headers:          64 (bytes into file)
  Start of section headers:          33208 (bytes into file)
  ...
```

エントリポイントが確認できる。

```sh
$ readelf -S /bin/sleep
There are 28 section headers, starting at offset 0x81b8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
...
  [14] .text             PROGBITS         00000000000018d0  000018d0
       0000000000003989  0000000000000000  AX       0     0     16
...
  [24] .data             PROGBITS         0000000000208000  00008000
       0000000000000080  0000000000000000  WA       0     0     32
...
```

`.text`はコード領域、`.data`はデータ領域を示している。

`readelf`から得られたメモリマップの情報は次のようになる。

| | 情報 | 値 |
| :- | :- | :- |
| コード領域 | ファイルオフセット | 0x18d0 |
| | サイズ | 0x3989 (= 14729) |
| | メモリマップ開始アドレス | 0x18d0 |
| データ領域 | ファイルオフセット | 0x8000 |
| | サイズ | 0x80 (= 128) |
| | メモリマップ開始アドレス | 0x208000 |
| | エントリポイント | 0x1b70 |

プログラム実行時に作成されたプロセスのメモリマップは、ファイル`/proc/<PID>/maps`で参照できる。

```sh
$ /bin/sleep 10000 &
[1] 3754
$ cat /proc/3754/maps
563b5a090000-563b5a097000 r-xp 00000000 08:02 5636249                    /bin/sleep
563b5a297000-563b5a298000 r--p 00007000 08:02 5636249                    /bin/sleep
563b5a298000-563b5a299000 rw-p 00008000 08:02 5636249                    /bin/sleep
563b5a914000-563b5a935000 rw-p 00000000 00:00 0                          [heap]
...
```

別の新規プロセスを生成する場合は、親プロセスで`fork()`を発行した後、生成された子プロセスで`execve()`を発行する fork_and_exec の形を取ることが多い。

## `_exit()`, `exit()`

プログラムの終了時には`_exit()`を実行（内部的にはシステムコール`exit_group()`の発行）し、プロセスに割り当てていたメモリをすべて回収する。

標準Cライブラリの`exit()`関数は、終了処理として登録されている関数の呼び出しやストリームのフラッシュを行った上で`_exit()`を発行し、プログラムを終了する。

`_exit()`はシステムコールのラッパー関数、`exit()`は標準ライブラリの関数である。
