---
title: "Install Hugo on Ubuntu"
date: 2021-11-10T08:00:50-05:00
lastmod: 2026-03-02T07:00:00-05:00

categories: ["Jamstack", "How-to"]
tags: ["Hugo", "Ubuntu"]
toc: false
author: "Igor Baiborodine"
---

In this post, I will show how to install Hugo on Ubuntu in three different ways.

<!--more-->

Hugo releases come in two flavors, namely **standard** and **extended**. 
The extended version differs in that it supports SASS/SCSS and PostCSS without any additional tooling or configuration.
Therefore, you should opt for the extended version if you choose a Hugo theme that uses Hugo pipes instead of tools like
Webpack for handling SASS/SCSS.

You can get the following error message while trying to build an "extended" website with the standard version:
```shell
Error: Error building site: TOCSS: failed to transform "style.scss" (text/x-scss). 
Check your Hugo installation; you need the extended version to build SCSS/SASS.: this feature is not available in your current Hugo version, see https://goo.gl/YMrWcn for more information
```

### APT
The first and obvious option would be to use the **APT**, a package-handling utility for Debian and Debian-based Linux distributions.
The following command will install an official [Hugo Debian package](https://packages.debian.org/search?keywords=hugo) which shared with Ubuntu:
```shell
sudo apt-get install hugo
```
But this command is not recommended as it will not install the latest version of Hugo. 
For example, the above command executed on Ubuntu 20.04.3 LTS installs `0.68.3/extended` Hugo release, whereas, at the time of writing, the newest version is `0.89.2`. 

When you check the location of the binary with the `whereis hugo` command, it gives `/usr/bin/hugo`.

### dpkg
To install the latest version of Hugo or a specific one, you can use **dpkg**, the Debian package manager. 
But first, you have to download the `.deb` package from the [official Hugo release page](https://github.com/gohugoio/hugo/releases). 
For example, to install version `0.156.0`, use the following commands:
```shell
wget https://github.com/gohugoio/hugo/releases/download/v0.156.0/hugo_0.156.0_linux-amd64.deb
sudo dpkg -i hugo_0.156.0_linux-amd64.deb
```
If you need the extended version, replace `hugo_0.156.0_linux-amd64.deb` with `hugo_extended_0.156.0_linux-amd64.deb`.

The location of the binary will be `/usr/local/bin/hugo` when installed with `dpkg`.

### Snap
The third option is to use the **Snap** package manager with which you can install Hugo packaged as a **snap** that is available via the [Snap Store](https://snapcraft.io/hugo).
Essentially, a snap is a bundle that contains an application and all of its dependencies compressed into a single file.
With the following command, you can install the latest standard version:
```shell
snap install hugo
```
Or, if you need the extended version, add the `--channel=extended` option:
```shell
snap install hugo --channel=extended
```
You can easily switch between the standard and extended versions using either `snap refresh hugo --channel=extended` or `snap refresh hugo --channel=stable`. 

With snap, the location of the binary will be `/snap/bin/hugo`. 

When you want to run Hugo installed via Snap, you should prefix the `hugo` command with `snap run`, for example:

![Snap Run Hugo](/img/content/article/install-hugo-on-ubuntu/snap-run-hugo.png)

And to summarize, it is evident that the **APT** option is not the best as you cannot get the latest version of Hugo.
In contrast, with the **Snap** package manager, you can only get the latest version.
The advantage of Snap is the ease of switching from the standard version to the extended one and back.
With all this in mind, it can be argued that the **dpkg** option is the most versatile, as it allows you to install the
latest version and a specific one, either standard or extended.
Although switching with dpkg between the standard and extended versions might be a bit cumbersome.
