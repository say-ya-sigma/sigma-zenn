---
title: "スマホでコード書く 環境構築編"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Linux","Vim"]
published: true
---

スマホでコードを書くとなると、パソコンの場合と前提条件が異なります。スマホは画面が小さく、可能な限りテキスト編集のみにリソースを割くべきで、また、スマホのIMEを使った快適なテキスト編集のため、高度な補完も大切です。任意のキーバインドが使えることもアドバンテージとなるでしょう。

## Vimを使う

スマホでVimを使うことは非常に理に適っています。Vimはディスプレイが限られた文字数の文字しか表示できない時代から支持されてきたエディタであり、またスマホのテキストの表示能力はその時代のディスプレイよりもリッチです。

また、Vimはキーボードが標準化される前から存在していてスマホのIMEのような入力環境でも十分機能します。

## LSP

Vimの補完はLanguage Server Protocol (LSP)を使います。LSPはMicrosoftが開発した言語の補完機能等に使われるプロトコルで、LSPを喋るサーバーと実際に補完をサジェストするクライアントで動きます。VSCodeでも使われていて、多くの言語の補完に応用されています。

Language Serverをスマホ上に立てて、スマホ上のVimから利用します。非常に多くの言語で高度な補完を実現できます。

![](https://storage.googleapis.com/zenn-user-upload/b70c0992362fe0454d07b247.png)

## tmux

複数のウィンドウを扱うためtmuxをインストールします。uimのブリッジであるuim-fep、tmux、作業用のターミナルの3段構成となります。

この構成の狙いはコマンドを使用した開発プロセスの自動化です。apiサーバーの立ち上げやnuxtの開発サーバーの立ち上げなどを自動化します。

## .vimrc

MicrosoftのPrabir Shrestha氏のプラグインasyncomplete-lsp.vimを使います。これはvim-lspとasyncomplete.vimを使った自動補完プラグインです。

mattn/lsp-server-settingsも使います。Laguage Serverのマネージャーです。まだLanguage Serverがインストールされていない言語のファイルを開いた際に、対応するLanguage Serverがあればインストールをサジェストします。これらを入れておくだけで、小難しそうなVimがIDEのように簡単に扱えます。プラグインの設定を書いて終わりです。

また、カラースキームとしsolarizedをインストールして見た目もIDEのようにしました。

```vim
set fenc=utf-8
set directory=~/vimdir/swp
set backupdir=~/vimdir/bk
set undodir=~/vimdir/undo

set autoread
set hidden
set confirm

set showcmd
set number
set visualbell

set cursorline
set virtualedit=onemore
set smartindent
set showmatch
set laststatus=2
set wildmode=list:longest
nnoremap <up> gk
nnoremap k gk
nnoremap <down> gj
nnoremap j gj
syntax enable

:set list listchars=tab:\▶\-
set tabstop=2
set shiftwidth=2
set expandtab
autocmd BufEnter *.go setlocal noexpandtab

set ignorecase
set smartcase
set incsearch
set wrapscan
set hlsearch
nmap <Esc><Esc> :nohlsearch<CR><Esc>

call plug#begin('~/.vim/plugged')
Plug 'prabirshrestha/async.vim'
Plug 'prabirshrestha/asyncomplete.vim'
Plug 'prabirshrestha/asyncomplete-lsp.vim'
Plug 'prabirshrestha/vim-lsp'
Plug 'mattn/vim-lsp-settings'
Plug 'mattn/vim-goimports'
Plug 'posva/vim-vue'
Plug 'altercation/vim-colors-solarized'
call plug#end()

inoremap <expr> <Tab>   pumvisible() ? "\<C-n>" : "\<Tab>"
inoremap <expr> <S-Tab> pumvisible() ? "\<C-p>" : "\<S-Tab>"
inoremap <expr> <cr> pumvisible() ? asyncomplete#close_popup() . "\<cr>" : "\<cr>"
let g:asyncomplete_auto_completeopt = 0
set completeopt=menuone,noinsert,noselect,preview
let lsp_signature_help_enabled = 0

let g:solarized_termcolors=256
set background=light
colorscheme solarized
```

## .bashrc

solarized用に256色モードにします。bashなどで使用できるSHLVL変数を使用して1段目2段目3段目で実行するコマンドを制御します。

PS1はプロンプトの形式を定義する変数でlsも独自カスタムにしています。色や画面サイズに合わせて調整するためです。

既にtmuxのmainセッション起動している場合にはアタッチし、そうでない場合はmainセッションを起動します。

```bash
export PATH=$PATH:/usr/local/go/bin
export PS1="\[\e[1;36m\]\W\[\e[m\]$ "
export LANG=ja_JP.UTF8
export TERM=xterm-256color
export LS_COLORS="di=1:fi=0:ex=32"

alias ls="ls -F --color"

if [ $SHLVL = 1 ]; then
	uim-fep -C black:white
fi

if [ $SHLVL = 2 ]; then
	tmux a -t main
	TMUX_RETURN=$?
	if [ TMUX_RETURN != 0 ]; then
		tmux new-session -s main
	fi
fi
```

## tmuxの使用例

今回の環境におけるtmuxの使い方の一例を紹介します。例えばzennのプレビュー機能を別windowで開くにはこのようなコマンドを叩きます。

```
tmux new-window -d -c $PWD -n zenn-pre "npx zenn preview"
```

UserLAndを使用し、ホストとネットワークを共有しているのでlocalhost:8000でプレビューできます。

![](https://storage.googleapis.com/zenn-user-upload/21d7a849dd14ce8d09bba379.png)

`-d`オプションは新しいウィンドウをカレントウィンドウにしないという意味のオプションです。`-c`オプションは新しいウィンドウのカレントディレクトリを指定するコマンドで、ここでは$PWDを渡しています。このコマンドを実行したディレクトリを引き継げます。`-n`オプションはウィンドウの名前を指定し、引数として与えられる""囲まれた文字列は新しいウィンドウで実行されるコマンドとして解釈されます。

名前を指定してウィンドウを起動しているので、ウィンドウの削除はこのようなコマンドで実現できます。

```
tmux kill-window -t zenn-pre
```

このシンプルな方法を応用すれば多くの開発プロセスを自動化できます。これだけでも十分強力なツールです。

## 執筆環境が快適になってしまった

開発環境を整備していたはずが、いつの間にか執筆環境が快適になっていました。リアルタイムプレビューしながらいい感じのテーマで執筆できています。物理キーボードを繋いでも良いですし、寝転がりながらポチポチしてもよく、快適です。
![](https://storage.googleapis.com/zenn-user-upload/22a1f9d43f78a3a2c5d54ab0.png)
