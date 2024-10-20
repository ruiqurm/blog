+++
title = "zettelkasten method"
date = 2024-09-03T22:21:00+08:00
tags = ["emacs"]
categories = ["emacs"]
draft = false
+++

## Zettelkasten方法很强调“联系” {#zettelkasten方法很强调-联系}


### fleeting notes: 使用org-capture捕捉idea {#fleeting-notes-使用org-capture捕捉idea}


### 永久笔记：分为literature 和concept两种 {#永久笔记-分为literature-和concept两种}


### literature note偏向内容引用 {#literature-note偏向内容引用}

> -   A literature note is a single note containing references to all the interesting passages in a book (or other piece of media) that you encounter.
> -   A literature note is one of the resources you will use to create main notes.


### concept note {#concept-note}

需要更加细致，能够“自我解释”


## Org-roam {#org-roam}

Zettl方法在org-mode下的实现是org-roam. （org-brain也是很好的插件，但是我没用过）


## 工作流 {#工作流}


### 知识/idea：这个用的是org-roam-catpure,即上文说的fleeting notes {#知识-idea-这个用的是org-roam-catpure-即上文说的fleeting-notes}


#### 即时的想法，或者任何其他的东西 {#即时的想法-或者任何其他的东西}


#### 其实还可以叠加上anki这样的软件辅助记忆。 {#其实还可以叠加上anki这样的软件辅助记忆}


#### fleet notes最终会被整理成blog，或者叫concept notes {#fleet-notes最终会被整理成blog-或者叫concept-notes}


### 事实（引用类）: 类似上面的literate note {#事实-引用类-类似上面的literate-note}


#### 一个人的做法是，把这些事实类的东西都放到一个文件里，然后通过heading和org-ql去索引 {#一个人的做法是-把这些事实类的东西都放到一个文件里-然后通过heading和org-ql去索引}


#### org-roam也建议有一个子目录索引事实的类别，可以通过tag or heading去索引 {#org-roam也建议有一个子目录索引事实的类别-可以通过tag-or-heading去索引}


### 文章blog： {#文章blog}


#### 这个主要是多一个导出的插件。例如可以做成能用hugo导出就可以了 {#这个主要是多一个导出的插件-例如可以做成能用hugo导出就可以了}


### 个人进度task：用于维护一个项目的上下文，记录每个项目的进度 {#个人进度task-用于维护一个项目的上下文-记录每个项目的进度}


#### 结合org-column-view,org-agneda,org-todo,org-superlink这些插件 {#结合org-column-view-org-agneda-org-todo-org-superlink这些插件}


## 参考 {#参考}


### <https://www.orgroam.com/manual.html> {#https-www-dot-orgroam-dot-com-manual-dot-html}


### [What is a Literature Note](https://writing.bobdoto.computer/what-is-a-literature-note/) {#what-is-a-literature-note}
