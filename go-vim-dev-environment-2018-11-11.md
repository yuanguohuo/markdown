---
title: 搭建vim的go开发环境 
date: 2018-11-11 10:43:16
tags: [vim, go, vim-go, YouCompleteMe]
categories: dev-tool
---

本文记录基于vim搭建go开发环境的过程。

<!-- more -->

# 更新vim (1)

由于vim-go插件在vim7.4上有bug，所以先把vim更新到vim8.1。准备好源码包vim-8.1.tar.bz2。

## 卸载vim7.4 (1.1)

```
# rpm -qa | grep vim
vim-minimal-7.4.160-4.el7.x86_64
vim-common-7.4.160-4.el7.x86_64
vim-enhanced-7.4.160-4.el7.x86_64
vim-filesystem-7.4.160-4.el7.x86_64

# yum remove vim-minimal-7.4.160-4.el7.x86_64  \
             vim-common-7.4.160-4.el7.x86_64   \
             vim-enhanced-7.4.160-4.el7.x86_64 \
             vim-filesystem-7.4.160-4.el7.x86_64
```

## 安装vim8.1 (1.2)

vim依赖ncurses需要事先安装。另外由于后面的YouCompleteMe依赖python，所以这里编译vim8.1的时候要加上python支持。先安装python-devel，否则，虽然configure不会报错，但其实python并没有被支持。这里只支持了python2，本来应该可以同时支持python3，但是没搞成功(vim的文档里提到Ubuntu16.04里同时支持python2和python3会有bug，但是我的环境是CentOS7，暂不深究)。

```
# yum install -y ncurses.x86_64  ncurses-devel.x86_64
# yum install -y python-devel.x86_64

# tar xjvf vim-8.1.tar.bz2
# cd vim81/  
# ./configure --with-features=huge                                 \
              --enable-multibyte                                   \
              --enable-rubyinterp=yes                              \
              --enable-pythoninterp=yes                            \
              --with-python-config-dir=/usr/lib64/python2.7/config \
              --enable-luainterp=yes                               \
              --enable-cscope                                      \
              --prefix=/usr/local/vim8.1

# make
# make install
```

需要说明的是:
* `--enable-multibyte`是为了打开多字节支持，否则在Vim中不能输入中文。
* 如此安装之后，vim8.1的runtime dir就是/usr/local/vim8.1/share/vim/vim81。

把vim的路径加入PATH，然后确认一下python被支持：

```
# vim --version | grep python
+comments          +libcall           +python            +vreplace
+conceal           +linebreak         -python3           +wildignore
```

# 安装go (2)

下载go语言包go1.11.2.linux-amd64.tar.gz，并解压到/usr/local/go1.11.2.linux-amd64。然后配置GOROOT，GOPATH和PATH。

```
# mkdir -p /home/workspace/go

# tail ~/.bashrc 
......
GOVERSION=1.11.2.linux-amd64
export GOROOT=/usr/local/go$GOVERSION
export GOPATH=/home/workspace/go
export PATH=$GOROOT/bin:$GOPATH/bin:/usr/local/vim8.1/bin:$PATH
```

# 安装vim插件管理工具Vundle (3)

```
mkdir -p ~/.vim/bundle
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

创建一个空的~/.vimrc文件，并添加如下内容：

```
"=========================== begin: Vundle settings ===========================
" Use Vim settings, rather than Vi settings (much better!).
" This must be first, because it changes other options as a side effect.
set nocompatible  "required

filetype off      "required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
" alternatively, pass a path where Vundle should install plugins
"call vundle#begin('~/some/path/here')

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'

" All of your Plugins must be added before the following line
call vundle#end()      "required

" Enable file type detection.
" Use the default filetype settings, so that mail gets 'tw' set to 72,
" 'cindent' is on in C files, etc.
" Also load indent files, to automatically do language-dependent indenting.
filetype plugin indent on    "required

" To ignore plugin indent changes, instead use:
"filetype plugin on
"
" Brief help
" :PluginList       - lists configured plugins
" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
" :PluginSearch foo - searches for foo; append `!` to refresh local cache
" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
"
" see :h vundle for more details or wiki for FAQ
" Put your non-Plugin stuff after this line
"=========================== end: Vundle settings ===========================
```

然后，执行
```
# vim +PluginInstall
```

# vim个性化配置 (4)

根据自己的习惯，为vim做一些个性化配置。例如，我在~/.vimrc末尾append如下内容：

```
"=================begin: general vim settings=================================
" When started as "evim", evim.vim will already have done these settings.
if v:progname =~? "evim"
  finish
endif

" allow backspacing over everything in insert mode
set backspace=indent,eol,start

if has("vms")
  set nobackup
else
  set nobackup
endif
set history=50		" keep 50 lines of command line history
set ruler		" show the cursor position all the time
set showcmd		" display incomplete commands
set incsearch		" do incremental searching

" For Win32 GUI: remove 't' flag from 'guioptions': no tearoff menu entries
" let &guioptions = substitute(&guioptions, "t", "", "g")

" Don't use Ex mode, use Q for formatting
map Q gq

" In an xterm the mouse should work quite well, thus enable it.
" set mouse=a

" This is an alternative that also works in block mode, but the deleted
" text is lost and it only works for putting the current register.
"vnoremap p "_dp

set t_Co=256

" Switch syntax highlighting on, when the terminal has colors
" Also switch on highlighting the last used search pattern.
if &t_Co > 2 || has("gui_running")
  syntax on
  set hlsearch
endif

" Only do this part when compiled with support for autocommands.
if has("autocmd")
  " Put these in an autocmd group, so that we can delete them easily.
  augroup vimrcEx
  au!

  " For all text files set 'textwidth' to 78 characters.
  autocmd FileType text setlocal textwidth=78

  " When editing a file, always jump to the last known cursor position.
  " Don't do it when the position is invalid or when inside an event handler
  " (happens when dropping a file on gvim).
  autocmd BufReadPost *
    \ if line("'\"") > 0 && line("'\"") <= line("$") |
    \   exe "normal g`\"" |
    \ endif

  augroup END
else
  set autoindent		"always set autoindenting on
endif " has("autocmd")

set textwidth=120

"whether not tabs are expanded to spaces, we set it on here.
"Yuanguo: it seems not to work for go source files; this is good.
set expandtab

" Number of spaces that a <Tab> in the file counts for.
set tabstop=4

"Number of spaces to use for each step of (auto)indent.
set shiftwidth=4

"disable the annoying beep sound
set vb

"on linux, if you press 'n' (next line) when vim has already reached the end of
"the file, the screen splashes, that is quite annoying; disable it:
set t_vb="" 

"yuanguo: This variable controls whether a window has a status line. 
"  0: A window never has a status line;
"  1: If there are more than one windows, each has a status line; 
"  2: A window always has a status line; 
set laststatus=2

"automatically read the changes made from the outside.
set autoread

"set mapleader
let mapleader = ","

"when move cursor virtically, screen will scroll if there are 7 lines at the
"bottom/up of the screen.
set so=7

"show line number; to hide the line number, use command 
"  :set nonu
set nu

"let the directory where session file resides become current directory of vim, 
"when loading the seesion file.
set sessionoptions-=curdir
set sessionoptions+=sesdir
"=================end: general vim settings=================================
 
"=================begin: make modification to .vimrc easy=====================
"open a file named fname in a new tab if it is not opened. If it is already
"opened but not the current tab (not disaplayed), then change it to the
"current tab.
function! SwitchToBuf(fname)
	let bwnr=bufwinnr(a:fname)
	if bwnr!=-1
		exe bwnr."wincmd w"
		return 
	else
		tabfirst
		let tab=1
		while tab<=tabpagenr("$")
			let bwnr=bufwinnr(a:fname)
			if bwnr!=-1
				exe "normal".tab."gt"
				exe bwnr."wincmd w"
				return
			endif
			tabnext
			let tab=tab+1
		endwhile
		exe "tabnew".a:fname
	endif
endfunction

map <silent> <leader>ss :source ~/.vimrc<cr>
map <silent> <leader>ee :call SwitchToBuf("~/.vimrc")<cr>
"execute source ~/.vimrc whenever it is re-written.
autocmd! bufwritepost .vimrc source ~/.vimrc
"=================end: make modification to .vimrc easy=======================
 
"=================begin: session and viminfo==================================
"save and load session
map <silent> <leader>mks :mksession! .working.session<cr>
map <silent> <leader>ls :source .working.session<cr>

"save and load viminfo
map <silent> <leader>mki :wviminfo! .working.viminfo<cr>
map <silent> <leader>li :rviminfo .working.viminfo<cr>
"=================end: session and viminfo====================================

"=================begin: buffer settings======================================
"A buffer in buffer-list can be in one of the following three stats:
" 1. active: the file is loaded into memory and displayed in a window; the
"            buffer may have been modified and thus be different from the file;
" 2. hidden: the file is loaded into memory but not displayed in a window; like
"            in activethe state, buffer may have been modified and thus be 
"            different from the file;
" 3. inactive: the buffer is not loaded into memory, nor displayed. The buffer
"            in buffer-list but doesn't contain anything.

"unload a buffer: free the memory allocated for the buffer. If the buffer is
"modified, unload will fail unless "!" is added to discard the modifications;
"unloaded buffer is still in the buffer-list but in "inactive" state;

"abandon a buffer: not display it anymore. for example, 1. quit from vim; 2.
"display another buffer by vim command "{buffer-number}b" or "e {file-name}".

"hidden is a global option for buffers. bufhidden is an option specific to a
"buffer. These 2 options affects the abandon operation:

"when abandon a buffer:
" 1. if bufhidden is empty:
"      a. if hidden is on: the buffer becomes hidden no matter if it is
"         modified;
"      b. if hidden is off: the buffer is unloaded (fail if modified);
" 2. if bufhidden is "hide": the buffer becomes hidden no matter if it is 
"    modified;
" 3. if bufhidden is "unload": the buffer is unloaded (fail if modified);
" 4. if bufhidden is "delete": the buffer is unloaded (fail if modified) and
"    then deleted from buffer-list; when deleted from buffer-list, it goes into
"    unlisted-buffer-list.
" 5. if bufhidden is "wipe": the buffer is unloaded (fail if modified), deleted
"    from buffer-list, and then deleted from unlisted-buffer-list;

" vim command "ls" displays only buffer-list, while "ls!" displays both
" buffer-list and unlisted-buffer-list;

"set hidden
"=================end: buffer settings========================================
```

# vim-go插件 (5)

## 安装vim-go (5.1)

有了第3节安装的vim插件管理工具Vundle，很容易安装vim-go。

* 在~/.vimrc中的Vundle配置里加入：
```
Plugin 'fatih/vim-go'
```

* 在~/.vim/bundle/目录下安装vim-go：
```
vim +PluginInstall
```

* 在$GOPATH/bin下安装二进制工具，并在$GOPATH/src/golang.org和$GOPATH/src/github.com下安装它们的源代码。
```
vim +GoInstallBinaries
```
这一步比较慢，并且若没有科学上网，golang.org/x/下的东西都会失败。可以手动安装它们：

```
mkdir -p $GOPATH/src/golang.org/x/  
cd $GOPATH/src/golang.org/x/  
git clone https://github.com/golang/tools.git   
git clone  https://github.com/golang/net.git
git clone  https://github.com/golang/text.git  
git clone  https://github.com/golang/crypto.git  
git clone  https://github.com/golang/sys.git
git clone  https://github.com/golang/sync.git
go install ./...
```

然后再通过`vim +GoInstallBinaries`安装其他工具。


## 配置vim-go (5.2)

vim-go提供的功能：
* 输入fmt.然后按ctrl+x, ctrl+o，vim会弹出补齐提示下拉框。这个补齐是由gocode提供的。不过这个补齐比较麻烦，后面YouCompleteMe会提供更加方便的补齐。
* 输入代码：`time.Sleep(time.Second)`并执行:GoImports，vim会自动导入time package。
* 执行:GoDef，vim会打开光标下的标识符的定义。
* 执行:GoDoc，vim会打开光标下的标识符的文档。
* 执行:GoLint，vim会在当前go源文件上运行golint。
* 执行:GoVet，vim会在在当前目录下运行go vet。
* 执行:GoRun，vim会编译当前package并运行它的main。
* 执行:GoBuild，vim会编译当前package。GoBuild不产生结果文件。
* 执行:GoInstall，vim会安装当前package。
* 执行:GoTest，vim会运行当前路径下的test.go文件。
* 执行:GoCoverage，vim会创建一个测试覆盖结果文件，并打开浏览器展示当前package的情况。
* 执行:GoErrCheck，vim会检查当前package里可能的未捕获的errors。
* 执行:GoFiles，vim会显示当前package对应的源文件列表。
* 执行:GoDeps，vim会显示当前package的依赖列表。
* 执行:GoImplements，vim会显示当前类型实现的interface列表。
* 执行:GoRename [to]，将当前光标下的标识符remame为[to]。

我们在~/.vimrc中为一些常用的工具设置快捷map：

```
"=================== begin: vim-go settings ===================
map <silent> <leader>imp :GoImports<cr>
map <silent> <leader>doc :GoDoc<cr>
map <silent> <leader>run :GoRun<cr>
map <silent> <leader>bld :GoBuild<cr>
map <silent> <leader>test :GoTest<cr>
map <silent> <leader>ec :GoErrCheck<cr>
"=================== end: vim-go settings ===================
```

# YouCompleteMe (6)

首先确认vim已经支持了python，否则，安装了YouCompleteMe之后每次运行vim都会报错：

```
YouCompleteMe unavailable: requires Vim compiled with Python (2.7.1+ or 3.4+) support
```

## 安装YouCompleteMe (6.1)

和安装vim-go插件一样，先在~/.vimrc中的Vundle配置里加入：`Plugin 'Valloric/YouCompleteMe'`，然后执行：`vim +PluginInstall`。这就把YouCompleteMe安装到~/.vim/bundle里了。但是，和vim-go不同的是，YouCompleteMe还需要进一步的安装。在 ~/.vim/bundle/YouCompleteMe/有一个install.py脚本，我们需要运行这个脚本来完成安装。为支持不同的语言，可以给这个脚本提供不同的选项，例如：`--go-completer`，`--java-completer`，`--clang-completer`等。这里我使用`--all`选项，支持所有常见语言。这样依赖也是最多的。下面先安装依赖，然后运行脚本。

```
# yum install -y cmake
# yum install -y mono-addins-devel.x86_64
# yum install -y npm.x86_64
# yum install -y cargo.x86_64

# cd ~/.vim/bundle/YouCompleteMe
# install.py --all
```

## 配置YouCompleteMe (6.2)

在~/.vimrc中加入:

```
"=================== begin: YCM settings ===================
"the number of characters the user needs to type before identifier-based completion 
"suggestions are triggered.
"Setting this option to a high number like 99 effectively turns off identifier-based
"completion and just leaves the semantic engine.
let g:ycm_min_num_of_chars_for_completion = 2

"the minimum number of characters that a completion candidate of the identifier
"completion must have.
"Yuanguo: if an identifier is shorter than this, it will never be a candidate of the 
"identifier-based completion.
let g:ycm_min_num_identifier_candidate_chars = 0

"the maximum number of completion suggestions from the identifier-based engine.
"0 means there is no limit.
"Setting this option to 0 or to a value greater than 100 is not recommended as it will slow
"down completion when there are a very large number of suggestions.
let g:ycm_max_num_identifier_candidates = 10

"the maximum number of completion suggestions from semantic completion engines.
"0 means there is no limit.
"Setting this option to 0 or to a value greater than 100 is not recommended as it will slow 
"down completion when there are a very large number of suggestions.
let g:ycm_max_num_candidates = 50


"when set to 0, this option turns off YCM's identifier-based completion (the as-you-type popup)
"and the semantic triggers (the popup you'd get after typing . or -> in say C++). You can still 
"force semantic completion with key mappings set by g:ycm_key_invoke_completion.
"if you want to just turn off the identifier-based completion but keep the semantic triggers, you 
"should set g:ycm_min_num_of_chars_for_completion to a high number like 99.
let g:ycm_auto_trigger = 1 

"this option controls for which Vim filetypes (see :h filetype) should YCM be turned on. the option 
"value should be a Vim dictionary with keys being filetype strings (like python, cpp, etc.) and values 
"being unimportant (the dictionary is used like a hash set, meaning that only the keys matter).
"the * key is special and matches all filetypes. By default, the whitelist contains only this * key.
"YCM also has a g:ycm_filetype_blacklist option that lists filetypes for which YCM shouldn't be turned 
"on. YCM will work only in filetypes that both the whitelist and the blacklist allow (the blacklist "allows" 
"a filetype by not having it as a key).
let g:ycm_filetype_whitelist = {'*': 1}

"which Vim filetypes (see :h filetype) should YCM be turned off. see g:ycm_filetype_whitelist above.
let g:ycm_filetype_blacklist = {
      \ 'tagbar': 1,
      \ 'qf': 1,
      \ 'notes': 1,
      \ 'markdown': 1,
      \ 'unite': 1,
      \ 'text': 1,
      \ 'vimwiki': 1,
      \ 'pandoc': 1,
      \ 'infolog': 1,
      \ 'mail': 1
      \}

"Yuanguo: Vim's has an option named 'completeopt' (see :h completeopt). we can 
"check its value by
"       :set completeopt?'    --yes, the question mark is important
"for example, on my Vim:
"       completeopt=preview,menuone
"if 'preview' is contained in the value, like above, Vim will open a preview-window
"automatically when completing. 

"if this option is set to 1
"   a. if there is no 'preview' in value of completeopt, YCM will add it; 
"   b. if there 'preview' already, there will be no effect;
"so if you want to disable the preview-window, you should:
"   a. remove 'preview' by 'set completeopt=menuone'
"   b. set this option to 0
let g:ycm_add_preview_to_completeopt = 0

"now, if the preview-window is not disabled, you may want it to be colsed
"automatically at some time. you have the following 2 options, and
"if the former is set, the latter is useless:

"When this option is set to 1, YCM will auto-close the preview window after 
"the user accepts the offered completion string.
let g:ycm_autoclose_preview_window_after_completion = 1

"When this option is set to 1, YCM will auto-close the preview window after 
"the user leaves insert mode; 
"this option takes effect only when g:ycm_autoclose_preview_window_after_completion
"is not set;
let g:ycm_autoclose_preview_window_after_insertion = 0


"key mappings used to select the first completion string. Invoking any of them repeatedly 
"cycles forward through the completion list.
let g:ycm_key_list_select_completion = ['<Tab>', '<Down>']

"key mappings used to select the previous completion string. Invoking any of them repeatedly 
"cycles backwards through the completion list.
let g:ycm_key_list_previous_completion = ['<S-Tab>', '<Up>']

"key mappings used to close the completion menu. This is useful:
"   1. when the menu is blocking the view; 
"   2. when you need to insert the <TAB> character;
"   3. when you want to expand a snippet from UltiSnips and navigate through it.
let g:ycm_key_list_stop_completion = ['<C-y>']

"key mapping used to invoke the completion menu for semantic completion. By default, semantic 
"completion is triggered automatically after typing ., -> and :: in insert mode (if semantic 
"completion support has been compiled in). 
"This key mapping can be used to trigger semantic completion anywhere. Useful for searching for 
"top-level functions and classes.
"Yuanguo: the default 'ctrl-Space' didn't work for me, so I set it as ctrl-u
let g:ycm_key_invoke_completion = '<C-u>'

"When this option is set to 1, YCM will query the UltiSnips plugin for possible completions of 
"snippet triggers.
let g:ycm_use_ultisnips_completer = 0


"If the ycmd completion server suddenly stops for some reason, you can restart it with this command.
"     :YcmRestartServer 

"Calling this command will force YCM to immediately recompile your file and display any new diagnostics 
"it encounters. Do note that recompilation with this command may take a while and during this time the 
"Vim GUI will be blocked.
"     :YcmForceCompileAndDiagnostics 
"=================== end: YCM settings ===================
```

# UltiSnips (7)

首先在~/.vimrc中的Vundle配置里加入：`Plugin 'SirVer/ultisnips'`，然后执行：`vim +PluginInstall`。需要注意的是，UltiSnips默认使用tab键进行补全，但是YouCompleteMe也使用tab键。所以，我们必须重新设置一个键。如下所示，我设置的是Ctrl-h。

```
"=================== begin: UltiSnips setting ===================
"Trigger configuration; defaut is <tab>; because we have already used <tab> in YCM, we cannot use it 
"again here.
let g:UltiSnipsExpandTrigger="<c-h>"
let g:UltiSnipsJumpForwardTrigger="<c-b>"
let g:UltiSnipsJumpBackwardTrigger="<c-z>"

"If you want :UltiSnipsEdit to split your window.
let g:UltiSnipsEditSplit="vertical"
"=================== end: UltiSnips setting ===================
```

# tagbar (8)

tarbar依赖前面vim-go提供的gotags和ctags。所以，要先安装ctags，然后在~/.vimrc中的Vundle配置里加入：`Plugin 'majutsushi/tagbar'`，最后再次执行：`vim +PluginInstall`。配置如下：

```
"=================== begin: go-tagbar setting ===================
let g:tagbar_type_go = {
    \ 'ctagstype' : 'go',
    \ 'kinds'     : [
        \ 'p:package',
        \ 'i:imports:1',
        \ 'c:constants',
        \ 'v:variables',
        \ 't:types',
        \ 'n:interfaces',
        \ 'w:fields',
        \ 'e:embedded',
        \ 'm:methods',
        \ 'r:constructor',
        \ 'f:functions'
    \ ],
    \ 'sro' : '.',
    \ 'kind2scope' : {
        \ 't' : 'ctype',
        \ 'n' : 'ntype'
    \ },
    \ 'scope2kind' : {
        \ 'ctype' : 't',
        \ 'ntype' : 'n'
    \ },
    \ 'ctagsbin'  : 'gotags',
    \ 'ctagsargs' : '-sort -silent'
\ }

map <silent> <leader>t :TagbarToggle<cr>
"=================== end: go-tagbar setting ===================
```

# 总结 (9)

记录一下配置go开发环境的过程，方便日后查询。
