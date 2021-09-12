---
title: What happens when you run apt commands
date: 2019-09-09 11:30:56
tags: apt, Ubuntu, Linux, package management
---

### Background

In Linux world softwares are delivered in the form of `package`. As a software engineer wants to work in Linux platform, you'll run package management commands again and again. So you need to understand what actually happens when you run such commands. In this article, I'll introduce this topic. 

**Note** that different Linux distros have different package management tool. In this article let's focus on Ubuntu system and the package management tool is `apt`. 

### Software repositories

Nowadays, everybody is familiar with app store, which is one central location. And users can retrieve applications there. In Linux world, **software repositories** work just like app store. Software repositories are generally made available in the form of mirros, to which your Linux system subscribe. 

There are many repository mirrors in the world, you can decide which ones to use. In Ubuntu system, it's defined in the config file of `/etc/apt/sources.list` as follows: 

<img src="/images/sourceslist.png" title="sources list" width="800px" height="600px">

The URLs defined in the file are just software repositories. 


### Package index

The APT package index is essentially a database of available packages from the repositories mentioned above. 

In Ubuntu system, the package index files are stored in `/var/lib/apt/lists`

### apt update

Based on the above two points, we can easily understand what happens when we run `apt update`. It doesn't update any installed packages, instead it updates the **package index** files in your local machine by sending new network requests to the **software repositories**. That's also the reason why we need to run it before we run `apt upgrade` or `apt install`. Because the upgraded version or the newly installed version will depend on the updated package index files. In this way users will get the lastest version of the target package. 

### apt-cache search

Before you actually install a package, you can run `apt-cache search` command with the package name to get the list of any matched available packages. This can help you in case that you don't know the exact package name in advance. 

`apt-cache` command doesn't send any network request, instead it works based on the package index files in your local machine. How to prove this point? Of course the most direct way is reading the source code. But I want to show you another way to prove it with the tool of `strace`.

`strace` is a tool can show you all the invoked `system call` when you run a command. If our guess about `apt-cache search` is correct, it should invoke system call to read the files in the package index directory `/var/lib/apt/lists`, right? Let's have a look by running this command: 

```bash
 strace apt-cache search openvpn 2>&1 | grep open
```

Not that the **2>&1** part in the preceding command, this is because `strace` writes all its output to `stderr`, not `stdout`. And a `pipe` redirects stdout, not stderr. So you need to redirect stderr of strace to stdout before piping to `grep`. The output is very long but contains the following lines: 

<img src="/images/strace_aptcache.png" title="sources list" width="800px" height="600px">

Same as what we guessed. All right. 

### apt install

There are many people asking questions on the forums about the secret of `apt install` like [this one](https://askubuntu.com/questions/162477/how-are-packages-actually-installed-via-apt-get-install). Does `apt install` download the source code of packages and then build it on your machine? That's also one of the question came to my mind when I first touched Linux world. My intuition tells me there shouldn't be compiling and building process involved when I run `apt install`. Because I think it's too time consuming. 

In fact, `apt install` is used to install the **prebuilt binary package** to your machine. Simply speaking, it does two tasks:
- Download the package, a **Debian package**, to `/var/cache/apt/archives`
- Extract files from the Debian package and copy them to the specific system location

Because Ubuntu is originated from Debian system, so the package you install on Ubuntu is in fact a Debian package which is an archive file with the name ending `.deb`. For instance, the `openvpn` Debian package is as follows: 

<img src="/images/deb_package.png" title="sources list" width="800px" height="600px">

Then we can use `ar x <package_name.deb>` command to extract the package, you will find that there are three files inside each Debian package:

<img src="/images/content_debian.png" title="sources list" width="800px" height="600px">

- debian-binary – A text file indicating the version of the .deb package format.
- control.tar.gz – A compressed file and it contains md5sums and control directory for building package.
- data.tar.xz – A compressed file and it contains all the files to be installed on your system. This is what we need continue to exmpolre. 

Run the following `tar` command to open this tarball

```bash
tar -xf data.tar.xz
```

Note that the tarball contains the following directories: 

<img src="/images/content_datatar.png" title="sources list" width="600px" height="400px">

take a look into the `usr` directory, you'll find the prebuild binaries. 


