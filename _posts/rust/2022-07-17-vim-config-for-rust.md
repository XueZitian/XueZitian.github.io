---
layout: post
title: rust的vim编辑器环境配置
category: rust
---

作为一个vim爱好者，之前基于global+cscope搭建了一套用于c程序的开发环境，其可以方便的完成定义或引用的跳转，并且具有不错的性能。最近开始学习rust程序，发现对于rust程序，这套环境不太好用，于是在网上搜索了一番，为rust搭建了这样一个vim环境：rust.vim + rust-analyzer + coc.nvim + coc-rust-analyzer。

### [rust.vim][3]

rust.vim是rust官方提供的vim插件，提供了rust文件检测，语法高亮，错误检查，格式化等功能。

错误检查：rust.vim基于vim的Syntastic，自动注册cargo作为语法检查器。（Syntastic是vim的语法检查器插件，目前已经不再维护，替代插件是ALE，这里我并没有安装Syntasitic插件，但是错误检查是可以工作的，这一部分有待进一步了解）

概要预览：Tagbar是vim的一个插件，可以实现在侧边栏列出代码文件的中结构体，函数名和变量等，方便预览，其依赖**Universal Ctags**程序。rust.vim对Tagbar做了一些配置。

代码格式化：":RustFmt"命令可以借助rustfmt来格式化rust代码。rustfmt可以通过"rustup component add rustfmt"来安装。

### [rust-analyzer][4]

rust-analyzer是rust模块化编译器的前端，它的的核心是一个库，用于对随时间变化的Rust代码进行语义分析。在这里其主要作为LSP(Language Server Protocol)服务器的一部分运行。LSP允许各种代码编辑器，如VSCode、Emacs或Vim，通过与外部语言服务器进程通信来获取代码的语义信息，从而实现补全，跳转到定义和预览注释信息等。

那么对于vim，与rust-analyzer建立LSP通信的插件是coc.nvim。


### [coc.nvim][5]

coc.nvim是vim的一个插件，其提供了LSP功能支持，同时其本身也支持插件扩展，从而实现与不同语言的LSP服务器通信。对于rust，需要为coc.nvim安装coc-rust-analyzer插件。该插件依赖nodejs程序。

### [coc-rust-analyzer][6]

coc-rust-analyzer作为coc.nvim的插件，是建立coc.vim与rust-analyzer通信的桥梁。其可以通过":CocInstall coc-rust-analyzer"安装，并通过":CocConfig"来配置。

### 配置步骤如下：
1. 在linux上安装rust-analyzer
```bash
mkdir -p ~/.local/bin
curl -L https://github.com/rust-lang/rust-analyzer/releases/latest/download/rust-analyzer-x86_64-unknown-linux-gnu.gz | gunzip -c - > ~/.local/bin/rust-analyzer
chmod +x ~/.local/bin/rust-analyzer
```
2. 安装nodejs，版本>= 12.12。（需要切到root用户）
```bash
curl -sL install-node.vercel.app/lts | bash
```
3. 为vim安装rust.vim，Tagbar和coc.nvim插件  
我使用的是vundle插件管理工具，在.vimrc添加如下插件，然后运行":PluginInstall"。（coc.nvim安装完后，再次开启vim，可能会出现这个错误："[coc.nvim] build/index.js not found, please install dependencies and compile coc.nvim by: yarn install"。解决办法：1）sudo npm install -g yarn；2）cd ~/.vim/bundle/coc.nvim/；3）yarn install；4）yarn build）
```vim
Plugin 'taglist.vim'
Plugin 'rust-lang/rust.vim'
Plugin 'neoclide/coc.nvim', {'branch': 'release'} 
```
4. 安装和配置coc-rust-analyzer  
运行":CocInstall coc-rust-analyzer"安装，然后运行":CocConfig"来配置rust-analyzer程序的路径：
```bash
{
 "rust-analyzer.updates.prompt": false,
 "rust-analyzer.server.path": "$HOME/.local/bin/rust-analyzer"
}
```
这里的配置主要是为了解决每次打开rust文件都会弹窗提示更新rust-analyzer，所以禁止更新提醒，如果需要则手动下载最新的rust-analyzer。
5. 配置Tagbar，快捷键开启Tagbar，并且将Tagbar设置在左侧。.vimrc配置如下：  
```vim
" Tagbar
nmap <F9> :TagbarToggle<CR>
let g:tagbar_left = 1
```
6. Coc配置，这里主要是从coc.nvim的参考配置中挑选了一部分，配置了自动补全，定义跳转，参考跳转和预览浮窗的一些快捷建。.vimrc配置如下：
```vim
" Coc
" Use tab for trigger completion with characters ahead and navigate.
" NOTE: Use command ':verbose imap <tab>' to make sure tab is not mapped by
" other plugin before putting this into your config.
inoremap <silent><expr> <TAB>
      \ pumvisible() ? "\<C-n>" :
      \ CheckBackspace() ? "\<TAB>" :
      \ coc#refresh()
inoremap <expr><S-TAB> pumvisible() ? "\<C-p>" : "\<C-h>"
"
function! CheckBackspace() abort
  let col = col('.') - 1
  return !col || getline('.')[col - 1]  =~# '\s'
endfunction
"
" Use <c-space> to trigger completion.
if has('nvim')
  inoremap <silent><expr> <c-space> coc#refresh()
else
  inoremap <silent><expr> <c-@> coc#refresh()
endif
"
" Make <CR> auto-select the first completion item and notify coc.nvim to
" format on enter, <cr> could be remapped by other vim plugin
inoremap <silent><expr> <cr> pumvisible() ? coc#_select_confirm()
                              \: "\<C-g>u\<CR>\<c-r>=coc#on_enter()\<CR>"
"
nmap <silent> gf <Plug>(coc-definition)
nmap <silent> gy <Plug>(coc-type-definition)
nmap <silent> gi <Plug>(coc-implementation)
nmap <silent> gr <Plug>(coc-references)
"
" Use K to show documentation in preview window.
nnoremap <silent> K :call ShowDocumentation()<CR>
"
function! ShowDocumentation()
  if CocAction('hasProvider', 'hover')
    call CocActionAsync('doHover')
  else
    call feedkeys('K', 'in')
  endif
endfunction
"
nnoremap <silent><nowait><expr> f coc#float#has_scroll() ? coc#float#scroll(1) : "\<C-f>"
nnoremap <silent><nowait><expr> b coc#float#has_scroll() ? coc#float#scroll(0) : "\<C-b>"
inoremap <silent><nowait><expr> <C-f> coc#float#has_scroll() ? "\<c-r>=coc#float#scroll(1)\<cr>" : "\<Right>"
inoremap <silent><nowait><expr> <C-b> coc#float#has_scroll() ? "\<c-r>=coc#float#scroll(0)\<cr>" : "\<Left>"
vnoremap <silent><nowait><expr> <C-f> coc#float#has_scroll() ? coc#float#scroll(1) : "\<C-f>"
vnoremap <silent><nowait><expr> <C-b> coc#float#has_scroll() ? coc#float#scroll(0) : "\<C-b>"
```

## Reference
1. [Configuring Vim for Rust development][1]
2. [Rust：vim 环境配置][2]
3. [rust.vim][3]
4. [rust-analyzer][4]
5. [coc.nvim][5]
6. [coc-rust-analyzer][6]

[1]: https://blog.logrocket.com/configuring-vim-rust-development/
[2]: https://blog.csdn.net/m0_37952030/article/details/118372011
[3]: https://github.com/rust-lang/rust.vim
[4]: https://github.com/rust-lang/rust-analyzer
[5]: https://github.com/neoclide/coc.nvim/
[6]: https://github.com/fannheyward/coc-rust-analyzer
