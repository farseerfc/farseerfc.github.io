我眼中的 Wayland 的是与非
=====================================

:slug: wayland-good-and-bad
:lang: zhs
:date: 2015-02-15 22:45
:tags: linux, wayland, xorg

连着 `五六年了 <http://www.phoronix.com/scan.php?page=news_topic&q=Wayland&selection=20>`_
，每年都有人说 wayland_ 要来了， X11 即将寿终正寝了。
毕竟 X11 这个显示服务器远在 Linux 诞生之前就有了，岁数都比我大，历史遗留问题
一大堆，安全性、扩展性都跟不上时代了。

.. _wayland: http://wayland.freedesktop.org/

看样子 2015年的确像是 Wayland 终于能用了 
--------------------------------------------------------------------

根据 `Arch 的 Wiki <https://wiki.archlinux.org/index.php/Wayland>`_ 上跟踪着的 wayland 进展，
`toolkit 方面 <https://wiki.archlinux.org/index.php/Wayland#GUI_libraries>`_ 
GTK+ 3, Qt 5, EFL, Clutter, SDL 等等的几个图形库都完整支持 wayland 并且在 
archlinux 中默认启用了。
`WM 和 DE 方面 <https://wiki.archlinux.org/index.php/Wayland#Window_managers_and_desktop_shells>`_
Gnome 3 已经有了实验性支持， KDE 5 方面大部分 QT5 程序都已经支持了就等 kwin_wayland
作为 Session Manager 成熟起来，E19 很早就支持了，以及目前 wayland 上的 
WM/Compositor 除了作为实验性参考实现的 weston 和上述 DE 之外，还有不少
`有趣 <https://github.com/Cloudef/loliwm>`_ 又
`好玩 <https://github.com/evil0sheep/motorcar>`_ 的新 WM 。
另外各个发行版支持程度方面 Fedora 21 上的 Gnome 3 也有实验性支持了。
总体上可以说从 Xorg 迁移到 Wayland 的准备已经基本就绪了。

于是问题就是 **我们是否应该换到 Wayland** 
--------------------------------------------------------------------

换句话说， **wayland 相比较于现在的 Xorg 来说到底有什么优势** ？