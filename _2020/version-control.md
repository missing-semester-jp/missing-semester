---
layout: lecture
title: "バージョン管理 (Git)"
date: 2020-01-22
ready: true
video:
  aspect: 56.25
  id: 2sjqTHE0zok
---

バージョン管理システム（VCS）はソースコード（や、ファイルやフォルダーのセット）に加えられた変更を追跡するためのツールです。
名前が示している通りこれらのツールは変更履歴の保存に役立ち、さらに共同作業を円滑にしてくれます。
バージョン管理システムは一連のスナップショットによってフォルダーやその中身に加えられた変更を追跡します。
そこで、それぞれのスナップショットはファイル、フォルダ内の全ての記述を一番上のディレクトリに内包しています。
また、バージョン管理システムはそれぞれのスナップショットの作成者、スナップショットに関わるメッセージ、などのメタデータも保存してくれます。

なぜバージョン管理は便利なのでしょうか？
あなたが一人で作業しているときでさえバージョン管理をしていると、プロジェクトの古いスナップショットを確認することができ、その変更が行われた理由の記録を残したり、複数のブランチで開発作業を並行して行ったり、といったことをはじめとしてさらに沢山のことができるようになります。
他の作業者とともに作業するときには、他の人が行った変更を確認するとともに同時に作業することによって生じるコンフリクト（衝突）を解決するために非常に貴重なツールです。

また、現代のバージョン管理システムは以下のような疑問にも簡単に（しばしば自動的に）答えを与えてくれます：

- 誰がこのモジュールを書いたのか？
- いつ、誰によって、なぜこの特定のファイル内のこの特定の行が編集されたのか？
- 過去１０００回の修正において、いつ/なぜ特定のユニットテストが動作停止してしまったのか？

他のバージョン管理システムは存在しますが、**Git** はバージョン管理のデファクトスタンダード（事実上の標準）です。
この [XKCD comic](https://xkcd.com/1597/) はGitの評判をよくとらえています。

![xkcd 1597](https://imgs.xkcd.com/comics/git.png)
「これはGitだよ。Git では共同作業の履歴を追跡することができるんだ。これを使って、美しい分散グラフ理論的木構造を利用したプロジェクトで作業しよう。」

「いいね、どうやって使うの？」

「わかんない。ただ覚えるんだよ。このシェルコマンドを打って、それを同期させるんだ。エラーが出たらどこか他のところにセーブして、エラーが出たプロジェクトを消去して、新しいコピーをダウンロードするんだ。」


Gitのインターフェースは漏りやすく抽象的なので、Gitを（そのインターフェース、つまりコマンドラインインターフェースを皮切りにして）全体から細部まで理解しようとすると、多くの混乱を招いてしまうことがあります。
一握りのコマンドを暗記してしまい、それらを魔法の呪文のようなものと捉え、何かうまくいかないことがあれば上のイラストのような手法に従ってみることは可能です。


確かにGitのインターフェースはあまり使いやすいものではありませんが、その根底にあるデザインとアイディアは美しいものです。
使いにくいインターフェースでは _暗記_ をしなければならない一方で、美しいデザインは _理解_ することができます。
なので、これからGitをまずはデータモデル（データ型）から、そして後からコマンドラインインターフェースを網羅することで、細部から全般に至るような説明をしていきます。
一度データモデルを理解すれば、基礎となるデータモデルをどのように操作するかという点においてコマンドをよく理解できるようになります。


# Gitのデータモデル

バージョン管理のためにあなたが取ることができるアドホックな（特定の目的のための）手法は、たくさんあります。
Gitは、履歴の保有、ブランチの保持や共同作業の有効化などといったバージョン管理の全ての優れた機能を使用可能にするように考え抜かれたモデルになっています。


## スナップショット

Gitはいくつかの最高レベルのディレクトリの中に、一連のスナップショットとしてファイルやフォルダーの蓄積の履歴をモデル化します。
Gitの専門用語では、ファイルは『ブロッブ（blob）』と呼ばれ、大量のバイト（byte）に過ぎません。
ディレクトリは『ツリー（tree）』と呼ばれ、名前をブロッブやツリーに対応づけます（そのためディレクトリが他のディレクトリを内包することができます）。
スナップショットは追跡される最高レベルのツリーです。
例えば、以下のようなツリーになることがあります。

```
<root（根）> (ツリー)
|
+- foo (ツリー)
|  |
|  + bar.txt (ブロッブ, 内容 = "hello world")
|
+- baz.txt (ブロッブ, 内容 = "git is wonderful")
```

最高レベルのツリーは二つの要素を含んでいます。
ひとつは『foo』というツリー（それ自体は『bar.txt』というブロッブを一つの要素として含んでいます）であり、もうひとつは『baz.txt』というブロッブです。

## スナップショットに関連する、履歴のモデル化

バージョン管理システムはどのようにスナップショットを関連付けているのでしょうか？
一つの単純なモデルとしては、線形の履歴を持つものがあります。
履歴はスナップショットを時系列順に並べたリストになります。
しかし様々な理由で、Gitはこのような単純なモデルを使用していません。

Gitでは、履歴はスナップショットの有向非巡回グラフ（DAG）になっています。
この有向非巡回グラフは派手な数学用語のように聞こえるかもしれませんが、恐れることはありません。
この言葉が意味することは、Git内のそれぞれのスナップショットが『ペアレント（親）』のセット、つまり先行するスナップショットを参照しているということです。
これは（線形の履歴の場合のような）単一のペアレントではなくペアレントのセットです、なぜならスナップショットは複数のペアレントから受け継がれることがあるからです。
たとえば、二つの並行な開発のブランチを統合（マージ）した場合がこれにあたります。

Gitはこのようなスナップショットのことを『コミット（commit）』と呼びます。
コミット履歴を視覚化すると以下のようになることがあります。

```
o <-- o <-- o <-- o
            ^
             \
              --- o <-- o
```

上のアスキーアートでは、 `o`が個々のコミット（スナップショット）に対応しています。
矢印はそれぞれのコミットのペアレントを指しています（これは「前に来る」関係であり、「後に来る」関係ではありません）。
三個目のコミットの後、履歴ブランチが二つのブランチに分かれています。
これは例えば、それぞれから独立して並行に開発された二つの別々の機能に対応しています。
将来的にこれらのブランチはどちらもの機能が組み込まれた、新しいスナップショットへとマージ（融合）され、以下の図においてボールド体で示された新しく生成されたマージコミットとともに、新たな履歴を作ることがあります。

<pre class="highlight">
<code>
o <-- o <-- o <-- o <---- <strong>o</strong>
            ^            /
             \          v
              --- o <-- o
</code>
</pre>

Gitの中のコミットは不変です。
これは誤りを修正することができないという意味ではありません。
しかし、コミット履歴への「編集」は実際にはまったく新しいコミットを作成しているだけであり、参照（以下を見てください）は新しいものを指し示すようにアップデートされます。

## 疑似コードとしてのデータモデル

疑似コードで書かれたGitのデータモデルを確認することは役に立つかもしれません。

```
// ファイルは大量のバイト（byte）です。
type blob = array<byte>

// ディレクトリは名前付きファイルとディレクトリを内包しています。
type tree = map<string, tree | blob>

// コミットはペアレント、メタデータ、最高レベルのツリーを保持しています。
type commit = struct {
    parent: array<commit>
    author: string
    message: string
    snapshot: tree
}
```
これはきれいで単純な履歴のモデルです。

## オブジェクトと内容アドレシング（コンテンツのアドレス指定）

「オブジェクト」とはブロッブ、ツリーまたはコミットのことです。

```
type object = blob | tree | commit
```

Gitのデータの記憶装置では、全てのオブジェクトが [SHA-1
hash](https://en.wikipedia.org/wiki/SHA-1) で内容のアドレスを指定されています。

```
objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]
```

ブロッブ、ツリーそしてコミットは次のように統一されています。
それらは全てオブジェクトです。
それらが他のオブジェクトを参照したときには、それらは実際にはオブジェクトをそれらのディスク上の表示に _内包_ しているわけではなく、ハッシュ（hash）によって参照を行なっています。

例えば、 [上](#スナップショット)の例のディレクトリ構造のツリーは、（`git cat-file -p 698281bc680d1995c5f4caaf3359721a5a58d48d` を使用して視覚化すると）以下のようになります。

```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85    baz.txt
040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87    foo
```

ツリーそれ自体は、 `baz.txt`（ブロッブ）と`foo`（ツリー）というコンテンツへのポインタを持っています。
`git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85`を使用してbaz.txtに対応するハッシュでアドレス指定されたコンテンツを見てみると、以下の文章が得られます。

```
git is wonderful
```

## リファレンス

以上のように、全てのスナップショットはそれらのSHA-1ハッシュによって識別されます。
しかし、40文字の16進数文字は人間が記憶するには適していないため、不便です。

この問題に対するGitの解決策は『リファレンス（参照）』と呼ばれる、人間にも読み取り可能なSHA-1ハッシュの名称です。
リファレンスはコミットへのポインタになっています。
オブジェクトは不変ですが、そのオブジェクトとは異なりリファレンスは可変のものです。（新しいコミットを指し示すためにアップデートされることがあります。）
例えば、  `マスター（master）`というリファレンス はたいていメインの開発ブランチ内における最新のコミットを指し示しています。

```
references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)
```

このようにGitは『マスター』のような人間にも読み取り可能な名称を、長い16進数の文字列の代わりとして、履歴内にある特定のスナップショットを参照するために使用することができます。

一つの詳細は、私たちは新たなスナップショットを取る際にそのスナップショットが何と関連しているのか（コミットの`ペアレント`の領域を設定方法）を知るために、「現在の作業場所」という概念が必要なことがよくあります。
Gitでは、「現在の作業場所」は『ヘッド（HEAD）』という特別なリファレンスで呼ばれます。

## レポジトリ

ようやく、Git _リポジトリ（repository）_ が何なのかを定義することができます。
リポジトリとはデータの`オブジェクト`と `リファレンス`のことなのです。

ディスク上において、全てのGitの記憶装置はオブジェクトとリファレンスです。
つまり、Gitのデータモデルにあるのはそれだけです。
全ての`git`コマンドはいくつかのコミット有向非巡回グラフ（DAG）への操作を、オブジェクトの追加やリファレンスの追加・更新によって図示します。

あなたがどんなコマンドを打ち込んでいるときでも、根底にあるグラフのデータ構造にそのコマンドがどのような操作を行なっているのかを考えるようにしましょう。
一方で、コミット有向非巡回グラフ（DAG）に対して特定の種類の変更、例えば『コミットされていない変更を破棄して、「マスター」リファレンスに`5d83f9e`というコミットを指し示させる』といったことを行おうとするならば、それを行うことができるコマンドがおそらくあるでしょう。
（例えばこの例の場合であれば、そのコマンドは`git checkout master; git reset
--hard 5d83f9e`というようになります。）

# ステージングエリア

これはデータモデルと直行する、別のコンセプトですが、コミット作成インターフェースの一部です。


上記のようにスナップショットを実装する際に考えることができる一つの方法は、『スナップショットを作成する』というコマンドによって作業中のディレクトリの _現在の状態_ に基づき、新しいスナップショットを作成するというものです。
いくつかのバージョン管理ツールはこのような手法を取っていますが、Gitは違います。
私たちはスナップショットを整理しておきたいと望みますが、現在の状態からスナップショットを作成することが常に理想的であるとは限りません。
例えば、次のような状況を想像してみてください。
あなたは二つ別々の機能を実装して、二つの別々のコミットを作成したいと考えます。
そしてその二つのコミットのうち一つには一つ目の機能を導入し、別のものには二つ目の機能を導入したいです。
または、次のような状況を想像してみましょう。
あなたは、あなたのコード全体にわたってバグ修正に伴って追加されたprintステートメントを、デバックします。
つまり、あなたは全てのprintステートメントを破棄しながら、バグ修正をコミットしたいのです。

Gitは『ステージングエリア（staging area）』という仕組みを通し、どの修正が次のスナップショットに含まれているべきなのかを細かく記述できるようにすることで、これらの状況にも対応しています。

# Gitのコマンドラインインターフェース

同じ情報を繰り返し述べてしまうことを避けるために、以下のコマンドを細かく説明はしません。
さらに情報を得るために[Pro Git](https://git-scm.com/book/en/v2)を確認することを強くお勧めします。
または講義ビデオを視聴してみてください。

## 基本的なもの

{% comment %}

`git init`というコマンドは`.git`ディレクトリ内に保管されているレポジトリのメタデータとともにGitのリポジトリを初期化します。

```console
$ mkdir myproject
$ cd myproject
$ git init
Initialized empty Git repository in /home/missing-semester/myproject/.git/
$ git status
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

この出力をどのように解釈すればいいでしょうか？
"No commits yet" は要するに、私たちのバージョン履歴が空であるということを意味しています。
では、それを直してみましょう。

```console
$ echo "hello, git" > hello.txt
$ git add hello.txt
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

        new file:   hello.txt

$ git commit -m 'Initial commit'
[master (root-commit) 4515d17] Initial commit
 1 file changed, 1 insertion(+)
 create mode 100644 hello.txt
```

これによってファイルをステージングエリアに`git add`し、そして変更を `git
commit`し、"Initial commit（最初のコミット）"という簡単なコミットメッセージを追加しました。
`-m` オプションを記述しなかった場合にはGitがテキストエディタを開いてくれるので、そこでコミットメッセージを打ち込むことができます。

現在、私たちのバージョン履歴は空ではないので、履歴を可視化することができます。
有向非巡回グラフ（DAG）としての履歴の可視化は、レポ（repo）の現在の状態を理解することやそれをあなたのGitのデータモデルの理解に繋げることに特に役立つことがあります。

`git log`は履歴を可視化するコマンドです。
他に選択の余地がないので、それは平になったバージョンを示し、グラフの構造を隠してしまいます。
`git log
--all --graph --decorate`のようなコマンドを使用すると、グラフの形に視覚化されたフルバージョンのレポジトリ履歴を確認することができます。

```console
$ git log --all --graph --decorate
* commit 4515d17a167bdef0a91ee7d50d75b12c9c2652aa (HEAD -> master)
  Author: Missing Semester <missing-semester@mit.edu>
  Date:   Tue Jan 21 22:18:36 2020 -0500

      Initial commit
```

これはグラフのように見えません、なぜなら一つのノードしか含んでいないからです。
さらにいくつかの変更をし、新しいコミットを書いてもう一度履歴を可視化してみましょう。

```console
$ echo "another line" >> hello.txt
$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   hello.txt

no changes added to commit (use "git add" and/or "git commit -a")
$ git add hello.txt
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        modified:   hello.txt

$ git commit -m 'Add a line'
[master 35f60a8] Add a line
 1 file changed, 1 insertion(+)
```

ここで再度履歴を可視化すると、いくつかのグラフ構造を確認することができます。

```
* commit 35f60a825be0106036dd2fbc7657598eb7b04c67 (HEAD -> master)
| Author: Missing Semester <missing-semester@mit.edu>
| Date:   Tue Jan 21 22:26:20 2020 -0500
|
|     Add a line
|
* commit 4515d17a167bdef0a91ee7d50d75b12c9c2652aa
  Author: Anish Athalye <me@anishathalye.com>
  Date:   Tue Jan 21 22:18:36 2020 -0500

      Initial commit
```

また、現在のブランチ（マスター）に加えて現在のHEADも示されることを覚えておきましょう。

古いバージョンは`git checkout` というコマンドを使用すると閲覧することができます。

```console
$ git checkout 4515d17  # previous commit hash; yours will be different
Note: checking out '4515d17'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 4515d17 Initial commit
$ cat hello.txt
hello, git
$ git checkout master
Previous HEAD position was 4515d17 Initial commit
Switched to branch 'master'
$ cat hello.txt
hello, git
another line
```

`git
diff` というコマンドを使用すると、ファイルがどのように発展したのか（差異や diff ）を確認することができます。

```console
$ git diff 4515d17 hello.txt
diff --git c/hello.txt w/hello.txt
index 94bab17..f0013b2 100644
--- c/hello.txt
+++ w/hello.txt
@@ -1 +1,2 @@
 hello, git
 +another line
```

{% endcomment %}

- `git help <command>`: Gitコマンドについての情報を得る
- `git init`:  `.git`ディレクトリ内に保存されたデータとともに、新しいGitレポを作成する
- `git status`: 何が起こっているのかを教えてくれる
- `git add <filename>`: ステージングエリアにファイルを追加する
- `git commit`: 新しいコミットを作成する
    -  [良いコミットメッセージ](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)を書きましょう！
    - [良いコミットメッセージ](https://chris.beams.io/posts/git-commit/)を書かなければいけないさらに多くの理由
- `git log`: 平らな履歴のログを示す
- `git log --all --graph --decorate`: 有向非巡回グラフ（DAG）として履歴を可視化する
- `git diff <filename>`:  ステージングアリアに関して行なった変更を表示する
- `git diff <revision> <filename>`: スナップショット間の、ファイル内の差異を表示する
- `git checkout <revision>`: HEADと現在のブランチを更新する

## ブランチング（branching）とマージング（merging）

{% comment %}

ブランチングによって、バージョン履歴を分岐（fork）させることができます。
これは並行して独立の機能を開発するときやバグ修正を行うときに便利です。
`git branch` というコマンドは新たなブランチを作るときに使用され、 `git checkout -b <branch name>`というコマンドは新たなブランチを作り、分岐させ、確認するときに使用されます。

マージングはブランチングの反対の操作です。
つまり、マージングによって分岐したバージョン履歴を結合することができます。
例えば、分岐して機能を持たせたブランチをマージ（結合）して、マスターに戻すという操作がこれにあたります。
このマージングには `git merge`というコマンドが使用されます。

{% endcomment %}

- `git branch`: ブランチを表示する
- `git branch <name>`: ブランチを作る
- `git checkout -b <name>`: ブランチを作り、そのブランチに切り替える
    -  `git branch <name>; git checkout <name>`と同様
- `git merge <revision>`: 現在のブランチにマージする
- `git mergetool`: マージコンフリクト（結合衝突）を解決するために便利なツールを使用する
- `git rebase`: パッチのセットを新しいベース（土台）の上にリベース（作業が完了したブランチを分岐元のブランチにくっつける）する

## リモート

- `git remote`: リモートをリスト化する
- `git remote add <name> <url>`: リモートを追加する
- `git push <remote> <local branch>:<remote branch>`: リモートにオブジェクトを送信し、リモートのリファレンスを更新する
- `git branch --set-upstream-to=<remote>/<remote branch>`: ローカルブランチとリモートブランチ間の通信を設定する
- `git fetch`: リモートからオブジェクトとリファレンスを回収する
- `git pull`:  `git fetch; git merge`と同様
- `git clone`: リモートからリポジトリをダウンロード（クローン）する

## 取り消し（Undo）

- `git commit --amend`: コミットの内容やメッセージを編集する
- `git reset HEAD <file>`: ファイルのステージングを解除する
- `git checkout -- <file>`: 変更を破棄する

# Git上級者向けコマンド

- `git config`: Gitを[もっとカスタマイズする](https://git-scm.com/docs/git-config)
- `git clone --depth=1`: バージョン履歴全体は持ってこない、浅いクローン（ダウンロード）
- `git add -p`: 対話的なステージング
- `git rebase -i`: 対話的なリベース
- `git blame`: どの行を誰が最後に編集したのかを表示する
- `git stash`: 作業ディレクトリへの変更を一時的に削除する
- `git bisect`: 履歴のバイナリサーチ（二分探索）を行う（回帰など）
- `.gitignore`: 意図的に追跡されていないファイルを、無視するために[特定する](https://git-scm.com/docs/gitignore)

# その他

- **グラフィカル・ユーザ・インターフェース（GUI）**: Gitにはたくさんの [GUI クライアント](https://git-scm.com/downloads/guis)があります。私たちとしては、コマンドラインの方をGUIのかわりに使用します。
- **シェルの統合**: あなたのシェルプロンプト（シェルコマンドで受付状態にあるとき表示される記号）の一部として、Gitの状態を保有することはとても役に立ちます （[zsh](https://github.com/olivierverdier/zsh-git-prompt)、
[bash](https://github.com/magicmonty/bash-git-prompt)） 。
[Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh)（zsh用の設定フレームワーク）のようなフレームワークに含まれていることがよくあります。
- **エディターの統合**: 上記のものに似ており、たくさんの機能を手軽に統合します。
[fugitive.vim](https://github.com/tpope/vim-fugitive)はVim（viから派生した高機能テキストエディタ）用の標準的なものです。
- **ワークフロー（作業工程）**: ここまででデータモデルと基本的なコマンドについて説明しましたが、大きなプロジェクトで作業をする際の慣例についてはお話していません。
（しかし、大きなプロジェクトで作業するときには小さなプロジェクトとは[異なる](https://www.endoflineblog.com/gitflow-considered-harmful)[アプローチが](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)[たくさん](https://nvie.com/posts/a-successful-git-branching-model/)あります。）
- **GitHub**: GitはGitHubとは違います。
GitHubには他のプロジェクトへコードを提供するための、[プルリクエスト（pull request）](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)という独特な方法があります。
- **その他のGitプロバイダ**: GitHubは特別なものではありません。
他にも [GitLab](https://about.gitlab.com/) や
[BitBucket](https://bitbucket.org/)といった、たくさんのGitリポジトリホストがあります。

# 追加教材、資料

- [Pro Git](https://git-scm.com/book/en/v2)は**とてもオススメの読み物です**。
第１章から第５章まで目を通すと、Gitを上手に使うために必要なもののほとんどがわかるはずです。
これで、データモデルを理解できます。
その後の章ではいくつかの面白く、上級者向けの題材について述べられています。
- [Oh Shit, Git!?!](https://ohshitgit.com/) はGitを使う際によくある間違いを解決する方法について書かれた、短めのガイドです。
- [Git for Computer
Scientists](https://eagain.net/articles/git-for-computer-scientists/) は疑似コードとこの講義ノートよりもわかりやすい図が付いている、Gitのデータモデルについての短めな解説です。
- [Git from the Bottom Up](https://jwiegley.github.io/git-from-the-bottom-up/) は好奇心旺盛な人のための、データモデルだけに限らないGit実装の詳細についての細部にまでわたる解説です。
- [How to explain git in simple
words](https://smusamashah.github.io/blog/2017/10/14/explain-git-in-simple-words)
- [Learn Git Branching](https://learngitbranching.js.org/) はブラウザでプレイ可能なGitについて学ぶことができるゲームです。

# 演習問題

1.   もし過去にGitを一度も使ったことがなければ、 [Pro Git](https://git-scm.com/book/en/v2) の最初の２、３章を読むか、[Learn Git Branching](https://learngitbranching.js.org/)のようなチュートリアルに取り組んでみましょう。
それらに取り組みながら、Gitのコマンドをデータモデルと関連づけましょう。
1.  [講義サイト用のレポジトリ](https://github.com/missing-semester/missing-semester)をクローン（ダウンロード）しましょう。
    1. グラフとしてバージョン履歴を可視化して、細かく見てみましょう。
    1. `README.md`を最後に編集したのは誰でしたか？（ヒント：引数をつけて、`git log` を使用しましょう。）
    1. `_config.yml`の`collections:`行に最後に行われた変更に付けられたコミットメッセージは何でしょう？（ヒント： `git blame`と`git show`を使ってみましょう。）
1. Gitを学んでいるときによくある間違いは、Gitで管理すべきでないほど大きなファイルをコミットしてしまうことや、細心の注意を要するような情報をgit上に追加してしまうことです。レポジトリにファイルを追加する、いくつかのコミットをする、履歴からそのファイルを削除するという作業を試してみましょう。（[これ](https://help.github.com/articles/removing-sensitive-data-from-a-repository/)を見ると良いかもしれません。）
1. GitHubからいくつかのレポジトリをクローン（ダウンロード）し、存在しているファイルのうち一つに変更を加えてみましょう。 `git stash`をすると何が起こりますか？ `git log
   --all --oneline`を実行したときには何を確認することができますか？`git stash pop` を実行して `git stash`で行ったことを取り消し（undo）ましょう。どのような状況においてこの機能は便利でしょうか？
1. 多くのコマンドラインツールのように、Gitは設定ファイル（またはドットファイル）を提供してくれます。その設定ファイルは `~/.gitconfig`と呼ばれます。 `git graph`を実行したときに`git log --all --graph --decorate
   --oneline`という出力を得るために、`~/.gitconfig`の中にエイリアスを作成しましょう。
1.  `git config --global core.excludesfile ~/.gitignore_global`を実行した後には、 `~/.gitignore_global`の中にグローバルな無視パターンを定義することができます。これを実行してみて、OS固有またはエディタ固有の`.DS_Store`といった一時ファイルを無視するための、あなたのグローバルgitignoreファイルを設定しましょう。
1. [講義サイト用のレポジトリ](https://github.com/missing-semester/missing-semester)を分岐させ、打ち間違えや他にあなたが改善することができる箇所を見つけて、プルリクエストをGitHub上で提出しましょう。
