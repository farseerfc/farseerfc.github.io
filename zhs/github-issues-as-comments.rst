启用 GitHub Issue 作为博客留言系统
====================================================

:id: github-issues-as-comments
:translation_id: github-issues-as-comments
:lang: zhs
:date: 2016-08-07 16:28
:tags: pelican, github, pages, issues
:series: pelican
:issueid: 52


从今天起本博客将启用 GitHub Issue 作为留言系统。
原本使用的 Disqus 将继续保留一段时间，目前没有关闭的计划。

换用 GitHub Issue 是计划了好久的事情了，最初重做这个主题的时候就有考虑过。
这个想法的契机是看到了这篇
`GitHub hosted comments for GitHub hosted blogs <http://ivanzuzak.info/2011/02/18/github-hosted-comments-for-github-hosted-blogs.html>`_
，然后立马觉得这个想法很符合寄宿在 GitHub Pages 上的博客。
一个限制是要求评论者必须有 GitHub
账户，考虑到我的博客的受众这个要求估计不算太过分。
使用 GitHub Issue 的好处么，比如自带的 GFMD
富文本格式，邮件通知，还有订阅和取消订阅通知，邮件回复，
这些方面都不比第三方留言系统逊色。

换用 GitHub Issue 另一方面原因是最近听说 Disqus
被部分墙了，想必以后墙也会越来越高。之前曾经试过在这个博客换上多说，
然而效果我并不喜欢，多说喜欢侵入页面加很多奇怪的东西，比如用户的头像通常是
http 的……也试过结合新浪微博的评论，而新浪微博越来越封闭，API 也越来越不靠谱。

使用 GitHub Issue 作为评论的方式比较简单，上面那篇博客里面提到了，代码量不比
加载 Disqus 多多少，而且没有了 iframe 的困扰，唯一麻烦的地方就是要稍微设计一下布局方式让它融入
现有的页面布局。
`我参考上面的实现在这里 <https://github.com/farseerfc/pelican-bootstrap3/blob/2ea6c9f3227275fe86ddaa75d8fc6496b3b03d8c/templates/includes/comments.html#L32>`_ 。
这个加载代码使用两个变量加载 Issue Comments ，一个是在 pelicanconf.py 里的
:code:`GITHUB_REPO` ，可以指向任何 Repo ，我指向 farseerfc/farseerfc.github.io
的这个 GitHub Page repo ，另一个变量是每篇文章里需要加上 :code:`issueid`
的元数据，关连文章到每个 Issue 上。

还有一个稍微麻烦的事情是现在每写一篇文章之后都要新建一个 issue 了。
手动操作有点累人，于是我 `写了个脚本 <https://github.com/farseerfc/farseerfc/blob/master/createissue.py>`_
自动搜索 pelican 的 content 文件夹里面文章的 slug 并且对没有 issueid 关连的
文章创建 issue 。

好啦新的留言系统的外观样式还在测试中，希望大家多留言帮我测试一下！

.. label-warning::

    **2016年8月7日19:30更新**

新增了对 GitHub Issue comments 里面
`reactions <https://developer.github.com/v3/issues/comments/#reactions-summary>`_
的支持，套用 font-awesome 的图标（似乎没 GitHub 上的图标好看）。这个还属于 GitHub API
的实验性功能，要加入 :code:`Accept: application/vnd.github.squirrel-girl-preview`
HTTP 头才能拿到。

.. label-warning::

    **2016年8月7日23:16更新**

感谢 @iovxw 的测试让我发现 github 的高亮回复和邮件回复是需要特殊处理的。
高亮回复用上了 `这里的 CSS <https://github.com/sindresorhus/github-markdown-css>`_
邮件引言的展开事件直接用 jQuery 做了：

.. code-block:: javascript

	  $(".email-hidden-toggle > a").on("click", function (e){
        e.preventDefault();
        $(".email-hidden-reply", this.parent).toggle();
      });


还得注意邮件的回复需要 CSS 里面 :code:`white-space: pre-wrap` 。