---
title: What happens when you run apt commands
date: 2019-09-09 11:30:56
tags: apt, Ubuntu, Linux, package management
---

### Background

In Linux world softwares are delivered in the form of `package`. As a software engineer wants to work in Linux platform, you'll run package management commands again and again. So you need to understand what actually happens when you run such commands. In this article, I'll introduce this topic. 

**Note** that different Linux distros have different package management tool. In this article let's focus on Ubuntu system and the package management tool is `apt`. 

repository list

package index

debian package

apt update

repo list
/etc/apt/sources.list

package index database
/var/lib/apt/lists

apt-cache search
strace

apt install

.deb package stored in: 
/var/cache/apt/archives
content of .deb package
tarball

dpkg -i
