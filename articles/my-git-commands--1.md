---
title: "俺様がよく使う Git コマンドを共有するぜ"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git"]
published: true
---

Git 使ってるかい？俺様はよく使ってるぜ。
今日は俺様がよく使う Git コマンドを共有するから、参考にしてくれ。
俺様の知らないコマンドがあったら、ぜひ教えてくれよな。

## 前提

俺様はバージョン 2.48.1 の Git を使ってて、普段は MacOS で開発しているぜ。

## コマンド

### ⭐︎⭐︎⭐︎★★ `git add .`

とりあえず `git add .` で全ての変更をステージングするよな！
ただ俺様はこの操作を VSCode の Source Control パネルでやることも多いから、星は3つだぜ。

### ⭐︎⭐︎⭐︎⭐︎⭐︎ `git commit -m "コミットメッセージ"`

コミットするよな！
コミットメッセージは history に残る方が便利だから VSCode の Source Control パネルは使わないようにしているぜ。

### ⭐︎⭐︎⭐︎⭐︎⭐︎ `git push`

コミットしたらリモートにプッシュするよな！
プッシュしないと他の人が俺様の変更を見れないぜ。

### ⭐︎⭐︎⭐︎⭐︎★ `git commit --amend --no-edit`

直前のコミットに対して、現在ステージングされている変更を追加するぜ。
フォーマットを修正した時とか、ちょっとテストコードを変えた時とかに使うことが多いぜ。
ただし歴史改変になるから必ず自分のブランチで使うようにしているぜ。

### ⭐︎⭐︎⭐︎⭐︎★ `git commit --amend -m "コミットメッセージ"`

コミットメッセージを変更したい時に使うぜ。typo しちゃったときとか、微妙に意図が伝わらないコミットメッセージを書いてしまった時とかに使うことが多いぜ。
これも歴史改変になるから、注意してくれよな。

### ⭐︎⭐︎⭐︎⭐︎⭐︎ `git switch ブランチ名`

ブランチを切り替えるぜ。俺様は `git checkout` よりも `git switch` を使うことが多いぜ。

### ⭐︎⭐︎⭐︎⭐︎⭐︎ `git switch -c ブランチ名`

新しいブランチを作成して切り替えるぜ。特に `git switch -c feature/xxxx` のように使うことが多いな！

### ⭐︎⭐︎⭐︎★★ `git switch -`

直前にいたブランチに戻るぜ！これは便利なんだよな！

よくやるコンボを紹介するぜ。

```bash
# 現在は main ブランチにいるとするぜ。
git switch -c feature/xxxx # 作業ブランチを作成するぜ
# --- 何かしら作業をして、数日経過したりしてしまったぜ。

git switch - # ここで main ブランチに戻るぜ
git pull # main ブランチを最新にするぜ
git switch - # さっきの作業ブランチに戻るぜ
git (rebase|merge) main # main ブランチの変更を作業ブランチに取り込むぜ。rebase か merge は好みで選んでくれよな。
```

### ⭐︎⭐︎⭐︎⭐︎★ `git rebase ブランチ名`

現在のブランチに対して、指定したブランチの変更を適用するぜ。
これは `git merge` と似てるけど、コミット履歴がきれいになるから俺様はよく使うぜ。
ただし、歴史改変になるから必ず自分のブランチで使うようにしているぜ。

これは作業ブランチを二段階に分けている時とかは rebase も2回使うことが多いぜ。

```bash
# 現在は main ブランチにいるとするぜ。
git switch -c feature/xxxx--1 # 作業ブランチを作成するぜ。
# --- 何かしら作業を行ったぜ
git switch -c feature/xxxx--2 # PR を分けるためにもう一つ作業ブランチを作成するぜ。
# --- 何かしら作業を行ったぜ

# xxxx--1 と xxxx--2 の両方に main ブランチの変更を取り込みたいとするぜ。
git switch main
git pull # main ブランチを最新にするぜ
git switch feature/xxxx--1
git rebase main # まずは xxxx--1 に main ブランチの変更を取り込むぜ
git switch feature/xxxx--2
git rebase feature/xxxx--1 # 次に xxxx--2 に xxxx--1 の変更を取り込むぜ。こうすることで綺麗なコミット履歴になるんだぜ。
```

### ⭐︎⭐︎⭐︎★★ `git merge ブランチ名`

現在のブランチに対して、指定したブランチの変更をマージするぜ。
これは `git rebase` と似てるけど、マージコミットが作成されるぜ。俺様はマージコミットが残るのが嫌いだから積極的には使わないぜ。
ただし、現在レビューしてもらっているリモートブランチに変更を追加する時や、複数人で作業する作業ブランチに対しては使うようにしているぜ。
そうしないとどこまでレビューしたかわからなくなるからな。


### ⭐︎⭐︎⭐︎⭐︎★ `git pull`

リモートの変更をローカルに取り込むぜ。`git pull` は `git fetch` と `git merge` を一度にやるコマンドなんだぜ。便利だろう？

### ⭐︎⭐︎⭐︎★★ `git pull --rebase`

これは自分の作業ブランチがリモートで変更されてかつ、ローカルも変更されている時に使うことが多いぜ。
`git pull --rebase` を使うと、リモートの変更をローカルの変更の前に適用してくれるんだ。
ただし、リモートの変更が自分の変更と競合する場合は、手動で解決する必要があるから注意が必要だぜ。

### ⭐︎★★★★ `git init`

新しくレポジトリを初期化する時に使うぜ。
そんな頻繁に使うコマンドじゃないから、星は1つだな。

### ⭐︎⭐︎★★★ `git commit --allow-empty -m 'initial commit'`

新しいレポジトリを初期化した際は、ひとまずこれを使って空のコミットを作成することが多いぜ！
1つ目のコミットは歴史改変ができないから、最初のコミットは空のコミットにしておくことが多いな。

同じ理由で CI が不調な時に、CI を再実行するためだけに空のコミットを作成することもあるぜ。
`git commit --allow-empty -m "run CI"` のように使うことがあるぜ。

### ⭐︎★★★★ `git status`

正直そんな使わないけど、困った時はこれを使うことが多いぜ！
作業状況は VSCode の Source Control パネルで確認することが多いから、星は1つだな。
ごく稀に rebase や merge の途中のままになってしまうことがあって、その時は `git status` を使って状況を確認することが多いぜ。

### ⭐︎⭐︎★★★ `git reset --soft HEAD~1`

直前のコミットを取り消して、ステージング状態に戻すぜ。
俺様は `git commit --amend --no-edit` を使うことが多いけど、一回整理したい時はこれを使うこともあるぜ。

### ⭐︎⭐︎★★★ `git reset --hard HEAD~1`

こっちは直前のコミットを取り消して、変更も全て破棄するぜ。
コミットしたものの、やっぱりいらないなって思った時に使うことが多いぜ。
ただし、これを使うと変更が全て失われるから、注意してくれよな。
よく使うのは間違って main ブランチにコミットしてしまった時とかだな。

```bash
# 現在は main ブランチにいるとするぜ。
# --- 何かしら作業をしているぜ
git add . # 変更をステージングするぜ
git commit -m "コミットメッセージ" # 変更をコミットするぜ

# --- ここで作業ブランチではなく main ブランチにコミットしてしまったことに気づくぜ。
git switch -c feature/xxxx # 作業ブランチを作成するぜ
git switch - # main ブランチに戻るぜ
git reset --hard HEAD~1 # 直前のコミットを取り消して、変更も全て破棄するぜ。
# --- これで main ブランチにはコミットが残らないぜ。
git switch feature/xxxx # 作業ブランチに戻って作業を続けるぜ
```

### ⭐︎⭐︎★★★ `git stash`

一時的な変更を保存するぜ。俺様は stash したことも忘れちゃうことが多いから、あまり使わないぜ。
ただ一瞬だけ作業を中断したい時とか、急にブランチを切り替えたい時に使うことがあるぜ。

### ⭐︎⭐︎★★★ `git stash pop`

直前に stash した変更を復元するぜ。

### ⭐︎⭐︎⭐︎★★ `git cherry-pick コミットハッシュ`

特定のコミットを現在のブランチに適用するぜ。
あるコミットを別のブランチにも適用したいことってあるよな？そんな時に使うぜ。
ただし cherry-pick した大元のコミットを改変してしまうと意味わかんねーことになることがあるから、注意してくれよな。

### ⭐︎⭐︎★★★ `git rebase -i コミットハッシュ`

インタラクティブなリベースを行うぜ。俺様は VSCode の `[GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)` っていう拡張機能で GUI でリベースを行うことが多いからあまり使わないけど、コマンドラインでリベースを行う時はこれを使うぜ。

### ⭐︎⭐︎⭐︎★★ `git bisect start HEAD $(git rev-list -n 1 --before="2 month ago" HEAD)`

git bisect を使ってバグの原因となるコミットを特定するぜ。
`git rev-list` と組み合わせることで、「２ヶ月前は確かに動いていたはずだ」というのをいい感じに指定できるんだ。
もちろん、１週間前とか、１年前とか、好きな期間を指定してくれよな。

### ⭐︎⭐︎⭐︎★★ `git bisect run bash -c "test コマンド"`

`git bisect` の結果を自動で判定するために使うぜ。探索する範囲が広い時に、手動で判定するのは大変だからな！

### ⭐︎★★★★ `git reflog`

Git の操作履歴を表示するぜ。これを使うと、過去にどんな操作をしたかを確認できるんだ。
間違えて `git reset --hard` しちゃった時とか、間違えてブランチを削除しちゃった時でも、ここから過去の状態に戻すことができるんだぜ。すごいだろ？

まぁ、俺様は間違えることが少ないからあまり使わないけどな！

### ⭐︎⭐︎★★★ `git worktree add ./フォルダ名 ブランチ名`

新しい作業ツリーを作成するぜ。これを使うと同じレポジトリの複数のブランチを同時にローカルで作業できるんだ。
基本はブランチ切り替えで作業するけど、サブの worktree があると便利なこともあるんだぜ！

よく使うコンボも紹介しちゃうぜ

```bash
# 現在は main ブランチにいるとするぜ。
git branch -c yutaura/worktree/sub-1 # サブの worktree を作成するぜ
git worktree add ../sub-1 yutaura/worktree/sub-1 # サブの worktree を作成するぜ
git branch --set-upstream-to origin/main yutaura/worktree/sub-1 # サブの worktree の upstream を設定するぜ
cd ../sub-1 # サブの worktree に移動するぜ
ln -s ../main-repository/.env .env # サブの worktree の方とメインの方で共有したい ignore ファイルがあれば、シンボリックリンクを作成するぜ。もちろん分けたい設定ファイルは個別に作成したっていいぜ。
```

## config

俺様がよく設定する Git の設定を紹介するぜ

### `git config --global --add --bool push.autoSetupRemote true`

こいつは鉄板だな！
これを設定しておくと `git push` を実行した時に、まだリモートブランチが存在しない場合でも自動でリモートブランチを作成してくれるんだ。
これがあると、毎回 `git push -u origin ブランチ名` って打たなくて済むから、めちゃくちゃ便利なんだぜ！

### `git config --global core.ignorecase false`

俺様は MacOS で開発しているから、ファイルの大文字小文字は区別されているが、Git ではデフォルトで大文字小文字を区別しないようになっているんだ。
ただそれだと `README.md` と `readme.md` のようなファイル名の違いを認識できなくなってしまうから、俺様は大文字小文字を区別するように設定しているぜ。
これを設定しておかないと例えば `table.tsx` っていうファイルを `Table.tsx` にリネームした際に Git が変更を検知しなくて、リモートとローカルで差分が出ることがあって、めちゃくちゃ困るんだ。

## 終わりに

思い出せる範囲で俺様がよく使う Git コマンドを紹介したぜ。
もし俺様の知らない便利なコマンドがあったら、ぜひ教えてくれよな！
俺様も忘れてたコマンドがあったら、この記事を更新するぜ！
もしこの記事が役に立ったら、いいねやシェアをしてくれると嬉しいぜ！