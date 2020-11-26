---
layout: post
title: Vim & Nerdtree
date: '2013-10-11 07:48'
comments: true
tags:
  - Vim
  - System
abbrlink: 10479
---
最近重新整理vim的設定檔，意外的發現
[http://yoursachet.com/](http://yoursachet.com/)
這個網站滿好用的，可以根據你的需求來自動打造vim設定檔，對於不想動腦去研究設定檔而言的人來說是滿好用的工具
用滑鼠輕鬆點點就可以產生堪用的VIM了!!

<!--more-->

### vimrc 設定
```
set encoding=utf-8
set fileencodings=ucs-bom,utf-8,big5,latin1
set fileencoding=utf-8
set termencoding=utf-8
set number              " 行號
set statusline=%<\ %n:%f\ %m%r%y%=%-35.(line:\ %l\ of\ %L,\ col:\ %c%V\ (%P)%)
set ai                  " 自動縮排
syntax on               " 色彩標示

set tabstop=4               " tab使用四個空白取代
set shiftwidth=4            " 縮排空白數，要搭配set cin使用
set cin
set cursorline              " 該行的線
set t_Co=256                " 支援 256 色
set textwidth=0
set backspace=2 		    "按下backspace會後退，道行首後會刪除到前一行
set showmatch			    "顯示括號配對情況
set nocompatible			"用vim的特性去運行，捨棄vi的特性

" Pathogen
call pathogen#infect()
call pathogen#helptags()

filetype plugin indent on

" Nerdtree
autocmd VimEnter * NERDTree
autocmd VimEnter * wincmd p
let NERDTreeShowBookmarks=1
let NERDTreeChDirMode=0
let NERDTreeQuitOnOpen=0
let NERDTreeMouseMode=2
let NERDTreeShowHidden=1
let NERDTreeIgnore=['\.pyc','\~$','\.swo$','\.swp$','\.git','\.hg','\.svn','\.bzr']
let NERDTreeKeepTreeInNewTab=1
let g:nerdtree_tabs_open_on_gui_startup=0


set background=dark                 "背景顏色
colorscheme wombat
nnoremap <silent> <F5> :NERDTree<CR>
"normal mode的時候+數字 可以切換tab
nnoremap <Esc>1 gt1
nnoremap <Esc>2 gt2
nnoremap <Esc>3 gt3
nnoremap <Esc>4 gt4
nnoremap <Esc>5 gt5
nnoremap <Esc>6 gt6
nnoremap <Esc>7 gt7
nnoremap <Esc>8 gt8

```



### NERDTree


### 更改呼叫方式，使用F5
nnoremap <silent> <F5> :NERDTree<CR>


### 在各界面中移動

- 按照順序往下移動 (crtl+w+w)
- 上一個view (ctrl+w+h)
- 下一個view (ctrl+w+l)

### 切割視窗

- 水平切割 (在該檔案前按i)
- 垂直切割 (在該檔案前按s)
i :水平
	s :垂直

### tab使用

- 開新tab並且切換過去 (t)
- 開新tab但不切換過去 (T)
- 下一個tab (gt)
- 上一個tab (gT)

# 個人資訊
我目前於 Hiskio 平台上面有開設 Kubernetes 相關課程，歡迎有興趣的人參考並分享，裡面有我從底層到實戰中對於 Kubernetes 的各種想法

組合包
https://hiskio.com/packages/7ey2vdnyN

疑難雜症除錯篇
https://hiskio.com/courses/440/about?promo_code=VEQ4N7G

單堂(CI/CD)
https://hiskio.com/courses/385?promo_code=13K49YE&p=blog1

基礎概念
https://hiskio.com/courses/349?promo_code=13LY5RE

另外，歡迎按讚加入我個人的粉絲專頁，裡面會定期分享各式各樣的文章，有的是翻譯文章，也有部分是原創文章，主要會聚焦於 CNCF 領域
https://www.facebook.com/technologynoteniu

如果有使用 Telegram 的也可以訂閱下列頻道來，裡面我會定期推播通知各類文章
https://t.me/technologynote

你的捐款將給予我文章成長的動力
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="hwchiu" data-color="#000000" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#fff" data-font-color="#fff" data-coffee-color="#fd0" ></script>