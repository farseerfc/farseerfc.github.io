总结一下 Material Design 的 CSS 框架
=======================================

:slug: summarize-material-design-css-framework
:lang: zhs
:date: 2015-01-16 03:27
:tags: css, material, paper
:series: pelican

.. PELICAN_BEGIN_SUMMARY

现在这里的界面风格要从 Google 在 `I/O 2014 大会 <https://www.google.com/events/io>`_
上公布Android L 也即 后来的 Lollipop 说起。 他们在谈论界面设计的时候公布了他们的
设计准则： `Material Design <http://www.google.com/design/spec/material-design/introduction.html>`_ (`中文非官方翻译 <http://wcc723.gitbooks.io/google_design_translate/>`_ )。
当然这只是一些准则，总结并描述了之前在 Web 设计和移动端 App 界面设计方面的一些规范，
并且用材料的类比来形象化的比喻这个准则。关于 Material Design 的更多中文资料可
`参考这里 <http://www.ui.cn/Material/>`_ 。

看到 Material Design 之后就觉得这个设计风格非常符合直觉，于是想在这边也用上
Material Design。 但是我在 Web 前端科技树上没点多少技能点，所以想找找别人实现好的模板
或者框架直接套用上。在网络上搜索数日找到了这几个：


Polymer Paper Elements
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. PELICAN_END_SUMMARY

.. panel-default::
  :title: Polymer

  .. image:: https://www.polymer-project.org/images/logos/p-logo.svg
      :alt: Polymer logo

.. PELICAN_BEGIN_SUMMARY

Google 官方提供的参考实现应该是 `Polymer <https://www.polymer-project.org/>`_ 中的
`Paper Elements <https://www.polymer-project.org/docs/elements/paper-elements.html>`_ 。

.. PELICAN_END_SUMMARY

由于是 *官方参考实现* ，这个框架的确非常忠实地实现了 Material Design 的设计，但是同时
由于它基于 `HTML5 Web Components <http://webcomponents.org/>`_ 构建，相关技术我还
不太懂，浏览器兼容性和其余 HTML 技术的兼容性也还不太完善的样子……

并且对于我这个 Web 开发的半吊子来说，Polymer 只是提供了一组设计组建，没有完善的 
*响应式* (responsive) 布局支持，也没有 Navbar 这种常见的框架组建，真的要用起来的话还
需要手工实现不少东西。于是口水了半天之后只好放弃……以后可能真的会换用这个，只是目前需要学
的东西太多了。


Angular Material Design
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. panel-default::
  :title: AngularJS

  .. image:: https://angularjs.org/img/AngularJS-large.png
      :alt: AngularJS logo


`AngularJS <https://angularjs.org/>`_ 是 Google 对 Web Components 技术的另一个
尝试。而这额 `Angular Material Design <https://material.angularjs.org/>`_ 项目
就是基于 AngularJS 构建的Material Design 库啦，同样是 Google 出品所以应该算得上半个
官方实现吧。 相比于 Polymer, AngularJS 算是实用了很多，提供了基于 
`CSS Flexbox <http://www.w3.org/TR/css3-flexbox/>`_ 的布局。有人对这两者的评价是，
如果说 Polymer 代表了 *未来趋势* ，那么 AngularJS 就是 *眼下可用* 的 Web
Components 实现了。

只不过同样是因为它是 Components 的框架，对 WebApp 的支持很丰富，大量采用 Ajax 等
JavaScript 技术， 对于我这个静态博客来说仍然稍显高级了……非常担心还不支持 HTML5 的浏览器
比如 w3m 甚至 cURL 对它的支持程度。 于是最终也没有使用它。


Materialize
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. panel-default::
  :title: Materialize

  .. image:: https://raw.githubusercontent.com/Dogfalo/materialize/master/images/materialize.gif
      :alt: Materialize logo

`Materialize <http://materializecss.com/>`_ 这是一批(自称?)熟悉 Android 上
Material Design 的设计师们新近出炉的框架，试图提供一个接近 Bootstrap 的方案。
最早是在 `Reddit <http://www.reddit.com/r/web_design/comments/2lt4qy/what_do_you_think_of_materialize_a_responsive/>`_ 上看到对它的讨论的，立刻觉得这个想法不错。

体验一下官网的设计就可以看出，他们的动画效果非常接近 Polymer 的感觉，响应式设计的布局
也还不错。 只是同样体验一下他们现在的官网就可以看出，他们目前的
`bug 还比较多 <https://github.com/Dogfalo/materialize/issues>`_ ，甚至一些 bug
在他们自己的主页上也有显现。 虽然不想给这个新出炉的项目泼凉水，不过看来要达到他们声称的接近
Bootstrap 的易用度还任重而道远……


bootstrap-material-design + bootstrap3
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

这是我最终选择的方案。这个方案将三个项目组合在了一起，分别是 
`bootstrap-material-design <http://fezvrasta.github.io/bootstrap-material-design/>`_
, `pelican-bootstrap3 <https://github.com/DandyDev/pelican-bootstrap3>`_
和 `Bootstrap 3 <http://getbootstrap.com/>`_ 。
Bootstrap 3 想必不用再介绍了，很多网站都在使用这套框架，定制性很高。 
bootstrap-material-design 是在 Bootstrap 3 的基础上套用 Material Design 风格
制作的一套 CSS 库，当然也不是很完善并且在不断改进中，一些细节其实并不是很符合我的要求。
最后 pelican-bootstrap3 是用 Bootstrap 3 做的 pelican 模板。
这三个项目或多或少都有点不合我的口味，于是嘛就把 pelican-bootstrap3 fork了一套放在
`这里 <https://github.com/farseerfc/pelican-bootstrap3>`_ ，其中还包括我自己改
过的 `Bootstrap3 样式 <https://github.com/farseerfc/pelican-bootstrap3/tree/master/static/bootstrap>`_
和 `Material 样式 <https://github.com/farseerfc/pelican-bootstrap3/tree/master/static/material>`_
，需要的可以自取。

至于细节上我定制了哪些地方，敬请听下回分解……