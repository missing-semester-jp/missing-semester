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

## Printfデバッグとロギング

「最も効果的なデバッグツールは依然として、思慮深く配置されたprint文に伴った注意深い考察である」 — Brian Kernighan, _Unix for Beginners_

<!-- "The most effective debugging tool is still careful thought, coupled with judiciously placed print statements" — Brian Kernighan, _Unix for Beginners_. -->

プログラムをデバッグする最初のアプローチは、何がこの問題に関係しているのかを理解するために必要な情報を得るまで、問題がありそうな場所にprint文を足すことを繰り返すというものです。

<!-- A first approach to debug a program is to add print statements around where you have detected the problem, and keep iterating until you have extracted enough information to understand what is responsible for the issue. -->

２番めのアプローチは、逐次print文を足す代わりにロギングを利用するというものです。ロギングは普通のprint文よりも、以下のようないくつかの理由により優れています。

<!-- A second approach is to use logging in your program, instead of ad hoc print statements. Logging is better than regular print statements for several reasons: -->

- ログを標準出力の代わりにファイルやソケット、リモートにあるサーバーに出力できます。
- ロギングは深刻度（INFO、DEBUG、WARN、ERRORなど）をサポートしており、出力を適切にフィルターすることができます。
- 新しい問題が発生した際、何が悪かったのかを検出するために必要な問題がログにすでに含まれている可能性が結構あります。

<!-- - You can log to files, sockets or even remote servers instead of standard output.
- Logging supports severity levels (such as INFO, DEBUG, WARN, ERROR, &c), that allow you to filter the output accordingly.
- For new issues, there's a fair chance that your logs will contain enough information to detect what is going wrong. -->

[これ](/static/files/logger.py) はログメッセージのサンプルコードです。

<!-- [Here](/static/files/logger.py) is an example code that logs messages: -->

```bash
$ python logger.py
# Raw output as with just prints
# printだけを利用した生の出力
$ python logger.py log
# Log formatted output
# 整形されたログの出力
$ python logger.py log ERROR
# Print only ERROR levels and above
# ERROR以上のレベルのみを表示
$ python logger.py color
# Color formatted output
# 色付きの整形された出力
```

ログを読みやすくする私のよくつかうコツのひとつとして、色を付けるというものがあります。
これまでにおそらくすでにターミナルを読みやすくするため色が使われているのに気づいたことでしょう。これはどうすればできるのでしょうか？

`ls` や `grep` のようなプログラムは、 [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code) というシェルに出力の色を変えるように伝えるための特別な文字列を利用しています。例えば `echo -e "\e[38;2;255;0;0mThis is red\e[0m"` を実行すると、ターミナルが [true color](https://gist.github.com/XVilka/8346728#terminals--true-color) をサポートしている場合は `This is red` が赤色で出力されます。 もしターミナルがそれをサポートしていなかった場合 （例えば macOS の Terminal.app）、より普遍的にサポートされている１６色のエスケープコード、例えば `echo -e "\e[31;1mThis is red\e[0m"` を使うことができます。

<!--
One of my favorite tips for making logs more readable is to color code them.
By now you probably have realized that your terminal uses colors to make things more readable. But how does it do it?
Programs like `ls` or `grep` are using [ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code), which are special sequences of characters to indicate your shell to change the color of the output. For example, executing `echo -e "\e[38;2;255;0;0mThis is red\e[0m"` prints the message `This is red` in red on your terminal, as long as it supports [true color](https://gist.github.com/XVilka/8346728#terminals--true-color). If your terminal doesn't support this (e.g. macOS's Terminal.app), you can use the more universally supported escape codes for 16 color choices, for example `echo -e "\e[31;1mThis is red\e[0m"`. -->

以下のスクリプトは、どうやっていろいろなRGBカラーをターミナルに表示するかを表しています（ただし true color をサポートしている場合に限ります）。

<!-- The following script shows how to print many RGB colors into your terminal (again, as long as it supports true color). -->

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

<!-- As you start building larger software systems you will most probably run into dependencies that run as separate programs.
Web servers, databases or message brokers are common examples of this kind of dependencies.
When interacting with these systems it is often necessary to read their logs, since client side error messages might not suffice. -->

幸運なことに、多くのプログラムではシステムのどこかしらにログを書き込んでいます。
UNIX システムでは、プログラムがログを `/var/log` に書くのが一般的です。
例えば、 [NGINX](https://www.nginx.com/) ウェブサーバーはログを `/var/log/nginx` に置きます。
より最近では様々なシステムが、すべてのログを集める場所である **システムログ** を使い始めるようになりました。
（全てではないですが）ほとんどの Linux システムでは、有効化され実行されているサービスたちを始めとする多くのものを管理するシステムデーモン、 `systemd` を利用しています。
`systemd` はそのログを独自のフォーマットで `/var/log/journal` に置き、あなたは [`journalctl`](https://www.man7.org/linux/man-pages/man1/journalctl.1.html) コマンドを使うことでメッセージを表示することができます。
同様に、 macOS では依然として `/var/log/system.log` が存在しますが、徐々にシステムログを利用するツールの数は増えており、 [`log show`](https://www.manpagez.com/man/1/log/) で表示することができます。
ほとんどの UNIX システムでは [`dmesg`](https://www.man7.org/linux/man-pages/man1/dmesg.1.html) コマンドを利用してカーネルのログにアクセスすることもできます。

<!-- Luckily, most programs write their own logs somewhere in your system.
In UNIX systems, it is commonplace for programs to write their logs under `/var/log`.
For instance, the [NGINX](https://www.nginx.com/) webserver places its logs under `/var/log/nginx`.
More recently, systems have started using a **system log**, which is increasingly where all of your log messages go.
Most (but not all) Linux systems use `systemd`, a system daemon that controls many things in your system such as which services are enabled and running.
`systemd` places the logs under `/var/log/journal` in a specialized format and you can use the [`journalctl`](https://www.man7.org/linux/man-pages/man1/journalctl.1.html) command to display the messages.
Similarly, on macOS there is still `/var/log/system.log` but an increasing number of tools use the system log, that can be displayed with [`log show`](https://www.manpagez.com/man/1/log/).
On most UNIX systems you can also use the [`dmesg`](https://www.man7.org/linux/man-pages/man1/dmesg.1.html) command to access the kernel log. -->

システムログにログを取るためには、シェルプログラム [`logger`](https://www.man7.org/linux/man-pages/man1/logger.1.html) を使うことができます。
これは `logger` を使う例と、システムログに作られたエントリーを確認する方法です。
さらに、ほとんどのプログラミング言語ではシステムログにログを吐くためのバインディングが存在します。

<!-- For logging under the system logs you can use the [`logger`](https://www.man7.org/linux/man-pages/man1/logger.1.html) shell program.
Here's an example of using `logger` and how to check that the entry made it to the system logs.
Moreover, most programming languages have bindings logging to the system log. -->

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

<!-- As we saw in the data wrangling lecture, logs can be quite verbose and they require some level of processing and filtering to get the information you want.
If you find yourself heavily filtering through `journalctl` and `log show` you can consider using their flags, which can perform a first pass of filtering of their output.
There are also some tools like  [`lnav`](http://lnav.org/), that provide an improved presentation and navigation for log files. -->

## デバッガー
<!-- ## Debuggers -->

printfデバッギングが十分ではないときは、デバッガーを使うべきです。
デバッガーは実行されているプログラムを操作するためのプログラムで、以下のような事ができます。

<!-- When printf debugging is not enough you should use a debugger.
Debuggers are programs that let you interact with the execution of a program, allowing the following: -->

- 特定の行にきたときにプログラムの実行を止める
- １命令ずつプログラムの実行を行う
- プログラムがクラッシュした後に変数の値を調査する
- ある条件を満たしたときにだけ実行を止める
- などなど他にもより発展的な機能があります

<!-- - Halt execution of the program when it reaches a certain line.
- Step through the program one instruction at a time.
- Inspect values of variables after the program crashed.
- Conditionally halt the execution when a given condition is met.
- And many more advanced features -->

多くのプログラミング言語は何らかの形でデバッガーを備えています。
Python では Python Debugger [`pdb`](https://docs.python.org/3/library/pdb.html) です。

<!-- Many programming languages come with some form of debugger.
In Python this is the Python Debugger [`pdb`](https://docs.python.org/3/library/pdb.html). -->

これは `pdb` サポートするコマンドのいくつかの簡単な紹介です。
<!--
Here is a brief description of some of the commands `pdb` supports: -->

- **l**(ist) - 今実行している行の周り11行か、前に表示していたコードを表示します。
- **s**(tep) - 今の行を実行し、最初の停止できる場所で止まります。
- **n**(ext) - 現在の関数の次の行まで、もしくは今の関数からリターンするまで実行を続けます。
- **b**(reak) - 渡した引数に応じた breakpoint を設定します。
- **p**(rint) - 現在のコンテキストで式を評価し、値を表示します。 [`pprint`](https://docs.python.org/3/library/pprint.html) を代わりに利用して表示をする、 **pp** コマンドもあります。
- **r**(eturn) - 現在の関数からリターンするまでプログラムを実行します。
- **q**(uit) - デバッガーを終了します。

<!--
- **l**(ist) - Displays 11 lines around the current line or continue the previous listing.
- **s**(tep) - Execute the current line, stop at the first possible occasion.
- **n**(ext) - Continue execution until the next line in the current function is reached or it returns.
- **b**(reak) - Set a breakpoint (depending on the argument provided).
- **p**(rint) - Evaluate the expression in the current context and print its value. There's also **pp** to display using [`pprint`](https://docs.python.org/3/library/pprint.html) instead.
- **r**(eturn) - Continue execution until the current function returns.
- **q**(uit) - Quit the debugger. -->

`pdb` をつかって、このバグがある Python のコードを直してみましょう（授業のビデオをみてください）。

<!-- Let's go through an example of using `pdb` to fix the following buggy python code. (See the lecture video). -->

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
[`ipdb`](https://pypi.org/project/ipdb/) は改良版の `pdb` で、`pdb` モジュールと同じインターフェースに加えて、タブキーでの予測変換が効く [`IPython`](https://ipython.org) REPL 、シンタックスハイライティング、わかりやすいトレースバック、イントロスペクションなどが使えます。

<!-- Note that since Python is an interpreted language we can use the `pdb` shell to execute commands and to execute instructions.
[`ipdb`](https://pypi.org/project/ipdb/) is an improved `pdb` that uses the [`IPython`](https://ipython.org) REPL enabling tab completion, syntax highlighting, better tracebacks, and better introspection while retaining the same interface as the `pdb` module. -->

より低いレベルのプログラミングでは [`gdb`](https://www.gnu.org/software/gdb/) （にQoLを高める変更を加えた [`pwndbg`](https://github.com/pwndbg/pwndbg)) や [`lldb`](https://lldb.llvm.org/) ）を検討することでしょう。
これらは C のような言語のデバッグに最適化されていますが、ほとんどどのようなプロセスでも調査でき、レジスタ、スタック、プログラムカウンタなどのマシンの状態を得ることができるでしょう。

<!-- For more low level programming you will probably want to look into [`gdb`](https://www.gnu.org/software/gdb/) (and its quality of life modification [`pwndbg`](https://github.com/pwndbg/pwndbg)) and [`lldb`](https://lldb.llvm.org/).
They are optimized for C-like language debugging but will let you probe pretty much any process and get its current machine state: registers, stack, program counter, &c. -->

## 特化したツール
<!-- ## Specialized Tools -->

たとえデバッグをしようとしている対象がブラックボックスのバイナリーだったとしても、あなたがデバッグをするのを手助けするツールがあります。
Linux カーネルしかできないようなことをする必要がある場合には、プログラムは [System Calls](https://en.wikipedia.org/wiki/System_call) を使います。
プログラムが発行したシステムコールを追跡するいくつかのコマンドがあります。 Linuxシステムでは [`strace`](https://www.man7.org/linux/man-pages/man1/strace.1.html) 、macOS や BSD では [`dtrace`](http://dtrace.org/blogs/about/) があります。 `dtrace` は独自の `D` 言語を利用する必要があるため使うのに癖がありますが、 `strace` と似たインターフェースを提供する [`dtruss`](https://www.manpagez.com/man/1/dtruss/) というラッパーがあります（詳細は[こちら](https://8thlight.com/blog/colin-jones/2015/11/06/dtrace-even-better-than-strace-for-osx.html)）。

<!-- Even if what you are trying to debug is a black box binary there are tools that can help you with that.
Whenever programs need to perform actions that only the kernel can, they use [System Calls](https://en.wikipedia.org/wiki/System_call).
There are commands that let you trace the syscalls your program makes. In Linux there's [`strace`](https://www.man7.org/linux/man-pages/man1/strace.1.html) and macOS and BSD have [`dtrace`](http://dtrace.org/blogs/about/). `dtrace` can be tricky to use because it uses its own `D` language, but there is a wrapper called [`dtruss`](https://www.manpagez.com/man/1/dtruss/) that provides an interface more similar to `strace` (more details [here](https://8thlight.com/blog/colin-jones/2015/11/06/dtrace-even-better-than-strace-for-osx.html)). -->

これは `strace` か `dtruss` をつかって `ls` を実行したときの [`stat`](https://www.man7.org/linux/man-pages/man2/stat.2.html) システムコールを追跡した例です。より深く `strace` について知るためには、 [これ](https://blogs.oracle.com/linux/strace-the-sysadmins-microscope-v2) を読むとよいでしょう。

<!-- Below are some examples of using `strace` or `dtruss` to show [`stat`](https://www.man7.org/linux/man-pages/man2/stat.2.html) syscall traces for an execution of `ls`. For a deeper dive into `strace`, [this](https://blogs.oracle.com/linux/strace-the-sysadmins-microscope-v2) is a good read. -->

```bash
# On Linux
sudo strace -e lstat ls -l > /dev/null
4
# On macOS
sudo dtruss -t lstat64_extended ls -l > /dev/null
```

状況によっては、あなたのプログラムの問題を理解するためにネットワークパケットを見る必要があるかもしれません。
 [`tcpdump`](https://www.man7.org/linux/man-pages/man1/tcpdump.1.html) や [Wireshark](https://www.wireshark.org/) のようなツールはネットワークのパケットを分析するソフトウェアで、ネットワークのパケットを読んだり様々な基準でフィルターしたりすることができます。

<!-- Under some circumstances, you may need to look at the network packets to figure out the issue in your program.
Tools like [`tcpdump`](https://www.man7.org/linux/man-pages/man1/tcpdump.1.html) and [Wireshark](https://www.wireshark.org/) are network packet analyzers that let you read the contents of network packets and filter them based on different criteria. -->

ウェブ開発においては、ChromeやFirefoxの開発者ツールがとても便利です。たくさんの機能がありますが、たとえば以下のようなものが含まれています：
- ソースコード - あらゆるサイトの HTML/CSS/JS のソースコードを調査する。
- 動的な HTML, CSS, JS の変更 - ウェブサイトの中身、スタイルや動作を変えてテストする（ウェブサイトのスクリーンショットが証拠として確実なものではないことがよく分かることでしょう）。
- Javascript のシェル - JSのREPLでコマンドを実行する。
- ネットワーク - リクエストのタイムラインを分析する。
- ストレージ - クッキーやアプリケーションのストレージを見る。


<!-- For web development, the Chrome/Firefox developer tools are quite handy. They feature a large number of tools, including: -->
<!-- - Source code - Inspect the HTML/CSS/JS source code of any website.
- Live HTML, CSS, JS modification - Change the website content, styles and behavior to test (you can see for yourself that website screenshots are not valid proofs).
- Javascript shell - Execute commands in the JS REPL.
- Network - Analyze the requests timeline.
- Storage - Look into the Cookies and local application storage. -->

## 静的解析
<!-- ## Static Analysis -->

いくつかの問題に関しては、コードを実際に走らせる必要は全くありません。。
たとえば、じっくりコードを読むだけでもループ変数がすでにある変数や関数名を隠している（shadowing）ことや、プログラムが変数を定義する前に読み込んでいることに気づくかもしれません。
これは、 [静的解析](https://en.wikipedia.org/wiki/Static_program_analysis) ツールが役に立つ分野です。
静的解析のプログラムはソースコードを入力とし、その正しさを判断するためにコーディング規約に基づいて分析を行います。

以下のパイソンのコードはいくつかの誤りがあります。
はじめに、ループ変数である `foo` はすでに定義されている関数 `foo` を隠しています。また、最後の行では `bar` ではなく `baz` と書いているので、プログラムは１分間かかる `sleep` を実行したあとにクラッシュしてしまいます。

<!-- For some issues you do not need to run any code.
For example, just by carefully looking at a piece of code you could realize that your loop variable is shadowing an already existing variable or function name; or that a program reads a variable before defining it.
Here is where [static analysis](https://en.wikipedia.org/wiki/Static_program_analysis) tools come into play.
Static analysis programs take source code as input and analyze it using coding rules to reason about its correctness.

In the following Python snippet there are several mistakes.
First, our loop variable `foo` shadows the previous definition of the function `foo`. We also wrote `baz` instead of `bar` in the last line, so the program will crash after completing the `sleep` call (which will take one minute). -->

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

静的解析ツールはこういった問題を見つけ出すことができます。 [`pyflakes`](https://pypi.org/project/pyflakes) を走らせると両方のバグに関するエラーを確認することができます。また、 [`mypy`](http://mypy-lang.org/) を使うと型チェックをすることができます。この例では、 `mypy` は `bar` ははじめは `int` で初期化されますが、後に `float` にキャストされるという警告を出すでしょう。
繰り返しになりますが、これらの問題はコードを走らせることなく検出することができます。

<!-- Static analysis tools can identify this kind of issues. When we run [`pyflakes`](https://pypi.org/project/pyflakes) on the code we get the errors related to both bugs. [`mypy`](http://mypy-lang.org/) is another tool that can detect type checking issues. Here, `mypy` will warn us that `bar` is initially an `int` and is then casted to a `float`.
Again, note that all these issues were detected without having to run the code. -->

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

<!-- In the shell tools lecture we covered [`shellcheck`](https://www.shellcheck.net/), which is a similar tool for shell scripts. -->

ほとんどのエディタやIDEでは、それらのツールの出力をエディタ内に表示し、警告やエラーの場所をハイライトする機能をサポートしています。
これらはよく **code linting** と呼ばれ、コードのスタイル違反や安全でない書き方といったエラーなどを表示するのにも使われます。

<!-- Most editors and IDEs support displaying the output of these tools within the editor itself, highlighting the locations of warnings and errors.
This is often called **code linting** and it can also be used to display other types of issues such as stylistic violations or insecure constructs. -->

vim では、 [`ale`](https://vimawesome.com/plugin/ale) や [`syntastic`](https://vimawesome.com/plugin/syntastic) といったプラグインで lint を行うことができます。
Python では、 [`pylint`](https://github.com/PyCQA/pylint) や [`pep8`](https://pypi.org/project/pep8/) がスタイルの linter の代表的なものであり、 [`bandit`](https://pypi.org/project/bandit/)　は一般的なセキュリティーに関する問題を見つけるためのツールです。
他の言語では、多くの人が便利な静的解析ツールのリストを作成しています。たとえば [Awesome Static Analysis](https://github.com/mre/awesome-static-analysis) (きっとあなたは _Writing_ の章に興味があるでしょう) や、 linter でに関しては [Awesome Linters](https://github.com/caramelomartins/awesome-linters) があります。

<!-- In vim, the plugins [`ale`](https://vimawesome.com/plugin/ale) or [`syntastic`](https://vimawesome.com/plugin/syntastic) will let you do that.
For Python, [`pylint`](https://github.com/PyCQA/pylint) and [`pep8`](https://pypi.org/project/pep8/) are examples of stylistic linters and [`bandit`](https://pypi.org/project/bandit/) is a tool designed to find common security issues.
For other languages people have compiled comprehensive lists of useful static analysis tools, such as [Awesome Static Analysis](https://github.com/mre/awesome-static-analysis) (you may want to take a look at the _Writing_ section) and for linters there is [Awesome Linters](https://github.com/caramelomartins/awesome-linters). -->

スタイルの linting を行う補助的なツールとして、Python では [`black`](https://github.com/psf/black) 、 Go では `gofmt` 、 Rust では `rustfmt` 、 JavaScript、HTML、CSS では [`prettier`](https://prettier.io/) のようなコードフォーマッターがあります。
これらのツールは、そのプログラミング言語の一般的なスタイルに合うようにあなたのコードを自動でフォーマットしてくれます。
あなたはもしかすると自身のコードのスタイルに関して自分でコントロールできないことが気に入らないかもしれませんが、コードの書き方を標準化することは他の人があなたのコードを読むときに助けになりますし、あなたも他の人の（スタイルが標準化されている）コードを読みやすくなるでしょう。

<!-- A complementary tool to stylistic linting are code formatters such as [`black`](https://github.com/psf/black) for Python, `gofmt` for Go, `rustfmt` for Rust or [`prettier`](https://prettier.io/) for JavaScript, HTML and CSS.
These tools autoformat your code so that it's consistent with common stylistic patterns for the given programming language.
Although you might be unwilling to give stylistic control about your code, standardizing code format will help other people read your code and will make you better at reading other people's (stylistically standardized) code. -->

# プロファイリング

たとえあなたのコードが期待した動作をしているように振る舞っていたとしても、もしCPUやメモリをすべて使い果たしていたら、それは十分に良いとは言えないかもしれません。
アルゴリズムの授業ではよく big _O_ 記法を教えますが、あなたのプログラムの中のホットスポットの見つけかたは教えてくれません。
[早すぎる最適化はすべての悪の元凶 (premature optimization is the root of all evil)](http://wiki.c2.com/?PrematureOptimization) ですから、プロファイラーやモニタリングツールを学ばなければなりません。それのツールはプログラムのどの部分が最も時間やリソースを消費しているかを理解する手助けをしてくれるので、その部分を集中して最適化することができるようになります。

<!-- Even if your code functionally behaves as you would expect, that might not be good enough if it takes all your CPU or memory in the process.
Algorithms classes often teach big _O_ notation but not how to find hot spots in your programs.
Since [premature optimization is the root of all evil](http://wiki.c2.com/?PrematureOptimization), you should learn about profilers and monitoring tools. They will help you understand which parts of your program are taking most of the time and/or resources so you can focus on optimizing those parts. -->

## タイミング
<!-- ## Timing -->

デバッグのときと同じように、多くのシナリオではコード中の2点における時刻を表示するだけで十分だったりします。
これは Python で [`time`](https://docs.python.org/3/library/time.html) モジュールを使う例です。

<!-- similarly to the debugging case, in many scenarios it can be enough to just print the time it took your code between two points.
here is an example in python using the [`time`](https://docs.python.org/3/library/time.html) module. -->

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

しかし、実時間は、あなたのコンピューターが同時に他のプロセスを実行していたり、イベントが発生するのを待っていたりする場合に紛らわしい場合があります。ツールが _Real_、 _User_、 _Sys_ 時間を区別するのは一般的なことです。一般的に、 _User_ + _Sys_ はあなたのプロセスが実際に CPU 上で使った時間を表します（より詳細な説明は[こちら](https://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1)）。

- _Real_ - プログラムの最初から最後までに経過した実時間であり、他のプロセスが消費した時間やブロックされていた時間（例： I/O やネットワークを待つ）も含む。
- _User_ - CPU 上でユーザーコードが走っていた時間
- _Sys_ - CPU 上でカーネルコードが走っていた時間

<!-- However, wall clock time can be misleading since your computer might be running other processes at the same time or waiting for events to happen. It is common for tools to make a distinction between _Real_, _User_ and _Sys_ time. In general, _User_ + _Sys_ tells you how much time your process actually spent in the CPU (more detailed explanation [here](https://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1)). -->

<!-- - _Real_ - Wall clock elapsed time from start to finish of the program, including the time taken by other processes and time taken while blocked (e.g. waiting for I/O or network)
- _User_ - Amount of time spent in the CPU running user code
- _Sys_ - Amount of time spent in the CPU running kernel code -->

例えば、 HTTP リクエストを行うコマンドを [`time`](https://www.man7.org/linux/man-pages/man1/time.1.html) と前につけて実行してみましょう。ネットワークが遅い場合には、結果は以下のようなものになるでしょう。ここでは、リクエストが完了するまでに2sかかっていますが、プロセスは 15ms の CPU ユーザー時間と 12ms の CPU カーネル時間しか使っていません。

<!-- For example, try running a command that performs an HTTP request and prefixing it with [`time`](https://www.man7.org/linux/man-pages/man1/time.1.html). Under a slow connection you might get an output like the one below. Here it took over 2 seconds for the request to complete but the process only took 15ms of CPU user time and 12ms of kernel CPU time. -->

```bash
$ time curl https://missing.csail.mit.edu &> /dev/null`
real    0m2.561s
user    0m0.015s
sys     0m0.012s
```

## プロファイラー
<!-- ## Profilers -->

### CPU

人々が _プロファイラー_ と言った際にはほとんどが _CPU プロファイラー_ を指し、これは最も一般的なものです。
CPU プロファイラには大きく分けて _トレーシング_ プロファイラーと _サンプリング_ プロファイラーという2つのタイプがあります。
トレーシングプロファイラーはプログラムの中で起こったすべての関数呼び出しを記録する一方で、サンプリングプロファイラはプログラムを定期的に（一般的には1ミリ秒ごとに）調べ、プログラム中のスタックを記録します。
これらのツールはその記録をつかって、あなたのプログラムがもっとも時間をつかったことに関する統計情報を表示します。
[これ](https://jvns.ca/blog/2017/12/17/how-do-ruby---python-profilers-work-)は、もしより詳細にこのトピックについて知りたい場合に良い導入のための記事となるでしょう。

<!-- Most of the time when people refer to _profilers_ they actually mean _CPU profilers_,  which are the most common.
There are two main types of CPU profilers: _tracing_ and _sampling_ profilers.
Tracing profilers keep a record of every function call your program makes whereas sampling profilers probe your program periodically (commonly every millisecond) and record the program's stack.
They use these records to present aggregate statistics of what your program spent the most time doing.
[Here](https://jvns.ca/blog/2017/12/17/how-do-ruby---python-profilers-work-) is a good intro article if you want more detail on this topic. -->

ほとんどのプログラミング言語は、なんらかのコードを分析するために使えるコマンドライン上のプロファイラーを備えています。
それらはよく一通りの機能を備えたIDEと合わせて利用されますが、この授業ではコマンドラインツールにフォーカスを当てていきましょう。
<!-- Most programming languages have some sort of command line profiler that you can use to analyze your code.
They often integrate with full fledged IDEs but for this lecture we are going to focus on the command line tools themselves. -->

Python では、 `cProfile` モジュールを関数呼び出しの時間を測定するために利用できます。これは、原始的な grep を Python で実装した簡単な例です。
<!-- In Python we can use the `cProfile` module to profile time per function call. Here is a simple example that implements a rudimentary grep in Python: -->

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

<!-- We can profile this code using the following command. Analyzing the output we can see that IO is taking most of the time and that compiling the regex takes a fair amount of time as well. Since the regex only needs to be compiled once, we can factor it out of the for. -->

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

<!--
A caveat of Python's `cProfile` profiler (and many profilers for that matter) is that they display time per function call. That can become unintuitive really fast, especially if you are using third party libraries in your code since internal function calls are also accounted for.
A more intuitive way of displaying profiling information is to include the time taken per line of code, which is what _line profilers_ do. -->

例えば、この Python コードは授業のウェブサイトにリクエストを投げ、レスポンスを構文解析することでページ内のすべての URL を得ます。
<!-- For instance, the following piece of Python code performs a request to the class website and parses the response to get all URLs in the page: -->

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

<!-- If we used Python's `cProfile` profiler we'd get over 2500 lines of output, and even with sorting it'd be hard to understand where the time is being spent. A quick run with [`line_profiler`](https://github.com/pyutils/line_profiler) shows the time taken per line: -->

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

<!-- In languages like C or C++ memory leaks can cause your program to never release memory that it doesn't need anymore.
To help in the process of memory debugging you can use tools like [Valgrind](https://valgrind.org/) that will help you identify memory leaks. -->

ガベージコレクションのある Python のような言語でも、オブジェクトへのポインターを持っている限りオブジェクトはガベージコレクションされないので、メモリープロファイラーを利用するのは役に立ちます。
これは、プログラムの例とそれに対して [memory-profiler](https://pypi.org/project/memory-profiler/) を実行した結果です（`line-profiler` のようにデコレーターを使っていることに注意してください）。

<!-- In garbage collected languages like Python it is still useful to use a memory profiler because as long as you have pointers to objects in memory they won't be garbage collected.
Here's an example program and its associated output when running it with [memory-profiler](https://pypi.org/project/memory-profiler/) (note the decorator like in `line-profiler`). -->

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
<!-- ### Event Profiling -->

`strace` をデバッグのために使うような場合に、プロファイリング中にプログラムの一部分を無視し、ブラックボックスのように扱いたいかもしれません。
[`perf`](https://www.man7.org/linux/man-pages/man1/perf.1.html) コマンドは CPU の差異を抽象化し時間やメモリを報告しませんが、代わりにあなたのプログラムに関係するシステムイベントを報告します。
例えば、 `perf` を使うことで簡単にキャッシュの局所性が悪いこと、たくさんのページフォルトやロックなどがわかります。以下はコマンドの概要です。

<!-- As it was the case for `strace` for debugging, you might want to ignore the specifics of the code that you are running and treat it like a black box when profiling.
The [`perf`](https://www.man7.org/linux/man-pages/man1/perf.1.html) command abstracts CPU differences away and does not report time or memory, but instead it reports system events related to your programs.
For example, `perf` can easily report poor cache locality, high amounts of page faults or livelocks. Here is an overview of the command: -->

- `perf list` - perf で記録されたイベントの一覧を表示する
- `perf stat COMMAND ARG1 ARG2` - プロセスやコマンドに関係する異なるイベントの数を得る
- `perf record COMMAND ARG1 ARG2` - コマンドの実行を記録して、統計データを `perf.data` というファイルに保存する
- `perf report` -　`perf.data` に記録されたデータを整形して表示する

<!-- - `perf list` - List the events that can be traced with perf
- `perf stat COMMAND ARG1 ARG2` - Gets counts of different events related a process or command
- `perf record COMMAND ARG1 ARG2` - Records the run of a command and saves the statistical data into a file called `perf.data`
- `perf report` - Formats and prints the data collected in `perf.data` -->

### 可視化（ Visualization ）
<!-- ### Visualization -->

実世界のプログラムからのプロファイラーの出力は、ソフトウェアプロジェクトの元来持つ複雑さにより、大量の情報を含むことでしょう。
人類は視覚を重視する生物であり、大量の数字を読み取り意味を理解するのは大変苦手です。
そのため、プロファイラーの出力を理解しやすい形に表示するツールはたくさんあります。

<!-- Profiler output for real world programs will contain large amounts of information because of the inherent complexity of software projects.
Humans are visual creatures and are quite terrible at reading large amounts of numbers and making sense of them.
Thus there are many tools for displaying profiler's output in an easier to parse way. -->

サンプリングプロファイラーからの CPU のプロファイル情報を表示するよくある方法のひとつに、 [Flame Graph](http://www.brendangregg.com/flamegraphs.html) を使うというものがあります。これは、階層的な関数呼び出しを Y 軸に表示し、かかった時間を X 軸に比例するように表示します。これは対話型であり、プログラムの特定の箇所を拡大して、そのスタックトレースを得ることができます（以下の画像をクリックしてみてください）。

<!-- One common way to display CPU profiling information for sampling profilers is to use a [Flame Graph](http://www.brendangregg.com/flamegraphs.html), which will display a hierarchy of function calls across the Y axis and time taken proportional to the X axis. They are also interactive, letting you zoom into specific parts of the program and get their stack traces (try clicking in the image below). -->

[![FlameGraph](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)](http://www.brendangregg.com/FlameGraphs/cpu-bash-flamegraph.svg)

コールグラフ（ Call Graph ）や制御フローグラフ（ Control Flow Graph ）はプログラム中のサブルーチンの関係性を、関数をノード、関数呼び出しを有向エッジとして表示します。呼び出し回数や時間といったプロファイリングの情報とともに扱えば、コールグラフはプログラムの流れを解釈するのに大変役に立ちます。
Python では、 [`pycallgraph`](http://pycallgraph.slowchop.com/en/master/) ライブラリを利用することでそれらを生成することができます。

<!-- Call graphs or control flow graphs display the relationships between subroutines within a program by including functions as nodes and functions calls between them as directed edges. When coupled with profiling information such as the number of calls and time taken, call graphs can be quite useful for interpreting the flow of a program.
In Python you can use the [`pycallgraph`](http://pycallgraph.slowchop.com/en/master/) library to generate them. -->

![Call Graph](https://upload.wikimedia.org/wikipedia/commons/2/2f/A_Call_Graph_generated_by_pycallgraph.png)


## リソースモニタリング
<!-- ## Resource Monitoring -->

ときには、パフォーマンスを解析するにあたっての最初のステップとして、実際のリソースの消費量がどうなっているかを把握することでしょう。
プログラムはしばしばリソースが足りないとき、たとえば十分なメモリがなかったりネットワークが遅い場合に遅くなります。
 CPU 使用率やメモリ消費量、ネットワーク、ディスクの利用量といったいろいろなシステムのリソースを表示するための無数のツールが存在します。

<!-- Sometimes, the first step towards analyzing the performance of your program is to understand what its actual resource consumption is.
Programs often run slowly when they are resource constrained, e.g. without enough memory or on a slow network connection.
There are a myriad of command line tools for probing and displaying different system resources like CPU usage, memory usage, network, disk usage and so on. -->

- **一般的なモニタリング** - おそらく最も有名なものは [`htop`](https://htop.dev/) でしょう。これは改良版の [`top`](https://www.man7.org/linux/man-pages/man1/top.1.html) です。 `htop` はシステム上で実行されているプロセスの様々な統計を表示します。 `htop` は無数のオプションとショートカットキーがあり、有名なものだと `<F6>` でプロセスをソートし、 `t` で階層構造のツリーを表示し、 `h` でスレッドをトグルします。
[`glances`](https://nicolargo.github.io/glances/) も素晴らしい UI を持つ同じようなツールです。すべてのプロセスをまとめた測定結果を得るには [`dstat`](http://dag.wiee.rs/home-made/dstat/) が気の利くツールであり、 I/O 、ネットワーク、 CPU 消費量、コンテキストスイッチといった様々なサブシステムのたくさんのリソースに関する情報をリアルタイムに計算することができます。
- **I/O 操作** - [`iotop`](https://www.man7.org/linux/man-pages/man8/iotop.8.html) は現在の I/O の使用量を表示することができ、プロセスが大量の I/O ディスク操作を行っていないか確認するのに便利です。
- **ディスク使用量** - [`df`](https://www.man7.org/linux/man-pages/man1/df.1.html) はパーティションごとの情報を表示し、 [`du`](http://man7.org/linux/man-pages/man1/du.1.html) はディスク （ **d**isk ）の使用量（ **u**sage ）を現在のディレクトリ内のファイルごとに表示します。これらのツールでは、 `-h` フラグを使うと人間（ **h**uman ）に読みやすい形式で表示することができます。
`du` のより対話的なバージョンとして、 [`ncdu`](https://dev.yorhel.nl/ncdu) というのもあり、これはフォルダーを移動しながら利用でき、ファイルやフォルダを削除することもできます。
- **メモリ使用量** - [`free`](https://www.man7.org/linux/man-pages/man1/free.1.html) はシステム内で使われているメモリーや利用できるメモリーの総量を表示します。メモリーは `htop` のようなツールにも表示されています。
- **使われているファイル** - [`lsof`](https://www.man7.org/linux/man-pages/man8/lsof.8.html) はプロセスによって開かれたファイルについての情報を列挙します。これは、どのプロセスが特定のあるファイルを開いたのか確認するのに大変便利です。
- **ネットワーク接続と設定** - [`ss`](https://www.man7.org/linux/man-pages/man8/ss.8.html) をつかうと、入ってきたり出ていったりするネットワークパケットやインターフェースに関する統計情報を監視することができます。 `ss` コマンドの一般的な利用法としては、マシンの特定のポートを使っているプロセスが何かを調べるというのがあります。ルーティングやネットワークデバイス、インターフェースを表示するには [`ip`](http://man7.org/linux/man-pages/man8/ip.8.html) コマンドが使えます。 `netstat` や `ifconfig` は、これらのツールがあるため非推奨となりました。
- **ネットワーク使用量** - [`nethogs`](https://github.com/raboof/nethogs) や [`iftop`](http://www.ex-parrot.com/pdw/iftop/) はネットワーク使用量を監視するための便利な対話的 CLI ツールです。

<!--
- **General Monitoring** - Probably the most popular is [`htop`](https://htop.dev/), which is an improved version of [`top`](https://www.man7.org/linux/man-pages/man1/top.1.html).
`htop` presents various statistics for the currently running processes on the system. `htop` has a myriad of options and keybinds, some useful ones  are: `<F6>` to sort processes, `t` to show tree hierarchy and `h` to toggle threads.
See also [`glances`](https://nicolargo.github.io/glances/) for similar implementation with a great UI. For getting aggregate measures across all processes, [`dstat`](http://dag.wiee.rs/home-made/dstat/) is another nifty tool that computes real-time resource metrics for lots of different subsystems like I/O, networking, CPU utilization, context switches, &c. -->
<!-- - **I/O operations** - [`iotop`](https://www.man7.org/linux/man-pages/man8/iotop.8.html) displays live I/O usage information and is handy to check if a process is doing heavy I/O disk operations -->
<!-- - **Disk Usage** - [`df`](https://www.man7.org/linux/man-pages/man1/df.1.html) displays metrics per partitions and [`du`](http://man7.org/linux/man-pages/man1/du.1.html) displays **d**isk **u**sage per file for the current directory. In these tools the `-h` flag tells the program to print with **h**uman readable format.
A more interactive version of `du` is [`ncdu`](https://dev.yorhel.nl/ncdu) which lets you navigate folders and delete files and folders as you navigate. -->
<!-- - **Memory Usage** - [`free`](https://www.man7.org/linux/man-pages/man1/free.1.html) displays the total amount of free and used memory in the system. Memory is also displayed in tools like `htop`. -->
<!-- - **Open Files** - [`lsof`](https://www.man7.org/linux/man-pages/man8/lsof.8.html)  lists file information about files opened by processes. It can be quite useful for checking which process has opened a specific file. -->
<!-- - **Network Connections and Config** - [`ss`](https://www.man7.org/linux/man-pages/man8/ss.8.html) lets you monitor incoming and outgoing network packets statistics as well as interface statistics. A common use case of `ss` is figuring out what process is using a given port in a machine. For displaying routing, network devices and interfaces you can use [`ip`](http://man7.org/linux/man-pages/man8/ip.8.html). Note that `netstat` and `ifconfig` have been deprecated in favor of the former tools respectively. -->
<!-- - **Network Usage** -  [`nethogs`](https://github.com/raboof/nethogs) and [`iftop`](http://www.ex-parrot.com/pdw/iftop/) are good interactive CLI tools for monitoring network usage. -->

もしこれらのツールをテストしたいなら、 [`stress`](https://linux.die.net/man/1/stress) コマンドを使うことでマシンに人工的な負荷をかけることができます。

<!-- If you want to test these tools you can also artificially impose loads on the machine using the [`stress`](https://linux.die.net/man/1/stress) command. -->


### 特化したツール
<!-- ### Specialized tools -->

ときには、どのソフトウェアを利用するか決めるために、ブラックボックス化されたベンチマークを取ることが必要だったりします。
[`hyperfine`](https://github.com/sharkdp/hyperfine) のようなツールを使うと、コマンドラインプログラムのベンチマークをすぐに取ることができます。
例えば、シェルツールとスクリプトの授業で `find` コマンドではなく `fd` コマンドを勧めました。 `hyperfine` をつかって、普段我々がよく実行するタスクについてこれらのツールを比べてみましょう。
たとえば、以下のサンプルでは `fd` は `find` より私のマシン上で20倍高速でした。

<!-- Sometimes, black box benchmarking is all you need to determine what software to use.
Tools like [`hyperfine`](https://github.com/sharkdp/hyperfine) let you quickly benchmark command line programs.
For instance, in the shell tools and scripting lecture we recommended `fd` over `find`. We can use `hyperfine` to compare them in tasks we run often.
E.g. in the example below `fd` was 20x faster than `find` in my machine. -->

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

<!-- As it was the case for debugging, browsers also come with a fantastic set of tools for profiling webpage loading, letting you figure out where time is being spent (loading, rendering, scripting, &c).
More info for [Firefox](https://developer.mozilla.org/en-US/docs/Mozilla/Performance/Profiling_with_the_Built-in_Profiler) and [Chrome](https://developers.google.com/web/tools/chrome-devtools/rendering-tools). -->

# 演習
<!-- # Exercises -->

## デバッグ
<!-- ## Debugging -->

1. Linux 上で `journalctl` 、もしくは macOS 上で `log show` をつかって最後にスーパーユーザーになって実行したコマンドを取得してください。
もし何もなければ、害のない `sudo ls` のようなコマンドを実行して再度確認してみてください。
<!-- 1. Use `journalctl` on Linux or `log show` on macOS to get the super user accesses and commands in the last day.
If there aren't any you can execute some harmless commands such as `sudo ls` and check again. -->

1. [この](https://github.com/spiside/pdb-tutorial) `pdb` のハンズオンチュートリアルを行い、コマンドに慣れてください。より詳細なチュートリアルとしては、[これ](https://realpython.com/python-debugging-pdb)を読んでみましょう。
<!-- 1. Do [this](https://github.com/spiside/pdb-tutorial) hands on `pdb` tutorial to familiarize yourself with the commands. For a more in depth tutorial read [this](https://realpython.com/python-debugging-pdb). -->

1. [`shellcheck`](https://www.shellcheck.net/) をインストールし、以下のスクリプトを確認してみてください。このコードの何が間違っているでしょうか？それを修正してみましょう。また、エディタにリンターのプラグインをインストールし、自動的に警告が出るようにしてみましょう。
<!-- 1. Install [`shellcheck`](https://www.shellcheck.net/) and try checking the following script. What is wrong with the code? Fix it. Install a linter plugin in your editor so you can get your warnings automatically. -->

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

<!-- 1. (Advanced) Read about [reversible debugging](https://undo.io/resources/reverse-debugging-whitepaper/) and get a simple example working using [`rr`](https://rr-project.org/) or [`RevPDB`](https://morepypy.blogspot.com/2016/07/reverse-debugging-for-python.html). -->
## プロファイリング
<!-- ## Profiling -->

1. [これ](/static/files/sorts.py) はいくつかのソートアルゴリズムを実装したものです。 [`cProfile`](https://docs.python.org/3/library/profile.html) と [`line_profiler`](https://github.com/pyutils/line_profiler) を使い、挿入ソートとクイックソートの実行時間を比べてみましょう。それぞれのアルゴリズムのボトルネックは何でしょうか。それから `memory_profiler` をつかってメモリー消費量を確認してみましょう。なぜ挿入ソートのほうが良いのでしょうか？それから、インプレースのバージョンのクイックソートを確認してみましょう。　チャレンジ： `perf` を使ってサイクルカウントとキャッシュヒット・ミスをそれぞれのアルゴリズムについて確認してみましょう。
<!-- 1. [Here](/static/files/sorts.py) are some sorting algorithm implementations. Use [`cProfile`](https://docs.python.org/3/library/profile.html) and [`line_profiler`](https://github.com/pyutils/line_profiler) to compare the runtime of insertion sort and quicksort. What is the bottleneck of each algorithm? Use then `memory_profiler` to check the memory consumption, why is insertion sort better? Check now the inplace version of quicksort. Challenge: Use `perf` to look at the cycle counts and cache hits and misses of each algorithm. -->

1. これは、それぞれの値ごとの関数をつかってフィボナッチ数を求める（あきらかに複雑な） Python のコードです。
<!--
1. Here's some (arguably convoluted) Python code for computing Fibonacci numbers using a function for each number. -->

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

   <!-- Put the code into a file and make it executable. Install prerequisites: [`pycallgraph`](http://pycallgraph.slowchop.com/en/master/) and [`graphviz`](http://graphviz.org/). (If you can run `dot`, you already have GraphViz.) Run the code as is with `pycallgraph graphviz -- ./fib.py` and check the `pycallgraph.png` file. How many times is `fib0` called?. We can do better than that by memoizing the functions. Uncomment the commented lines and regenerate the images. How many times are we calling each `fibN` function now? -->

1. リッスンしようとしているポートがすでに他のプロセスに取られていることはよくあります。そのプロセスの pid を見つける方法について学びましょう。最初に、 `python -m http.server 4444` を実行し、 `4444` ポートをリッスンしている最小のウェブサーバーを起動します。異なるターミナルで、 `lsof | grep LISTEN` を実行することですべてのリッスンしているプロセスやポートが表示されます。そのプロセスの pid を見つけ、 `kill <PID>` を実行することで終了させましょう。
<!-- 1. A common issue is that a port you want to listen on is already taken by another process. Let's learn how to discover that process pid. First execute `python -m http.server 4444` to start a minimal web server listening on port `4444`. On a separate terminal run `lsof | grep LISTEN` to print all listening processes and ports. Find that process pid and terminate it by running `kill <PID>`. -->

1. プロセスのリソースを制限するのは便利なツールとなるかもしれません。 `stress -c 3` を実行し、 `htop` で CPU 使用率を可視化してみましょう。さらに、 `taskset --cpu-list 0,2 stress -c 3` を実行して可視化してみましょう。 `stress` は3つの CPU を使っているでしょうか？なぜ使っていないのでしょう？ [`man taskset`](https://www.man7.org/linux/man-pages/man1/taskset.1.html) を読んでみてください。
チャレンジ：同じことを [`cgroups`](https://www.man7.org/linux/man-pages/man7/cgroups.7.html) を使ってやってみてください。メモリー消費量を `stress -m` をつかって制限してみてください。
<!-- 1. Limiting processes resources can be another handy tool in your toolbox.
Try running `stress -c 3` and visualize the CPU consumption with `htop`. Now, execute `taskset --cpu-list 0,2 stress -c 3` and visualize it. Is `stress` taking three CPUs? Why not? Read [`man taskset`](https://www.man7.org/linux/man-pages/man1/taskset.1.html).
Challenge: achieve the same using [`cgroups`](https://www.man7.org/linux/man-pages/man7/cgroups.7.html). Try limiting the memory consumption of `stress -m`. -->

1. （発展） `curl ipinfo.io` というコマンドは HTTP リクエストを投げ、あなたのパブリック IP に関する情報を取得します。 [Wireshark](https://www.wireshark.org/) を開き、 `curl` が送受信したリクエストや応答のパケットを監視してみましょう。（ヒント： `http` フィルターを使うと HTTP のパケットのみを見ることができます。）

<!-- 1. (Advanced) The command `curl ipinfo.io` performs a HTTP request and fetches information about your public IP. Open [Wireshark](https://www.wireshark.org/) and try to sniff the request and reply packets that `curl` sent and received. (Hint: Use the `http` filter to just watch HTTP packets). -->

