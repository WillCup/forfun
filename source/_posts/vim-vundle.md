title: vim-vundle
date: 2015-12-07 14:55:44
tags: [vim, vundle]
---

传说中的vim，今儿哥们儿要强迫自己继续装个B，吼吼~~ 为哥的勇气鼓掌！！

### 0.安装vundle
vundle是一个vim的插件管理器，涉猎了一段时间，发现很多人在用，为了更好更快地入门，就先玩儿个最活跃的，预料体验也不会特别差。

> git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim

### 1.配置vundle
就是下载自己需
要的一些插件，其实现在我毛线都不知道，就是网上随便搜了几个比较综合了一下。
建议参考：https://github.com/VundleVim/Vundle.vim，了解最新的配置说明。
BTW, 牛人参考: https://github.com/VundleVim/Vundle.vim/wiki/Examples

下面是我的~/.vimrc文件内容
```vim
et nocompatible                " be iMproved
filetype off                    " required!

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/vundle/
call vundle#rc()

" let Vundle manage Vundle
" 使用vundle来管理
Bundle 'gmarik/vundle'

"my Bundle here:
"
" original repos on github
" 相较于Command-T等查找文件的插件，ctrlp.vim最大的好处在于没有依赖，干净利落
Bundle 'kien/ctrlp.vim'

" 在输入()，""等需要配对的符号时，自动帮你补全剩余半个
Bundle 'AutoClose'

" 在()、""、甚至HTML标签之间快速跳转；
Bundle 'matchit.zip'

" 显示行末的空格
Bundle 'ShowTrailingWhitespace'

" 用全新的方式在文档中高效的移动光标，革命性的突破
Bundle 'EasyMotion'

" 自动识别文件编码；
Bundle 'FencView.vim'
" 必不可少，在VIM的编辑窗口树状显示文件目录
Bundle 'The-NERD-tree'

" 解放生产力的神器，简单配置，就可以按照自己的风格快速输入大段代码。
Bundle 'UltiSnips'
Bundle 'sukima/xmledit'
Bundle 'sjl/gundo.vim'
Bundle 'jiangmiao/auto-pairs'
Bundle 'klen/python-mode'
Bundle 'Valloric/ListToggle'
"Bundle 'SirVer/ultisnips'

" 迄今位置最好的自动VIM自动补全插件了吧
Bundle 'Valloric/YouCompleteMe'

Bundle 'scrooloose/syntastic'
Bundle 't9md/vim-quickhl'
" Bundle 'Lokaltog/vim-powerline'
Bundle 'scrooloose/nerdcommenter'

Plugin 'godlygeek/tabular'
Plugin 'plasticboy/vim-markdown'

"..................................
" vim-scripts repos
Bundle 'YankRing.vim'
Bundle 'vcscommand.vim'
Bundle 'ShowPairs'
Bundle 'SudoEdit.vim'
Bundle 'EasyGrep'
Bundle 'VOoM'
Bundle 'VimIM'
"..................................
" non github repos
" Bundle 'git://git.wincent.com/command-t.git'
"......................................
filetype plugin indent on	"这一行就是结束了

" 粘贴错位问题
nnoremap <F3> :set invpaste paste?<CR>
imap <F3> <C-O>:set invpaste paste?<CR>
set pastetoggle=<F3>


" nerdTree设置
" 在 vim 启动的时候默认开启 NERDTree（autocmd 可以缩写为 au）
autocmd VimEnter * NERDTree
" 按下 F2 调出/隐藏 NERDTree
" map  :silent! NERDTreeToggle
silent! map <F2> :NERDTreeToggle<CR>
let g:NERDTreeMapActivateNode="<F2>"
" 将 NERDTree 的窗口设置在 vim 窗口的右侧（默认为左侧）
" let NERDTreeWinPos="right"
" 当打开 NERDTree 窗口时，自动显示 Bookmarks
let NERDTreeShowBookmarks=1
```

### 2.简单使用
本人的一贯原则，首先找官方的文档或者手册怎样查找: 在vim中使用:h vundle定位
![](/imgs/vim/help_vundle.png)


|*命令*|*描述* |
|---|:---|
|:PluginUpdate|更新指定的plugin|
|:PluginList| 显示已经安装的plugin|
|:PluginClean| 清除配置里没有的plugin，针对bundle目录|
|:PluginSearch xxx| 搜索是否包含某个plugin|


### 3. do it yourself
既然配置好了插件，也学习了vundle的最基础命令，最后就只剩下我们去实际体验了。
![执行:PluginInstall之后的界面](/imgs/vim/install_vundle_plugin.png)
上面的过程其实就是在安装所有.vimrc里声明给vundle要去安装的插件，安装后的目录结构如下图：
![](/imgs/vim/vim_plugins_dir.png)
*友情提示*：安装完这些组件后，发现vundle的命令是可以通过tab进行自动补全的。


### 4.安装markdown插件
由于刚刚立志要强迫自己玩儿转vim，所以先安装个markdown插件试试咯。
- vimrc里添加 
> Plugin 'godlygeek/tabular'
> Plugin 'plasticboy/vim-markdown'
- 之后自己写这个*.md文件，发现vim已经识别了这个文件是markdown，表现是能够识别符号，比如-xxx之后换行会自动-等
- 遗憾是没有发现preview的方法，目前的方式是先启动hexo，然后一边编写另一边进行实际的view

### 5.资源
> http://vim-scripts.org/vim/scripts.html 
> https://github.com/vim-scripts
