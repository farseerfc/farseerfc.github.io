archlinux 上用 chrome 实现 :del:`透明计算` 远程登录 
====================================================================

:slug: arch-chrome-remote-desktop
:lang: zhs
:date: 2015-02-13 20:39
:tags: linux, archlinux, arch, chrome, remote, desktop

.. contents::

`透明计算 <http://news.sciencenet.cn/htmlnews/2015/1/311393.shtm>`_ 
具体是什么，因为他们没有公开技术细节所以我并不知道，只是看
`公开出来的演示视频 <http://v.qq.com/page/h/v/q/h0145ebh1vq.html>`_ 
，感觉似乎只要能从手机上远程登录系统桌面，就能算是透明计算了。
如果透明计算真是这个意思，那么我似乎已经用着这个技术很多年了嘛。

Xorg 上常用的远程桌面工具有很多，基于 VNC 协议的、基于NX的和基于 RDP 协议的都能找到，
直接 ssh X forwarding 效果也不错。只是这些方案的一个 *不太易用* 的地方在于，需要
通过 ip 访问到远程的电脑，所以在跨越 NAT 之类的情况下不太容易使用。

于是今天介绍一个使用方便设置也简单的方法： 通过 chrome-remote-desktop 在 archlinux 
上使用远程桌面。这个方案的优势在于，借助 Google 的云端服务器（内部貌似是XMPP协议下的握手）
方便地实现了 NAT 穿透，无论什么网络环境基本都能使用。当然，要支持远程登录，
位于远端的登录的计算机必须一直开着 Chrome Remote Desktop 的后台服务。

.. panel-default:: 
	:title: Chrome Remote Desktop 插件

	.. image:: {filename}/images/chrome-remote-desktop-plugin.png
	  :alt: Chrome Remote Desktop 插件

Chrome Remote Desktop 的客户端
------------------------------------------------

虽然可能有很多人不知道，不过 Chrome 内包括远程桌面的功能很久了。只是这个功能的界面默认
没有提供界面，要使用它需要安装 Google 官方出品的 
`remote-desktop 插件 <https://chrome.google.com/webstore/detail/chrome-remote-desktop/gbchcmhmhahfdphkhkmpfmihenigjmpp>`_ 。
装好之后远程桌面的客户端就准备好，可以用来远程访问别的计算机桌面了（无论是 Windows/OS X
还是 Linux 都支持）。并且不光可以自己远程访问自己账户的桌面，还可以远程协助朋友的桌面。


Archlinux 上设置远程登录的服务器
------------------------------------------------

有了客户端之后还要设置一下才能让桌面作为远程登录的服务器。Windows 和 OS X 上 Chrome
会自动下载需要的安装包，无脑下一步就能装好了。Linux上由于发行版众多，桌面配置各异，
所以需要一点手动配置。官方的设置步骤记载在 `这里 <https://support.google.com/chrome/answer/1649523>`_
其中给出了 debian 用的二进制包和 Ubuntu 12.10 上的设置方式，以下设置是参考官方步骤。

首先要安装 chrome-remote-desktop 这个包，这个包实际上对应了 Windows/OS X 上用安装程序
安装的 Remote Desktop Host Controller。 archlinux 上开启了
`[archlinuxcn] <https://github.com/archlinuxcn/repo>`_
仓库的话，可以直接安装打好的包。或者可以从
`AUR <https://aur.archlinux.org/packages/chrome-remote-desktop/>`_ 装。

.. raw:: html

	<pre>$ pacman -Ss chrome-remote-desktop<br/><span style="color:purple;font-weight:bold;">archlinuxcn/</span><span style="font-weight:bold;">chrome-remote-desktop </span><span style="color:green;font-weight:bold;">40.0.2214.44-1</span><br/>Allows you to securely access your computer over the Internet through Chrome.</pre>

装好之后从会说这么一段话：

	groupadd：无效的组 ID “chrome-remote-desktop”

	Please create ~/.config/chrome-remote-desktop folder manually, if it doesn't exist, or else you can't use CRD.
	The needed files are created by the Chrome app, inside the chrome-remote-desktop folder, after Enabling Remote Connections.
	To {enable,start} the service use systemctl --user {enable,start} chrome-remote-desktop

	You may need to create a ~/.chrome-remote-desktop-session file with commands to start your session

	Go to https://support.google.com/chrome/answer/1649523 for more information.

那句报错是 AUR 里打的包还没跟上上游 Google 的更改导致的错误，
首先我们需要把远程登录的用户添加入 chrome-remote-desktop 这个用户组里。
新版本的 chrome remote desktop 提供了一个命令做这个事情，所以执行以下命令就可以了：

.. code-block:: console

	$ /opt/google/chrome-remote-desktop/chrome-remote-desktop --add-user

然后我们需要手动创建 :code:`~/.config/chrome-remote-desktop` 这个文件夹，内容是空的
就好了，随后 chrome 会往这里面放 :code:`host#.json` 文件用于身份验证。

.. code-block:: console

	$ mkdir ~/.config/chrome-remote-desktop

然后我们要创建一个 shell 脚本 :code:`~/.chrome-remote-desktop-session` ，这是远程
登录时的 .xinitrc ，内容么就是启动你想在远程登录时用的桌面环境。
这里可以指定一个和你正在登录的 WM/DE 不同的桌面，比如我启动 xfce4：

.. code-block:: console

	$ cat ~/.chrome-remote-desktop-session
	#!/bin/bash
	startxfce4
	$ chmod 755 .chrome-remote-desktop-session


接下来需要从 Chrome 的插件里启用远程桌面。打开 Chrome 的 Remote Desktop 插件，这时
应该可以看到一个「启用远程链接」的按钮。

.. figure:: {filename}/images/chrome-remote-desktop-enable-button.png
  :alt: Chrome Remote Desktop 插件中「启用远程链接」的按钮

  Chrome Remote Desktop 插件中「启用远程链接」的按钮

.. alert-warning::
	
	在撰写本文的时候， Archlinux 官方源里的 chromium 的版本和 aur/google-chrome 
	的版本尚且还是 40.0.2214.111 ，而 Chrome Web Store 中提供的 Chrome Remote 
	Desktop 的插件的版本是 41.0.2272.41 。虽然通常并不要求两者版本一致，不过貌似最近
	Chrome 内部的 Remoting 功能更改了 API 导致可能出问题。如果你找不到
	「启用远程链接」的按钮，请尝试一下新版本的 Chrome 比如 google-chrome-dev 。
	在这一步启用之后，老版本的 chrome 应该也就能使用远程桌面了。

.. alert-warning::
	
	在32位的 Linux 版本上，最近更新的 Chrome Remote Desktop 插件可能无法正确识别 Host
	的版本，具体 `参考这个 bug <https://code.google.com/p/chromium/issues/detail?id=332930>`_ 。


点击「启用远程链接」，设定一个 PIN 密码（不需要很复杂，这里首先有 Google 帐号验证保证只有
你才能访问），然后就能看到这套电脑的 hostname 出现在「我的电脑」列表里。

.. figure:: {filename}/images/chrome-remote-desktop-after-enabled.png
  :alt: 启用远程链接之后的样子

  启用远程链接之后的样子


同时，启用了远程链接之后，可以在刚刚创建的 ~/.config/chrome-remote-desktop 
文件夹中找到记录了验证信息的文件。

.. code-block:: console

	$ ls .config/chrome-remote-desktop 
	chrome-profile  host#8cfe7ecfd6bb17955c1ea22f77d0d800.json  pulseaudio#8cfe7ecfd6

然后就可以启动对应的 systemd 用户服务了，如果想自动启动服务要记得 :code:`systemctl --user enable` ：

.. code-block:: console

	$ systemctl --user start chrome-remote-desktop.service

如果上面的设置一切正常，就可以看到 chrome-remote-desktop 启动了另外一个 Xorg 执行你
刚刚指定的桌面环境：

.. figure:: {filename}/images/chrome-remote-desktop-htop.png
  :alt: htop 中看到的 chrome-remote-desktop 启动的另外一个 Xorg

  htop 中看到的 chrome-remote-desktop 启动的另外一个 Xorg

然后就可以试着通过 Remote Desktop 插件登录到这个新开的 Xorg 了：

.. figure:: {filename}/images/chrome-remote-desktop-xfce4.png
  :alt: 「远程」登录到新的 XFCE4

  「远程」登录到新的 XFCE4


Linux 版本的 Chrome远程桌面 和 Windows/ OS X 上的区别 
------------------------------------------------------------------


通过上面的设置步骤也可以看出，Linux版本的远程桌面会在后台开一个独立的 X 会话，而不能
复用现在已有的 X 会话。对远程登录的用法而言这还能接受，对远程协助的功能而言有点问题，
因为正在使用的人不能观察协助者做了什么，协助者也不能继续请求协助的人的操作。

当然目前 Chrome 远程桌面的 Linux Host Controller 还只是 beta 版本，官方只测试支持 
Ubuntu 12.04 和 12.10 （14.04之后似乎有 
`Bug <https://code.google.com/p/chromium/issues/detail?id=366432>`_
），所以不能要求太多。希望以后能改善吧。


Bonus： 手机远程登录
----------------------------------------

.. panel-default:: 
	:title: 手机上的 Chrome 远程桌面 App

	.. image:: {filename}/images/chrome-remote-desktop-android.png
	  :alt: 手机上的 Chrome 远程桌面 App

通过上面的设置就可以从任何一个 Chrome 远程桌面客户端登录刚刚设置的这台电脑了。
因为 Chrome 在三大桌面系统 Windows / OS X / Linux 上都有，所以应该能覆盖大多数桌面
系统了。

除了桌面的 Chrome 之外还有一个客户端是 Android 上的
`Chrome 远程桌面 App <https://play.google.com/store/apps/details?id=com.google.chromeremotedesktop>`_ 经过上面的设置之后，从这个 App 也能看到并登录： 

.. figure:: {filename}/images/chrome-remote-desktop-android-logined.png
  :alt: 手机远程登录

  手机远程登录

好啦，开始享受国家自然科学一等奖的透明计算技术吧！