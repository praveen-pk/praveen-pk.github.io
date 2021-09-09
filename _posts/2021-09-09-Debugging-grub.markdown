---
layout: post
title:  "Debugging in grub"
date:   2021-09-09 10:00:00 +0000
categories: jekyll update
---
In order to run grub in debug mode the following configurations should be added to grub.cfg:

```
set pager=1
set debug=all
```
Reference: [Enable_debug_messages](https://wiki.archlinux.org/title/GRUB#Enable_debug_messages)

Debug messages are printed by ***grub_dprintf*** method which checks if a condition for a module is enabled. I couldn't find the list of modules that can be selected for debug directive. A way to list all the supported values for the ***debug*** directive is to search for ***grub_dprinf*** through grub source code.

For example to debug loading of linux and initrd images following variables shoudld be set in grub.cfg

```
set pager=1
set debug=linux
```

