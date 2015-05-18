换到 farseerfc.me 域名
=======================================

:slug: switch-to-farseerfc-dot-me-domain
:lang: zhs
:date: 2015-01-26 23:32
:tags: pelican, domain, cloudflare, github
:series: pelican

上个月就在 :ruby:`狗爹|godaddy` 上买了个自己的域名 :code:`farseerfc.me` 准备用在这个
博客上，当时试着转到过这个域名，发现 :ruby:`自定义域名|custom domain` 
只支持 http 不支持 https ，想着还要买自己的证书，于是就扔在了一旁。不用自定义域名的话，
放在 github.io 上是可以用 HTTPS 的。
今天在 :irc:`archlinux-cn` 上受大牛 :fref:`quininer` 和 :fref:`lilydjwg` 点播，
发现 cloudflare 有提供
`免费的支持 SSL 的 CDN 服务 <https://blog.cloudflare.com/introducing-universal-ssl/>`_
赶快去申请了一个，感觉非常赞，于是就换过来了。

设置的方法按照 `这篇博文 <https://me.net.nz/blog/github-pages-secure-with-cloudflare/>`_
说的一步步做下来，如它所述，用 CloudFlare 的优点如下：

#. CDN 加速
#. SSL (HTTPS) 加密
#. 支持 SPDY 协议
#. 支持 IPv6 

然后 *免费账户* 的一些缺点有：

#. CloudFlare 和 github.io 之间的数据不是加密的，因为 github
   :ruby:`自定义域名|custom domain` 还不支持使用自己的证书。这也是一开始我没用
   自定义域名的原因嘛，这没有办法……
#. CloudFlare 给免费账户签名的 SSL 证书比较新，不支持一些老的设备和浏览器，比如不支持
   老的 XP 系统的 IE 或者 2.x 的 Android。这种情况下没办法只能用没有加密的 HTTP 了。
#. 不支持 `HSTS 头 <https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security>`_
   ，所以不能从服务器这边强制浏览器用 HTTPS。当然可以放个 javascript 跳转，
   也可以用 `HTTPSEverywhere <https://www.eff.org/https-everywhere>`_ 这种方案。


设置步骤 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

基本按照默认的选项下一步就可以了。

#. 和那个博主一样我把 :ruby:`安全级别|Security profile` 降到了 Low ，即使是可疑流量也
   不会要求输入 CAPTCHA 。
#. 把 SSL 方式开在 Flexible SSL，访客到 CloudFlare 是加密的，而 CloudFlare 到 
   github.io 是不加密的。
#. 把 CDN 开到了 CDT+Full Optimization ，可以对访问加速。由于是完全静态的博客，没有
   动态变化的内容，所以应该比较安全。
#. 服务器设置的一步需要将 :ruby:`域名解析服务器|DNS nameservers` 从狗爹的服务器改到
   CloudFlare 的，如下图：

.. figure:: {filename}/images/godaddy.png
    :alt: 更改狗爹的域名服务器

    更改狗爹的域名服务器

申请好之后就由 CloudFlare 接管域名解析了，接下来在 CloudFlare 的 DNS 设置添加一条
`A 类规则指向 github pages 的 IP <https://help.github.com/articles/tips-for-configuring-an-a-record-with-your-dns-provider/>`_ 。

.. figure:: {filename}/images/cloudflaredns.png
    :alt: 更改CloudFlare的DNS规则

    更改CloudFlare的DNS规则

等一切都反映到 DNS 服务器上就设置完成了，接下来给 
`farseerfc.github.io push 一个 CNAME 文件 <https://help.github.com/articles/adding-a-cname-file-to-your-repository/>`_
写上我的域名就可以了。我用 Makefile 配合我的 pelican 配置做这个：

.. code-block:: Makefile

    publish: rmdrafts cc clean theme
      [ ! -d $(OUTPUTDIR) ] || find $(OUTPUTDIR) -mindepth 1 -not -wholename "*/.git*" -delete
      rm -rf cache
      echo $(SITEURL) > content/static/CNAME
      $(PELICAN) $(INPUTDIR) -o $(OUTPUTDIR) -s $(PUBLISHCONF) $(PELICANOPTS)
      $(MAKE) rsthtml

    github:
      (cd $(OUTPUTDIR) && git checkout master)
      env SITEURL="farseerfc.me" $(MAKE) publish
      (cd $(OUTPUTDIR) && git add . && git commit -m "update" && git push)

.. code-block:: python

    SITEURL = '//' + getenv("SITEURL", default='localhost:8000')
    STATIC_PATHS = ['static', 'images', 'uml', 'images/favicon.ico', 'static/CNAME']
    EXTRA_PATH_METADATA = {
        'images/favicon.ico': {'path': 'favicon.ico'},
        'static/CNAME': {'path': 'CNAME'}
    }

然后把生成的静态网站 push 到 github 之后可以从项目设置里看到域名的变化：

.. figure:: {filename}/images/githubdomain.png
    :alt: Github 配置好自定义域名之后的变化

    Github 配置好自定义域名之后的变化

最后把Disqus的评论也迁移到新的域名，disqus有方便的迁移向导，一直下一步就可以了。

这样就一切都设置妥当了。

致谢
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

最后要感谢提供消息的 :fref:`quininer` 和 :fref:`lilydjwg` ，感谢撰写设置步骤的
*Jonathan J Hunt* ， 感谢 CloudFlare 提供免费 SSL CDN 服务，感谢 Github 提供
方便免费的 Pages 托管。