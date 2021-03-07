---
layout: lecture
title: "デバッグとプロファイリング"
date: 2020-01-23
ready: true
video:
  aspect: 56.25
  id: l812pUnKxME
---

コードはあなたが期待した動きをするのではなく、あなたが書いた通りのことをするという黄金律がプログラミングにはあります。
その差を埋めるのはときには大変困難な芸当になる場合もあります。
この授業では、バグっていたりリソースを消費したりするようなコードに対処するための便利なテクニック、デバッグとプロファイリングについて扱います。

# デバッグ

## printf デバッグとロギング

「最も効果的なデバッグツールは依然として、思慮深く配置された print 文に伴った注意深い考察である」 — Brian Kernighan, _Unix for Beginners_

プログラムをデバッグする最初のアプローチは、何がこの問題に関係しているのかを理解するために必要な情報を得るまで、問題がありそうな場所に print 文を足すことを繰り返すというものです。

２番めのアプローチは、逐次 print 文を足す代わりにロギングを利用するというものです。ロギングは普通のprint文よりも、以下のようないくつかの理由により優れています。

- ログを標準出力の代わりにファイルやソケット、リモートにあるサーバーに出力できます。
- ロギングは深刻度（ INFO 、 DEBUG 、 WARN 、 ERROR など）をサポートしており、出力を適切にフィルターすることができます。
- 新しい問題が発生した際、何が悪かったのかを検出するために必要な問題がログにすでに含まれている可能性が結構あります。

[これ](/static/files/logger.py) はログメッセージのサンプルコードです。

```bash
$ python logger.py
# printだけを利用した生の出力
$ python logger.py log
# 整形されたログの出力
$ python logger.py log ERROR
# ERROR以上のレベルのみを表示
$ python logger.py color
# 色付きの整形された出力
```

ログを読みやすくする私のよくつかうコツのひとつとして、色を付けるというものがあります。
これまでにおそらくすでにターミナルを読みやすくするため色が使われているのに気づいたことでしょう。これはどうすればできるのでしょうか？

`ls` や `grep` のようなプログラムは、 [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code) というシェルに出力の色を変えるように伝えるための特別な文字列を利用しています。例えば `echo -e "\e[38;2;255;0;0mThis is red\e[0m"` を実行すると、ターミナルが [true color](https://gist.github.com/XVilka/8346728#terminals--true-color) をサポートしている場合は `This is red` が赤色で出力されます。 もしターミナルがそれをサポートしていなかった場合 （例えば macOS の Terminal.app）、より普遍的にサポートされている１６色のエスケープコード、例えば `echo -e "\e[31;1mThis is red\e[0m"` を使うことができます。

以下のスクリプトは、どうやっていろいろなRGBカラーをターミナルに表示するかを表しています（ただし true color をサポートしている場合に限ります）。

```bash
#!/usr/bin/env bash
for R in $(seq 0 20 255); do
    for G in $(seq 0 20 255); do
        for B in $(seq 0 20 255); do
            printf "\e[38;2;${R};${G};${B}m█\e[0m";
        done
    done
done
```

## 第三者のログ

大きなソフトウェア・システムを構築し始めるにつれて、独立して動く他のプログラムとの依存関係の問題に直面することでしょう。
ウェブサーバー、データベース、メッセージブローカーといったものはこのような依存関係のよくある例です。
これらのシステムとやり取りをするときはクライアントサイドのエラーメッセージでは不十分なために、しばしばそれらのログを読む必要があります。

幸運なことに、多くのプログラムではシステムのどこかしらにログを書き込んでいます。
UNIX システムでは、プログラムがログを `/var/log` に書くのが一般的です。
例えば、 [NGINX](https://www.nginx.com/) ウェブサーバーはログを `/var/log/nginx` に置きます。
より最近では様々なシステムが、すべてのログを集める場所である **システムログ** を使い始めるようになりました。
（全てではないですが）ほとんどの Linux システムでは、有効化され実行されているサービスたちを始めとする多くのものを管理するシステムデーモン、 `systemd` を利用しています。
`systemd` はそのログを独自のフォーマットで `/var/log/journal` に置き、あなたは [`journalctl`](https://www.man7.org/linux/man-pages/man1/journalctl.1.html) コマンドを使うことでメッセージを表示することができます。
同様に、 macOS では依然として `/var/log/system.log` が存在しますが、徐々にシステムログを利用するツールの数は増えており、 [`log show`](https://www.manpagez.com/man/1/log/) で表示することができます。
ほとんどの UNIX システムでは [`dmesg`](https://www.man7.org/linux/man-pages/man1/dmesg.1.html) コマンドを利用してカーネルのログにアクセスすることもできます。

システムログにログを取るためには、シェルプログラム [`logger`](https://www.man7.org/linux/man-pages/man1/logger.1.html) を使うことができます。
これは `logger` を使う例と、システムログに作られたエントリーを確認する方法です。
さらに、ほとんどのプログラミング言語ではシステムログにログを吐くためのバインディングが存在します。

```bash
logger "Hello Logs"
# On macOS
log show --last 1m | grep Hello
# On Linux
journalctl --since "1m ago" | grep Hello
```

「データの前処理」の授業で見たように、欲しい情報を得るためにはログをできる限り詳細にし、ある程度の処理とフィルタリングをすることが必要です。
もし `journalctl` や `log show` でフィルタリングしすぎているなと思ったら、フラグを使って予め出力をフィルターすることができます。
また、 [`lnav`](http://lnav.org/) のような、よりよい見た目やログファイル間の行き来をする機能を提供するようなツールがあります。

## デバッガー

printfデバッギングが十分ではないときは、デバッガーを使うべきです。
デバッガーは実行されているプログラムを操作するためのプログラムで、以下のような事ができます。

- 特定の行にきたときにプログラムの実行を止める
- １命令ずつプログラムの実行を行う
- プログラムがクラッシュした後に変数の値を調査する
- ある条件を満たしたときにだけ実行を止める
- などなど他にもより発展的な機能があります

多くのプログラミング言語は何らかの形でデバッガーを備えています。
Python では Python Debugger [`pdb`](https://docs.python.org/3/library/pdb.html) です。

これは `pdb` がサポートするコマンドのいくつかの簡単な紹介です。

- **l**(ist) - 今実行している行の周り11行か、前に表示していたコードを表示します。
- **s**(tep) - 今の行を実行し、最初の停止できる場所で止まります。
- **n**(ext) - 現在の関数の次の行まで、もしくは今の関数からリターンするまで実行を続けます。
- **b**(reak) - 渡した引数に応じた breakpoint を設定します。
- **p**(rint) - 現在のコンテキストで式を評価し、値を表示します。 [`pprint`](https://docs.python.org/3/library/pprint.html) を代わりに利用して表示をする、 **pp** コマンドもあります。
- **r**(eturn) - 現在の関数からリターンするまでプログラムを実行します。
- **q**(uit) - デバッガーを終了します。

`pdb` をつかって、このバグがある Python のコードを直してみましょう（授業のビデオをみてください）。

```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(n):
            if arr[j] > arr[j+1]:
                arr[j] = arr[j+1]
                arr[j+1] = arr[j]
    return arr

print(bubble_sort([4, 2, 1, 8, 7, 6]))
```

Python はインタープリター言語であり、コマンドやプログラムを実行できる `pdb` シェルがあります。
[`ipdb`](https://pypi.org/project/ipdb/) は [`IPython`](https://ipython.org) の REPL を用いた改良版の `pdb` で、`pdb` モジュールと同じインターフェースに加えて、タブキーでの予測変換が効いたり、シンタックスハイライティング、わかりやすいトレースバック、イントロスペクションなどが使えます。

より低いレベルのプログラミングでは [`gdb`](https://www.gnu.org/software/gdb/) （に QoL を高める変更を加えた [`pwndbg`](https://github.com/pwndbg/pwndbg)) や [`lldb`](https://lldb.llvm.org/) ）を検討することでしょう。
これらは C のような言語のデバッグに最適化されていますが、ほとんどどのようなプロセスでも調査でき、レジスタ、スタック、プログラムカウンタなどのマシンの状態を得ることができるでしょう。

## 特化したツール

たとえデバッグをしようとしている対象がブラックボックスのバイナリーだったとしても、あなたがデバッグをするのを手助けするツールがあります。
Linux カーネルしかできないようなことをする必要がある場合には、プログラムは [System Calls](https://en.wikipedia.org/wiki/System_call) を使います。
プログラムが発行したシステムコールを追跡するいくつかのコマンドがあります。 Linuxシステムでは [`strace`](https://www.man7.org/linux/man-pages/man1/strace.1.html) 、macOS や BSD では [`dtrace`](http://dtrace.org/blogs/about/) があります。 `dtrace` は独自の `D` 言語を利用する必要があるため使うのに癖がありますが、 `strace` と似たインターフェースを提供する [`dtruss`](https://www.manpagez.com/man/1/dtruss/) というラッパーがあります（詳細は[こちら](https://8thlight.com/blog/colin-jones/2015/11/06/dtrace-even-better-than-strace-for-osx.html)）。

これは `strace` か `dtruss` をつかって `ls` を実行したときの [`stat`](https://www.man7.org/linux/man-pages/man2/stat.2.html) システムコールを追跡した例です。より深く `strace` について知るためには、 [これ](https://blogs.oracle.com/linux/strace-the-sysadmins-microscope-v2) を読むとよいでしょう。

```bash
# On Linux
sudo strace -e lstat ls -l > /dev/null

# On macOS
sudo dtruss -t lstat64_extended ls -l > /dev/null
```

状況によっては、あなたのプログラムの問題を理解するためにネットワークパケットを見る必要があるかもしれません。
 [`tcpdump`](https://www.man7.org/linux/man-pages/man1/tcpdump.1.html) や [Wireshark](https://www.wireshark.org/) のようなツールはネットワークのパケットを分析するソフトウェアで、ネットワークのパケットを読んだり様々な基準でフィルターしたりすることができます。

ウェブ開発においては、ChromeやFirefoxの開発者ツールがとても便利です。たくさんの機能がありますが、たとえば以下のようなものが含まれています：
- ソースコード - あらゆるサイトの HTML/CSS/JS のソースコードを調査する。
- 動的な HTML, CSS, JS の変更 - ウェブサイトの中身、スタイルや動作を変えてテストする（ウェブサイトのスクリーンショットが証拠として確実なものではないことがよく分かることでしょう）。
- Javascript のシェル - JSのREPLでコマンドを実行する。
- ネットワーク - リクエストのタイムラインを分析する。
- ストレージ - クッキーやアプリケーションのストレージを見る。

## 静的解析

いくつかの問題に関しては、コードを実際に走らせる必要は全くありません。
たとえば、じっくりコードを読むだけでもループ変数がすでにある変数や関数名を隠している（ shadowing ）ことや、プログラムが変数を定義する前に読み込んでいることに気づくかもしれません。
これは、[静的解析](https://en.wikipedia.org/wiki/Static_program_analysis)ツールが役に立つ分野です。
静的解析のプログラムはソースコードを入力とし、その正しさを判断するためにコーディング規約に基づいて分析を行います。

以下の Python のコードはいくつかの誤りがあります。
はじめに、ループ変数である `foo` はすでに定義されている関数 `foo` を隠しています。また、最後の行では `bar` ではなく `baz` と書いているので、プログラムは１分間かかる `sleep` を実行したあとにクラッシュしてしまいます。


```python
import time

def foo():
    return 42

for foo in range(5):
    print(foo)
bar = 1
bar *= 0.2
time.sleep(60)
print(baz)
```

静的解析ツールはこういった問題を見つけ出すことができます。 [`pyflakes`](https://pypi.org/project/pyflakes) を走らせると両方のバグに関するエラーを確認することができます。また、 [`mypy`](http://mypy-lang.org/) を使うと型チェックをすることができます。この例で `mypy` は、 `bar` がはじめは `int` で初期化されているが後に `float` にキャストされている、という警告を出すでしょう。
繰り返しになりますが、これらの問題はコードを走らせることなく検出することができます。

```bash
$ pyflakes foobar.py
foobar.py:6: redefinition of unused 'foo' from line 3
foobar.py:11: undefined name 'baz'

$ mypy foobar.py
foobar.py:6: error: Incompatible types in assignment (expression has type "int", variable has type "Callable[[], Any]")
foobar.py:9: error: Incompatible types in assignment (expression has type "float", variable has type "int")
foobar.py:11: error: Name 'baz' is not defined
Found 3 errors in 1 file (checked 1 source file)
```

シェルツールの講義では、シェルスクリプトのための似たようなツールである [`shellcheck`](https://www.shellcheck.net/) を扱いました。

ほとんどのエディタやIDEでは、それらのツールの出力をエディタ内に表示し、警告やエラーの場所をハイライトする機能をサポートしています。
これらはよく **code linting** と呼ばれ、コードのスタイル違反や安全でない書き方といったエラーなどを表示するのにも使われます。

vim では、 [`ale`](https://vimawesome.com/plugin/ale) や [`syntastic`](https://vimawesome.com/plugin/syntastic) といったプラグインで lint を行うことができます。
Python では、 [`pylint`](https://github.com/PyCQA/pylint) や [`pep8`](https://pypi.org/project/pep8/) がスタイルの linter の代表的なものであり、 [`bandit`](https://pypi.org/project/bandit/)　は一般的なセキュリティーに関する問題を見つけるためのツールです。
他の言語では、多くの人が便利な静的解析ツールのリストを作成しています。たとえば [Awesome Static Analysis](https://github.com/mre/awesome-static-analysis) (きっとあなたは _Writing_ の章に興味があるでしょう) や、 linter でに関しては [Awesome Linters](https://github.com/caramelomartins/awesome-linters) があります。

スタイルの linting を行う補助的なツールとして、Python では [`black`](https://github.com/psf/black) 、 Go では `gofmt` 、 Rust では `rustfmt` 、 JavaScript、HTML、CSS では [`prettier`](https://prettier.io/) のようなコードフォーマッターがあります。
これらのツールは、そのプログラミング言語の一般的なスタイルに合うようにあなたのコードを自動でフォーマットしてくれます。
あなたはもしかすると自身のコードのスタイルに関して自分でコントロールできないことが気に入らないかもしれませんが、コードの書き方を標準化することは他の人があなたのコードを読むときに助けになりますし、あなたも他の人の（スタイルが標準化されている）コードを読みやすくなるでしょう。


# プロファイリング

たとえあなたのコードが期待した動作をしているように振る舞っていたとしても、もしCPUやメモリをすべて使い果たしていたら、それは十分に良いとは言えないかもしれません。
アルゴリズムの授業ではよく big _O_ 記法を教えますが、あなたのプログラムの中のホットスポットの見つけかたは教えてくれません。
[早すぎる最適化はすべての悪の元凶 (premature optimization is the root of all evil)](http://wiki.c2.com/?PrematureOptimization) ですから、プロファイラーやモニタリングツールを学ばなければなりません。それのツールはプログラムのどの部分が最も時間やリソースを消費しているかを理解する手助けをしてくれるので、その部分を集中して最適化することができるようになります。


## タイミング

デバッグのときと同じように、多くのシナリオではコード中の2点における時刻を表示するだけで十分だったりします。
これは Python で [`time`](https://docs.python.org/3/library/time.html) モジュールを使う例です。

```python
import time, random
n = random.randint(1, 10) * 100

# 現在時刻の取得
start = time.time()

# なんらかの処理
print("Sleeping for {} ms".format(n))
time.sleep(n/1000)

# start と現在時刻の時間を計算
print(time.time() - start)

# 出力
# Sleeping for 500 ms
# 0.5713930130004883
```

しかし、実時間は、あなたのコンピューターが同時に他のプロセスを実行していたり、イベントが発生するのを待っていたりする場合に紛らわしい場合があります。ツールが _Real_、 _User_、 _Sys_ 時間を区別するのはよくあることです。一般的に、 _User_ + _Sys_ はあなたのプロセスが実際に CPU 上で使った時間を表します（より詳細な説明は[こちら](https://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1)）。

- _Real_ - プログラムの最初から最後までに経過した実時間であり、他のプロセスが消費した時間やブロックされていた時間（例： I/O やネットワークの待ち時間）も含む。
- _User_ - CPU 上でユーザーコードが走っていた時間
- _Sys_ - CPU 上でカーネルコードが走っていた時間

例えば、 HTTP リクエストを行うコマンドを [`time`](https://www.man7.org/linux/man-pages/man1/time.1.html) と前につけて実行してみましょう。ネットワークが遅い場合には、結果は以下のようなものになるでしょう。ここでは、リクエストが完了するまでに2sかかっていますが、プロセスは 15ms の CPU ユーザー時間と 12ms の CPU カーネル時間しか使っていません。

```bash
$ time curl https://missing.csail.mit.edu &> /dev/null`
real    0m2.561s
user    0m0.015s
sys     0m0.012s
```

## プロファイラー
### CPU

人々が _プロファイラー_ と言った際にはほとんどが _CPU プロファイラー_ を指し、これは最も一般的なプロファイラーです。
CPU プロファイラーには大きく分けて _トレーシング_ プロファイラーと _サンプリング_ プロファイラーという2つのタイプがあります。
トレーシングプロファイラーはプログラムの中で起こったすべての関数呼び出しを記録する一方で、サンプリングプロファイラーはプログラムを定期的に（一般的には1ミリ秒ごとに）調べ、プログラム中のスタックを記録します。
これらのツールはその記録をつかって、あなたのプログラムがもっとも時間をつかったことに関する統計情報を表示します。
[これ](https://jvns.ca/blog/2017/12/17/how-do-ruby---python-profilers-work-)は、もしより詳細にこのトピックについて知りたい場合に良い導入のための記事となるでしょう。

ほとんどのプログラミング言語は、なんらかのコードを分析するために使えるコマンドライン上のプロファイラーを備えています。
それらはよく一通りの機能を備えたIDEと合わせて利用されますが、この授業ではコマンドラインツールに焦点を当てていきましょう。

Python では、 `cProfile` モジュールを関数呼び出しの時間を測定するために利用できます。これは、原始的な grep を Python で実装した簡単な例です。

```python
#!/usr/bin/env python

import sys, re

def grep(pattern, file):
    with open(file, 'r') as f:
        print(file)
        for i, line in enumerate(f.readlines()):
            pattern = re.compile(pattern)
            match = pattern.search(line)
            if match is not None:
                print("{}: {}".format(i, line), end="")

if __name__ == '__main__':
    times = int(sys.argv[1])
    pattern = sys.argv[2]
    for i in range(times):
        for file in sys.argv[3:]:
            grep(pattern, file)
```

このコードを以下のようなコマンドをつかってプロファイルすることができます。出力を分析することで、 IO がほとんどの時間を消費しており、正規表現をコンパイルするのにかなりの時間を使っていることもわかります。正規表現は一度だけコンパイルすればよいため、その部分を for 文から出すことができるでしょう。

```
$ python -m cProfile -s tottime grep.py 1000 '^(import|\s*def)[^,]*$' *.py

[omitted program output]

 ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     8000    0.266    0.000    0.292    0.000 {built-in method io.open}
     8000    0.153    0.000    0.894    0.000 grep.py:5(grep)
    17000    0.101    0.000    0.101    0.000 {built-in method builtins.print}
     8000    0.100    0.000    0.129    0.000 {method 'readlines' of '_io._IOBase' objects}
    93000    0.097    0.000    0.111    0.000 re.py:286(_compile)
    93000    0.069    0.000    0.069    0.000 {method 'search' of '_sre.SRE_Pattern' objects}
    93000    0.030    0.000    0.141    0.000 re.py:231(compile)
    17000    0.019    0.000    0.029    0.000 codecs.py:318(decode)
        1    0.017    0.017    0.911    0.911 grep.py:3(<module>)

[omitted lines]
```

Python の `cProfile` プロファイラーの注意点として（多くのプロファイラーもそうですが）、関数呼び出しごとにかかる時間を表示しているということがあります。これは、特にサードパーティーのライブラリーをコード中で使っている場合に内部の関数呼び出しも結果に含まれてしまうため、あっという間に直感的ではなくなります。
プロファイリングの情報を表示するのにより直感的な方法は、コードの行ごとにかかった時間を表示する方法であり、 _ラインプロファイラー_ がこれをしてくれます。

例えば、この Python コードは授業のウェブサイトにリクエストを投げ、レスポンスを構文解析することでページ内のすべての URL を得ます。

```python
#!/usr/bin/env python
import requests
from bs4 import BeautifulSoup

# これは、line_profilerにこの関数を
# 解析するよう伝えるデコレーターです。
@profile
def get_urls():
    response = requests.get('https://missing.csail.mit.edu')
    s = BeautifulSoup(response.content, 'lxml')
    urls = []
    for url in s.find_all('a'):
        urls.append(url['href'])

if __name__ == '__main__':
    get_urls()
```

もし Python の `cProfile` プロファイラーを使ったとしたら、2500行を超える、たとえソートしたとしてもどこで時間を使ったのか分かりづらいような出力を得るでしょう。 [`line_profiler`](https://github.com/pyutils/line_profiler) を簡単に走らせれば、行ごとにかかった時間を表示することができます。

```bash
$ kernprof -l -v a.py
Wrote profile results to urls.py.lprof
Timer unit: 1e-06 s

Total time: 0.636188 s
File: a.py
Function: get_urls at line 5

Line #  Hits         Time  Per Hit   % Time  Line Contents
==============================================================
 5                                           @profile
 6                                           def get_urls():
 7         1     613909.0 613909.0     96.5      response = requests.get('https://missing.csail.mit.edu')
 8         1      21559.0  21559.0      3.4      s = BeautifulSoup(response.content, 'lxml')
 9         1          2.0      2.0      0.0      urls = []
10        25        685.0     27.4      0.1      for url in s.find_all('a'):
11        24         33.0      1.4      0.0          urls.append(url['href'])
```

### メモリー

C や C++ のような言語では、メモリリークによってあなたのプログラムが必要のないメモリを決して解放しないということがあります。
メモリーのデバッグを助けるため、メモリリークを検出する [Valgrind](https://valgrind.org/) のようなツールを使うことができます。

ガベージコレクションのある Python のような言語でも、オブジェクトへのポインターを持っている限りオブジェクトはガベージコレクションされないので、メモリープロファイラーを利用するのは役に立ちます。
これは、プログラムの例とそれに対して [memory-profiler](https://pypi.org/project/memory-profiler/) を実行した結果です（`line-profiler` のようにデコレーターを使っていることに注意してください）。

```python
@profile
def my_func():
    a = [1] * (10 ** 6)
    b = [2] * (2 * 10 ** 7)
    del b
    return a

if __name__ == '__main__':
    my_func()
```

```bash
$ python -m memory_profiler example.py
Line #    Mem usage  Increment   Line Contents
==============================================
     3                           @profile
     4      5.97 MB    0.00 MB   def my_func():
     5     13.61 MB    7.64 MB       a = [1] * (10 ** 6)
     6    166.20 MB  152.59 MB       b = [2] * (2 * 10 ** 7)
     7     13.61 MB -152.59 MB       del b
     8     13.61 MB    0.00 MB       return a
```

### イベントプロファイリング

`strace` をデバッグのために使うような場合に、プロファイリング中にプログラムの一部分を無視し、ブラックボックスのように扱いたいかもしれません。
[`perf`](https://www.man7.org/linux/man-pages/man1/perf.1.html) コマンドは CPU の差異を抽象化し時間やメモリを報告しませんが、代わりにあなたのプログラムに関係するシステムイベントを報告します。
例えば、 `perf` を使うことで簡単にキャッシュの局所性が悪いこと、たくさんのページフォルトやロックなどがわかります。以下はコマンドの概要です。

- `perf list` - perf で記録されたイベントの一覧を表示する
- `perf stat COMMAND ARG1 ARG2` - プロセスやコマンドに関係する異なるイベントの数を得る
- `perf record COMMAND ARG1 ARG2` - コマンドの実行を記録して、統計データを `perf.data` というファイルに保存する
- `perf report` -　`perf.data` に記録されたデータを整形して表示する

### 可視化（ Visualization ）

実世界のプログラムからのプロファイラーの出力は、ソフトウェアプロジェクトの元来持つ複雑さにより、大量の情報を含むことでしょう。
人類は視覚を重視する生物であり、大量の数字を読み取り意味を理解するのは大変苦手です。
そのため、プロファイラーの出力を理解しやすい形に表示するツールはたくさんあります。

サンプリングプロファイラーからの CPU のプロファイル情報を表示するよくある方法のひとつに、 [Flame Graph](http://www.brendangregg.com/flamegraphs.html) を使うというものがあります。これは、階層的な関数呼び出しを Y 軸に表示し、かかった時間を X 軸に比例するように表示します。これは対話型であり、プログラムの特定の箇所を拡大して、そのスタックトレースを得ることができます（以下の画像をクリックしてみてください）。

[![FlameGraph](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)

コールグラフ（ Call Graph ）や制御フローグラフ（ Control Flow Graph ）はプログラム中のサブルーチンの関係性を、関数をノード、関数呼び出しを有向エッジとして表示します。呼び出し回数や時間といったプロファイリングの情報とともに扱えば、コールグラフはプログラムの流れを解釈するのに大変役に立ちます。
Python では、 [`pycallgraph`](http://pycallgraph.slowchop.com/en/master/) ライブラリを利用することでそれらを生成することができます。

![Call Graph](https://upload.wikimedia.org/wikipedia/commons/2/2f/A_Call_Graph_generated_by_pycallgraph.png)


## リソースモニタリング

ときには、パフォーマンスを解析するにあたっての最初のステップとして、実際のリソースの消費量がどうなっているかを把握することでしょう。
プログラムはしばしばリソースが足りないとき、たとえば十分なメモリがなかったりネットワークが遅い場合に遅くなります。
 CPU 使用率やメモリ消費量、ネットワーク、ディスクの利用量といったいろいろなシステムのリソースを表示するための無数のツールが存在します。

- **一般的なモニタリング** - おそらく最も有名なものは [`htop`](https://htop.dev/) でしょう。これは改良版の [`top`](https://www.man7.org/linux/man-pages/man1/top.1.html) です。 `htop` はシステム上で実行されているプロセスの様々な統計を表示します。 `htop` は無数のオプションとショートカットキーがあり、有名なものだと `<F6>` でプロセスをソートし、 `t` で階層構造のツリーを表示し、 `h` でスレッドをトグルします。
[`glances`](https://nicolargo.github.io/glances/) も素晴らしい UI を持つ同じようなツールです。すべてのプロセスをまとめた測定結果を得るには [`dstat`](http://dag.wiee.rs/home-made/dstat/) が気の利くツールであり、 I/O 、ネットワーク、 CPU 消費量、コンテキストスイッチといった様々なサブシステムのたくさんのリソースに関する情報をリアルタイムに計算することができます。
- **I/O 操作** - [`iotop`](https://www.man7.org/linux/man-pages/man8/iotop.8.html) は現在の I/O の使用量を表示することができ、プロセスが大量の I/O ディスク操作を行っていないか確認するのに便利です。
- **ディスク使用量** - [`df`](https://www.man7.org/linux/man-pages/man1/df.1.html) はパーティションごとの情報を表示し、 [`du`](http://man7.org/linux/man-pages/man1/du.1.html) はディスク （ **d**isk ）の使用量（ **u**sage ）を現在のディレクトリ内のファイルごとに表示します。これらのツールでは、 `-h` フラグを使うと人間（ **h**uman ）に読みやすい形式で表示することができます。
`du` のより対話的なバージョンとして、 [`ncdu`](https://dev.yorhel.nl/ncdu) というのもあり、これはフォルダーを移動しながら利用でき、ファイルやフォルダを削除することもできます。
- **メモリ使用量** - [`free`](https://www.man7.org/linux/man-pages/man1/free.1.html) はシステム内で使われているメモリーや利用できるメモリーの総量を表示します。メモリーは `htop` のようなツールにも表示されています。
- **使われているファイル** - [`lsof`](https://www.man7.org/linux/man-pages/man8/lsof.8.html) はプロセスによって開かれたファイルについての情報を列挙します。これは、どのプロセスが特定のあるファイルを開いたのか確認するのに大変便利です。
- **ネットワーク接続と設定** - [`ss`](https://www.man7.org/linux/man-pages/man8/ss.8.html) をつかうと、入ってきたり出ていったりするネットワークパケットやインターフェースに関する統計情報を監視することができます。 `ss` コマンドの一般的な利用法としては、マシンの特定のポートを使っているプロセスが何かを調べるというのがあります。ルーティングやネットワークデバイス、インターフェースを表示するには [`ip`](http://man7.org/linux/man-pages/man8/ip.8.html) コマンドが使えます。 `netstat` や `ifconfig` は、これらのツールがあるため非推奨となりました。
- **ネットワーク使用量** - [`nethogs`](https://github.com/raboof/nethogs) や [`iftop`](http://www.ex-parrot.com/pdw/iftop/) はネットワーク使用量を監視するための便利な対話的 CLI ツールです。

もしこれらのツールをテストしたいなら、 [`stress`](https://linux.die.net/man/1/stress) コマンドを使うことでマシンに人工的な負荷をかけることができます。

### 特化したツール

ときには、どのソフトウェアを利用するか決めるために、ブラックボックス化されたベンチマークを取ることが必要だったりします。
[`hyperfine`](https://github.com/sharkdp/hyperfine) のようなツールを使うと、コマンドラインプログラムのベンチマークをすぐに取ることができます。
例えば、シェルツールとスクリプトの授業で `find` コマンドではなく `fd` コマンドを勧めました。 `hyperfine` をつかって、普段我々がよく実行するタスクについてこれらのツールを比べてみましょう。
たとえば、以下のサンプルでは `fd` は `find` より私のマシン上で20倍高速でした。

```bash
$ hyperfine --warmup 3 'fd -e jpg' 'find . -iname "*.jpg"'
Benchmark #1: fd -e jpg
  Time (mean ± σ):      51.4 ms ±   2.9 ms    [User: 121.0 ms, System: 160.5 ms]
  Range (min … max):    44.2 ms …  60.1 ms    56 runs

Benchmark #2: find . -iname "*.jpg"
  Time (mean ± σ):      1.126 s ±  0.101 s    [User: 141.1 ms, System: 956.1 ms]
  Range (min … max):    0.975 s …  1.287 s    10 runs

Summary
  'fd -e jpg' ran
   21.89 ± 2.33 times faster than 'find . -iname "*.jpg"'
```

デバッグのためには、ブラウザーはウェブページの読み込みをプロファイリングするのに素晴らしいツールとなり、ローディング、レンダリング、スクリプティングなどのどこで時間を使ったのかを理解させてくれるでしょう。
[Firefox](https://developer.mozilla.org/en-US/docs/Mozilla/Performance/Profiling_with_the_Built-in_Profiler) や [Chrome](https://developers.google.com/web/tools/chrome-devtools/rendering-tools) についての詳細は、これらのサイトを参照してください。

# 演習
## デバッグ

1. Linux 上で `journalctl` 、もしくは macOS 上で `log show` をつかって最後にスーパーユーザーになって実行したコマンドを取得してください。
もし何もなければ、害のない `sudo ls` のようなコマンドを実行して再度確認してみてください。

1. [この](https://github.com/spiside/pdb-tutorial) `pdb` のハンズオンチュートリアルを行い、コマンドに慣れてください。より詳細なチュートリアルとしては、[これ](https://realpython.com/python-debugging-pdb)を読んでみましょう。

1. [`shellcheck`](https://www.shellcheck.net/) をインストールし、以下のスクリプトを確認してみてください。このコードの何が間違っているでしょうか？それを修正してみましょう。また、エディタに linter のプラグインをインストールし、自動的に警告が出るようにしてみましょう。

   ```bash
   #!/bin/sh
   ## Example: a typical script with several problems
   for f in $(ls *.m3u)
   do
     grep -qi hq.*mp3 $f \
       && echo -e 'Playlist $f contains a HQ file in mp3 format'
   done
   ```

1. （発展）[reversible debugging](https://undo.io/resources/reverse-debugging-whitepaper/) について読み、 [`rr`](https://rr-project.org/) や [`RevPDB`](https://morepypy.blogspot.com/2016/07/reverse-debugging-for-python.html) を使って簡単な例に取り組んでみてください。

## プロファイリング

1. [これ](/static/files/sorts.py) はいくつかのソートアルゴリズムを実装したものです。 [`cProfile`](https://docs.python.org/3/library/profile.html) と [`line_profiler`](https://github.com/pyutils/line_profiler) を使い、挿入ソートとクイックソートの実行時間を比べてみましょう。それぞれのアルゴリズムのボトルネックは何でしょうか。それから `memory_profiler` をつかってメモリー消費量を確認してみましょう。なぜ挿入ソートのほうが良いのでしょうか？それから、インプレースのバージョンのクイックソートを確認してみましょう。　チャレンジ： `perf` を使ってサイクルカウントとキャッシュヒット・ミスをそれぞれのアルゴリズムについて確認してみましょう。

1. これは、それぞれの値ごとの関数をつかってフィボナッチ数を求める（あきらかに複雑な） Python のコードです。

   ```python
   #!/usr/bin/env python
   def fib0(): return 0

   def fib1(): return 1

   s = """def fib{}(): return fib{}() + fib{}()"""

   if __name__ == '__main__':

       for n in range(2, 10):
           exec(s.format(n, n-1, n-2))
       # from functools import lru_cache
       # for n in range(10):
       #     exec("fib{} = lru_cache(1)(fib{})".format(n, n))
       print(eval("fib9()"))
   ```

   このコードをファイルに保存し、実行できるようにしてください。また、事前にこれらをインストールしてください：[`pycallgraph`](http://pycallgraph.slowchop.com/en/master/) 、 [`graphviz`](http://graphviz.org/) (もしすでに `dot` コマンドを実行できるなら、 GraphViz はすでにインストールされています)。 コードをこのコマンドをつかって実行してください。 `pycallgraph graphviz -- ./fib.py` そして `pycallgraph.png` を確認してみましょう。何回 `fib0` は呼ばれましたか？関数をメモ化することでこれより良くできます。コメントになっている部分のコメントを外してもう一度画像を生成してみてください。今回はそれぞれの `fibN` 関数は何回呼ばれているでしょうか？

1. リッスンしようとしているポートがすでに他のプロセスに取られていることはよくあります。そのプロセスの pid を見つける方法について学びましょう。最初に、 `python -m http.server 4444` を実行し、 `4444` ポートをリッスンしている最小のウェブサーバーを起動します。異なるターミナルで、 `lsof | grep LISTEN` を実行することですべてのリッスンしているプロセスやポートが表示されます。そのプロセスの pid を見つけ、 `kill <PID>` を実行することで終了させましょう。

1. プロセスのリソースを制限するのは便利なツールとなるかもしれません。 `stress -c 3` を実行し、 `htop` で CPU 使用率を可視化してみましょう。さらに、 `taskset --cpu-list 0,2 stress -c 3` を実行して可視化してみましょう。 `stress` は3つの CPU を使っているでしょうか？なぜ使っていないのでしょう？ [`man taskset`](https://www.man7.org/linux/man-pages/man1/taskset.1.html) を読んでみてください。
チャレンジ：同じことを [`cgroups`](https://www.man7.org/linux/man-pages/man7/cgroups.7.html) を使ってやってみてください。メモリー消費量を `stress -m` をつかって制限してみてください。

1. （発展） `curl ipinfo.io` というコマンドは HTTP リクエストを投げ、あなたのパブリック IP に関する情報を取得します。 [Wireshark](https://www.wireshark.org/) を開き、 `curl` が送受信したリクエストや応答のパケットを監視してみましょう。（ヒント： `http` フィルターを使うと HTTP のパケットのみを見ることができます。）
