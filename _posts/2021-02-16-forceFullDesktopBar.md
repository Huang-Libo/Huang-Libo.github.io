---
title: 在 macOS 的 Mission Control 中展示完整的桌面预览
categories: [macOS, macOS Tools]
tags: [macOS]
---

## 背景

在 macOS 的某个大版本升级后，Mission Control 的*桌面预览*变成了*只有桌面编号的缩略样式*，在不同桌面之间切换时非常不方便。  

后来我发现了一个叫 forceFullDesktopBar 的插件，可以强制 Mission Control 展示**完整的桌面预览**。

## forceFullDesktopBar 简介

原理：Modifies the Dock process so that the desktop bar in **Mission Control** is always full size and showing **previews**.  

仓库地址：[https://github.com/briankendall/forceFullDesktopBar](https://github.com/briankendall/forceFullDesktopBar)  

说明：暂不支持 arm64（即 M1 芯片）的机器。原因请看仓库介绍。

## 步骤一：先关闭 System Integrity Protection 的部分模块

说明：下述操作适用于 macOS 10.14 及以上的系统。

- 重启电脑，按 cmd + R 进入恢复模式（不要使用外接键盘）。
- 打开终端，输入：

```
csrutil enable --without debug --without fs
```

重启：

```
reboot
```

## 步骤二：下载 forceFullDesktopBar

```
git clone git@github.com:briankendall/forceFullDesktopBar.git
```

进入到 `forceFullDesktopBar/install` 目录，然后安装：

```
sudo ./install.sh
```

再使用 Mission Control 的桌面预览时，就能看到完整的视图了。

## FAQ

### 升级系统后，forceFullDesktopBar 失效了

先执行 `sudo ./uninstall.sh` 卸载旧版本，再按上面的步骤重新安装 forceFullDesktopBar（安装前最好去检查一下原仓库是否有更新或 **bugfix**）。
