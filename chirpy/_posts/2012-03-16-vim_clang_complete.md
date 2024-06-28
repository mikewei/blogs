---
title: vim开发辅助插件clang_complete
date: 2012-03-16 18:00:00 +0800
categories: [tools]
tags: [vim]
math: false
---
clang_complete号称是目前最强的vim上的语法自动补齐插件,其实现基于clang,一个强大的编译系统,动态地将c/c++文件解析成语法树,然后基于语法树做补齐,比其它实现更为准确

debian下安装过程:

    sudo apt-get install clang              #安装clang, 主要是一些命令行工具
	sudo apt-get install libclang-dev       #安装libclang, 底层库, 插件直接使用库更高效
    sudo apt-get install vim-nox            #安装支持python的vim

这里下载vim[clang_complete插件](https://www.vim.org/scripts/script.php?script_id=3302)

	vim clang_complete.vmb -c 'so %' -c 'q' #安装vim插件

修改~/.vimrc

    let g:clang_use_library=1               "这个选项不开有问题,原因未知
    let g:clang_auto_select=1				"自动高亮菜单第一个,更方便

试用中...

