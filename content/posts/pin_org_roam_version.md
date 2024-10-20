+++
title = "修复spacemacs org-mode插件bug"
date = 2024-08-28T12:17:00+08:00
tags = ["tag1"]
categories = ["emacs"]
draft = false
+++

org-mode的体验没有想象中的那么好,在体验org-roam的过程中，先是遇到了db-sync的问题，然后又
发现自己的自动补全有问题，completion-at-point不会补全；换了一套spacemacs的配置后，这才跑
起来，但是链接又出问题了。一搜，才发现去年就有人反应问题了。

1.  对于zoom用户，可以直接用这里的方法[doom中pin住org版本的方法](https://github.com/org-roam/org-roam/issues/2361#issuecomment-1632943836)。
2.  spacemacs用户,我在issue里面找到一个回答[pin a specific package version](https://github.com/syl20bnr/spacemacs/issues/16520)
