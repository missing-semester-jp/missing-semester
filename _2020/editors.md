---
layout: lecture
title: "エディタ (Vim)"
date: 2020-01-15
ready: true
video:
  aspect: 56.25
  id: a6Q8Na575qc
---

文章を書くこととプログラミングは全く異なる営みです。
英語のライティングがまとまった文章を書く作業であるのに対し、
プログラミングではファイルを切り替え、コードを読み、辿り、そして編集します。
そのため、英語という言語に適したエディタとプログラム言語に適したエディタがあるのです。（例: Microsoft Word と Visual Studio Code）

プログラマとして我々は大半の時間をコードの編集作業に費やすため、
ニーズに合ったエディタに習熟することは非常に時間対効果が高いといえます。
以下に新しいエディタの使い方を学ぶ際の一般的な方法論を紹介します。

- チュートリアルを始める（例: 本講座 + 紹介資料）
- （たとえ一時的に生産性を損ねても）あらゆる編集作業でそのエディタを使うことに拘る
- 常により良い方法がないかを模索する。もしもっと良い方法がないかという考えが頭をよぎったら、それは必ず存在します。

上記の方法に従い新たなプログラムをエディタの使い方を学ぶ題材にするならば、
タイムラインはこのようになるでしょう。
まず 1、2 時間以内にエディタの基礎機能（ファイルを開いて編集する、保存・終了する、バッファを操作する）
を習得します。
20 時間も経てば元のエディタと同じくらいの速度で編集できるようになっているでしょう。
新しいエディタに関する十分な知識を得て、使い方を身体で覚えたならば、
その恩恵を受け膨大な時間を節約することができるようになっていきます。
モダンなテキストエディタは素晴らしく強力なツールです。
あなたは学べば学ぶほどより速い編集能力を身につけることができるでしょう。

# どのエディタを学ぶべきですか？

プログラマはテキストエディタに関して[様々な選択肢](https://en.wikipedia.org/wiki/Editor_war)があります。
どのエディタが現在最も人気でしょうか？
[Stack Overflow
での調査](https://insights.stackoverflow.com/survey/2019/#development-environments-and-tools)
によると、
[Visual Studio Code](https://code.visualstudio.com/) が全体で最も人気があり、
[Vim](https://www.vim.org/) は CLI（コマンドラインインターフェース）のエディタの中で最も支持を集めています。
（ただし、Stack Overflow のユーザは全プログラマを代表するものではないため調査結果にはバイアスが含まれています。)

## Vim
本講座では全てのインストラクションに Vim を用います。
Vim には長い歴史があります。
Vim は元々 Vi（1976）として開発され、今日まで開発が継続されてきました。
Vim はいくつかの素晴らしいアイデアに基づいて設計されているため、
多くのツールが現在でも Vim の挙動をエミュレートするモードを採用しています
（VSCode においても、140 万のユーザが [Vim emulation for VS code](https://github.com/VSCodeVim/Vim) をインストールしています）。
つまり、たとえ他のエディタを最終的に使うことになるとしても、Vim は間違いなく学ぶ価値があるといえるでしょう。

Vim の機能全てを 50 分の講座で紹介し尽くすことは到底できません、
そこで本講座は Vim の哲学とその基礎、より高度な機能の一部を紹介するに留めようと思います。
講座の終わりでは Vim に更に習熟するための情報源をまとめて紹介します。

# Vim の哲学
プログラミングをする際、我々は殆どの時間をコードを書くことではなく、読むことと編集に費やします。
このため Vim は **モーダル** エディタとして設計されており、
テキストの挿入と編集にはそれぞれ異なるモードが割り当てられています。
また、Vim はプログラマブル（Vimscript や Python 等他のプログラミング言語で拡張機能を開発可能）
であり、覚えやすいキーストロークやコマンドは全て組み合わせて実行することができます。
Vim はマウスの使用を徹底して避けます。マウスでの操作はキーボードのそれと比較して余りにも遅いからです。
更に Vim は矢印キーの使用を避けます、これもキーボード上で必要な移動量が大きすぎるためです。
これらの思想を元に、思考のスピードで編集することが可能なエディタが完成しました。

# モーダル編集
多くのプログラマはまとまった長いテキストを書くよりも、
コードを読み、ファイルを辿り、そして僅かに編集を加える
という作業に大半の時間を費やしているという事実に着眼して Vim は設計されています。
このため、Vim は複数の操作モードを備えています。

- **ノーマルモード**：ファイル上を移動し、編集を加える
- **挿入モード**：テキストを挿入する
- **置換モード**：テキストを置換する
- **ビジュアルモード**：テキストをブロック選択する
- **コマンドラインモード**：コマンドを実行する

キーバインドは操作モード毎に異なる機能が割り当てられています。
挿入モードでの `x` は単に文字 x の挿入を行いますが、ノーマルモードではカーソル下の文字を消去し、
ビジュアルモードでは選択範囲を消去します。

初期設定では、Vim の画面左下に現在のモードが表示されます。
デフォルトのモードはノーマルモードです。
基本的には、ノーマルモードと挿入モードを行き来することになります。

`<ESC>` (Escキー) を押すと現在のモードからノーマルモードに切り替えることができます。
ノーマルモードからは、`i` で挿入モード、`R` で置換モード、`v` でビジュアルモード、`V` でビジュアルブロックモード、
`<C-v>`（Ctrl-V、`^V` とも表記）、コマンドラインモードに `:` で入ります。

Vim で編集する際は `<ESC>` を頻繁に使うため、
Caps Lock キーを Escape に置換することを検討してみてもいいでしょう
（[macOS
instructions](https://vim.fandom.com/wiki/Map_caps_lock_to_escape_in_macOS)）。

# 基本
## テキストを挿入する
ノーマルモードから、`i` を押下して挿入モードに入ります。
挿入モードでは、Vim は他のエディタと殆ど同じような挙動をしているでしょう。
そこで `<ESC>` を押すとまたノーマルモードに戻ってきます。
これが Vim によるファイル編集の全ての基本になります。
（挿入モードで大半の編集作業を行わず、速やかにノーマルモードに戻るようにして下さい。）

## バッファ、タブ、ウィンドウ
Vim ではメモリ上に読み込まれたファイルのことを”バッファ”と呼びます。
Vim のセッションはバッファの表示領域であるウィンドウと、そのウィンドウを集めたタブを管理しています。
ウェブブラウザ等他のアプリケーションとは異なり、Vim のバッファとウィンドウには 1 対 1 の対応関係があるわけではありません。
ウィンドウは単に表示領域に過ぎず、あるバッファの内容を同じタブ、複数のウィンドウに表示することもできます。
これは例えばあるファイルの 2 つの異なる領域を同時に表示したいという場合に役立ちます。

デフォルトでは、1 つのウィンドウを含む 1 つのタブを立ち上げます。

## コマンドライン
コマンドモードにはノーマルモードで `:` を入力して切り替えます。
画面下に入力領域が移り、`:` 以降にコマンドを入力することができます。
このモードはファイルを開く、保存する、閉じる、[Vimを終了する](https://twitter.com/iamdevloper/status/435555976687923200) といった多くの機能を担っています。

- `:q` 終了する（ウィンドウを閉じる）
- `:w` 保存する ("write")
- `:wq` 保存して閉じる
- `:e {ファイル名}` ファイルを開いて編集する
- `:ls` バッファの一覧を表示する
- `:help {topic}` ヘルプを開く
    - `:help :w` `:w` コマンドに関するヘルプを開く
    - `:help w` （ノーマルモードでの移動）`w` に関してヘルプを開く

# Vim のインタフェースはプログラム言語である
Vim における最も重要なアイデアは、Vimのインタフェース自体もプログラミング言語であるということです。
キー入力にはコマンドが割り当てられ、各コマンドは組み合わせることができます。
この仕様により、一度身体でコマンドを覚えれば、非常に効率的に移動や編集を行うことができるようになります。

## 移動
編集作業中は、
ノーマルモードにおいてバッファ上を移動することに多くの時間を使っているべきです。
Vim における移動は操作するテキストの断片を参照するため、"名詞"とも呼ばれます。

- 基本の移動： `hjkl` (左へ, 下へ, 上へ, 右へ)
- 単語： `w`（次の単語へ）、`b`（単語の先頭へ）、`e`（単語の末尾へ）
- 行： `0`（行の先頭へ）、`^`（行先頭の文字へ）`$`（行末へ）
- スクリーン： `H` (画面の先頭行へ) `M` (画面の中央行へ) `L` (画面の最終行へ)
- スクロール： `Ctrl-u` (上へ) `Ctrl-d` (下へ)
- ファイル： `gg` (ファイルの先頭へ) `G` (ファイルの末尾へ)
- 行番号： `:{number}<CR>` または `{number}G` (`{number}` 行目へ)
- その他： `%` （括弧等対応する要素へ）
- 文字への移動： `f{character}`、`t{character}`、`F{character}`、`T{character}`
  - 現在の行上で `{character}` を前 (f,t)  / 後 (F,T) に検索し、該当文字の上 (f,F) / 直前の文字 (t,T) に移動する
  - 現在の行において、次の該当文字上へ/該当文字の 1 文字前へ/前の該当文字へ 移動する
  - `,` / `;` マッチした文字を移動する
- 検索： `/{regex}`、`n` / `N` 正規表現で検索した文字列を前に/後に検索する

## 選択

ビジュアルモード：

- ビジュアル
- ビジュアルライン
- ビジュアルブロック

は移動キーを用いて選択を行うことができます。

## 編集

今までマウスで行っていた作業は、
編集コマンドと移動コマンドを組み合わせることで全てキーボードで行えるようになります。
Vim の編集コマンドは先で述べた名詞に対して機能させることができるため、”動詞”とも表現されます。
Vim のインタフェースがプログラミング言語のように感じられてきたでしょう。

- `i` 編集モードに入る
  - ただし、テキストを操作/消去する際は backspace よりノーマルモードを利用しましょう
- `o` / `O` 行を一行下に/一行上に挿入します
- `d{motion}` {motion} を消去する
  - 例: `dw` 文字を消去する、`d$` 行末まで文字を消去する、`d0` 行頭まで文字を消去する
- `c{motion}`  {motion} を変更する
  - 例: `cw` 単語を変更する（`d{motion}` + `i` と同等）
- `x` 文字を消去する（`dl` と同等）
- `s` 文字を置換する（`xi` と同等）
- ビジュアルモード + 操作
  - テキストを選択し、`d` で消去または `c` で変更する
- `u` 元に戻す、`<C-r>` やり直し
- `y` コピーする（`d` でも消去した内容のコピーを行います）
- `p` 貼り付け
- `~` 単語のキャピタライズを切り替える（大文字⇔小文字）

## カウント

名詞と動詞は数字カウントを付加して組み合わせることができます。
数字カウントによって、その回数分コマンドを繰り返すことができます。

- `3w` ３単語先に移動する
- `5j` ５行下に移動する
- `7dw` ７単語消去する

## 修飾子

名詞の機能を変更する修飾子を付加することができます。
例えば `i` は ”inner”　”inside”　を意味し、`a` は "around" を意味します。

- `ci(` 括弧内の内容を変更する
- `ci[` 角括弧内の内容を変更する
- `da'` シングルクオートで囲まれた文字を `'` を含めて消去する

# Demo

これは不完全な [fizz buzz](https://en.wikipedia.org/wiki/Fizz_buzz)
の実装です:

```python
def fizz_buzz(limit):
    for i in range(limit):
        if i % 3 == 0:
            print('fizz')
        if i % 5 == 0:
            print('fizz')
        if i % 3 and i % 5:
            print(i)

def main():
    fizz_buzz(10)
```

我々は以下の問題を修正しなければなりません。
- Main が呼ばれていない
- イテレーションが 1 ではなく 0 から始まっている
- 15 の倍数のとき、"fizz" "buzz" を別々の行に出力してしまう
- 5 の倍数のとき、"fizz" を出力してしまう
- コマンドライン引数ではなく、ハードコーディングされた入力(10) を使っている

{% comment %}
- main is never called
  - `G` end of file
  - `o` open new line below
  - type in "if __name__ ..." thing
- starts at 0 instead of 1
  - search for `/range`
  - `ww` to move forward 2 words
  - `i` to insert text, "1, "
  - `ea` to insert after limit, "+1"
- newline for "fizzbuzz"
  - `jj$i` to insert text at end of line
  - add ", end=''"
  - `jj.` to repeat for second print
  - `jjo` to open line below if
  - add "else: print()"
- fizz fizz
  - `ci'` to change fizz
- command-line argument
  - `ggO` to open above
  - "import sys"
  - `/10`
  - `ci(` to "int(sys.argv[1])"
{% endcomment %}

講義ビデオを視聴し、上記の変更が Vim を用いてどのように編集されるか、それが他のエディタを用いた場合といかに異なるかを確認してください。
Vim で必要なキータイプ数が非常に少なく、習熟すればまさしく思考のスピードで編集できるということが
実感できるでしょう。

# Vim のカスタマイズ
Vim は VimScript を含んだプレーンテキストの設定ファイル `~/.vimrc` によってカスタマイズすることができます。
Vim では有効化することが推奨される基本設定が多くあります。

そこで我々は入門用として、（ドキュメンテーションされた）基本のコンフィグファイルを用意しました。
やや扱いづらいデフォルトの挙動を修正するこのコンフィグをまず利用することをおすすめします。
**[ここ](/2020/files/vimrc)からファイルをダウンロードし、`~/.vimrc` に保存してください。**

Vim は隅々までカスタマイズが可能で、カスタマイズオプションを調査しつくす価値は大いに有ります。
GitHub 上で他のユーザの dotfiles を参考にしてもいいでしょう。
例えばインストラクターのコンフィグも GitHub に公開しています
([Anish](https://github.com/anishathalye/dotfiles/blob/master/vimrc),
[Jon](https://github.com/jonhoo/configs/blob/master/editor/.config/nvim/init.vim) ([neovim](https://neovim.io/) ユーザ),
[Jose](https://github.com/JJGO/dotfiles/blob/master/vim/.vimrc))。
また、ネット上には Vim の設定について多くのブログ記事が公開されています。
ただし、単に他のユーザの設定をコピーペーストするのではなく、熟読し、内容を理解し、必要な設定だけを取り入れるようにしましょう。

# Vim を拡張する

Vim では非常に多くの拡張プラグインが開発されています。
まずプラグインマネージャを導入する、という解説記事を多く目にするかもしれませんが、
Vim 8.0 以降では必ずしもプラグインマネージャを使用する必要はありません。
その代わりに、Vim 組み込みのパッケージ管理システムを利用することができます。
新規にディレクトリ `~/.vim/pack/vendor/start/` を作成し、このディレクトリにプラグインを（`git clone` 等で）
配置してください。

我々が愛用しているプラグインの一部を紹介します。

- [ctrlp.vim](https://github.com/ctrlpvim/ctrlp.vim): fuzzy file finder
- [ack.vim](https://github.com/mileszs/ack.vim): code search
- [nerdtree](https://github.com/scrooloose/nerdtree): file explorer
- [vim-easymotion](https://github.com/easymotion/vim-easymotion): magic motions

プラグインの膨大なリストをここで紹介するのは敢えて避けたいと思います。
インストラクタの dotfiles をまず参照し
([Anish](https://github.com/anishathalye/dotfiles),
[Jon](https://github.com/jonhoo/configs),
[Jose](https://github.com/JJGO/dotfiles)) どんなプラグインを我々が使っているか見てみて下さい。
[Vim Awesome](https://vimawesome.com/) では更に多くのプラグインを発見できます。
"best Vim plugins" で検索すれば、優れたプラグインについて多くの記事が見つかるはずです。

# ソフトウェアにおける Vim モード

多くのツールが Vim のエミュレーションモードをサポートしています。
その質はツールによって様々ですが、
素晴らしい機能の全てとはいかないまでも基本的な機能は大抵網羅されています。

## シェル

もし Bash を使っているならば、 `set -o vi` を、`Zsh` を使っているなら `bindkey -v` を、
`Fish` を使っているなら `fish_vi_key_bindings` で vi モードを使用してみましょう。
加えて、どのシェルを使っていようとも、 `export EDITOR=vim` を環境変数に設定しましょう。
これはあらゆるプログラムがエディタを呼び出す際に、どのエディタを使うかを定める環境変数です。
例えば、 `git` はコミットメッセージを入力する際この環境変数で設定されているエディタをデフォルトで起動します。

## Readline
多くのプログラムはコマンドラインインタフェースのためのライブラリ [GNU
Readline](https://tiswww.case.edu/php/chet/readline/rltop.html)
を使っています。Readline は基本的な vi キーバインドをサポートしています。
`~/.inputrc` ファイルに以下の行を追加すると、vi モードを有効化できます。

```
set editing-mode vi
```

この設定を有効化すると、例えば Python の REPL も Vim キーバインドをサポートします。

## その他

[Web ブラウザ](http://vim.wikia.com/wiki/Vim_key_bindings_for_web_browsers) にも Vim のキーバインドをエミュレートする拡張機能があります。
有名なものでは Google Chrome 用の
[Vimium](https://chrome.google.com/webstore/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb?hl=en)
や Firefox 用の [Tridactyl](https://github.com/tridactyl/tridactyl) があります。
[Jupyter
notebooks](https://github.com/lambdalisue/jupyter-vim-binding) でも Vim のキーバインドが提供されています。

# 発展的な Tips

このエディタの真価の一端をいくつか紹介します。
全てを教えつくすことはできませんが、Vim を使用するにつれ自然と他のテクニックについても習熟していくでしょう。
Vim での編集作業中、もっと良い方法があるのではという考えが頭をよぎったら、
必ずその方法は存在します。ネットで検索して常に最適な方法を模索する癖を付けましょう。

## 検索と置換

`:s` (substitute) コマンド ([documentation](http://vim.wikia.com/wiki/Search_and_replace)).

- `%s/foo/bar/g`
    - ドキュメント中全ての foo を bar に置換する
- `%s/\[.*\](\(.*\))/\1/g`
    - Markdown のリンクテキストをリンク内の URL に置換する

## 複数ウィンドウ
- `:sp` / `:vsp` ウィンドウを分割する
- 同じバッファに対して複数のビューを持つこともできます。

## マクロ

- `q{character}` レジスタ {character} へマクロのレコーディングを開始
- `q` レコーディングを停止
- `@{character}` レジスタ {character} の内容を実行する
- `{number}@{character}` マクロを {number} 回繰り返す
- マクロは再帰的にも実行される
  - まず `q{character}q` でマクロをクリアする
  - マクロを記録し、`@{character}` でマクロを実行する（レコーディングが完了するまで何もしない）
- 例: xml を json にする ([ファイル](/2020/files/example-data.xml))
  - "name" / "email" キーを持つオブジェクトの配列
  - Python のプログラムを使うか？
  - sed / 正規表現を使うか？
      - `g/people/d`
      - `%s/<person>/{/g`
      - `%s/<name>\(.*\)<\/name>/"name": "\1",/g`
      - ...
  - Vim コマンド / マクロを使うか？
      - `Gdd`, `ggdd` 最初と最後の行を削除する
      - 1 つの要素を整形するためのマクロをレジスタ `e` に登録する
        - `<name>` の行に移動
        - `qe^r"f>s": "<ESC>f<C"<ESC>q`
      - `<person>` を整形する
        - `<person>` の行に移動
        - `qpS{<ESC>j@eA,<ESC>j@ejS},<ESC>q`
      - `<person>` を整形し次の `<person>` へ移動
        - `<person>` の行に移動
        - `qq@pjq`
      - マクロを最終行まで実行する
        - `999@q`
      - 手動で最後の `,` を消去し、 `[` `]` で全要素を囲う

# 参考

- `vimtutor` はVimと共にインストールされるチュートリアルです。もしVimが既にインストールされている環境ならば、シェルで `vimtutor` を実行することができます。
- [Vim Adventures](https://vim-adventures.com/) は Vim を学ぶことができるゲームです。
- [Vim Tips Wiki](http://vim.wikia.com/wiki/Vim_Tips_Wiki)
- [Vim Advent Calendar](https://vimways.org/2019/) では様々な Vim Tips が紹介されています。
- [Vim Golf](http://www.vimgolf.com/) は Vimの[コードゴルフ](https://en.wikipedia.org/wiki/Code_golf)場です。
- [Vi/Vim Stack Exchange](https://vi.stackexchange.com/)
- [Vim Screencasts](http://vimcasts.org/)
- [Practical Vim](https://pragprog.com/titles/dnvim2/) (book)

# 演習
1. `vimtutor` を完了させましょう。[80x24](https://en.wikipedia.org/wiki/VT100) (80 columns by 24 lines) のターミナルで行うのが良いでしょう。
1. [基本の vimrc](/2020/files/vimrc) を`~/.vimrc` に保存し、コメント含めファイル全体を（**Vim を使って**）読み、設定ファイルによって Vim の挙動がどのように変化するかを確認して下さい。
1. プラグインのインストール、設定を行いましょう。
   1. プラグインのディレクトリを作成 `mkdir -p ~/.vim/pack/vendor/start`
   1. プラグインのダウンロード `cd ~/.vim/pack/vendor/start; gitclone https://github.com/ctrlpvim/ctrlp.vim`
   1. [プラグインのドキュメント](https://github.com/ctrlpvim/ctrlp.vim/blob/master/readme.md)
      を読みます。まず CtrlP を使ってファイルを見つけてみましょう。プロジェクトディレクトリに移動し、Vimを開いて `:CtrlP` を入力します。
   1. [configuration](https://github.com/ctrlpvim/ctrlp.vim/blob/master/readme.md#basic-options) を参考に、`~/.vimrc` へ設定を追加してみましょう。`Ctrl-P` で CtrlP を開けるようにします。
1. [Demo](#demo) をあなたのマシンで復習しましょう。
1. 次の月から**全ての**テキスト編集にVimを使います。何かが非行率または "もっと良い方法があるはずだ" と感じる度にベストな方法を検索する癖を付けましょう。
1. 他のツールを Vim のキーバインディングを使用するように設定してみましょう。
1. `~/.vimrc` を更にカスタマイズしましょう。好みのプラグインを入れてみてもいいでしょう。
1. （発展）XMLをJSON（[example file](/2020/files/example-data.xml)）に Vim マクロを使って変換してみましょう。躓いたら、[マクロ](#マクロ) のセクションがヒントになるはずです。
