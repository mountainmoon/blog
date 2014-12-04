---
layout: post
title: angular.js源码剖析(1)
description: "analysis of angular.js source code"
tags: [angular.js, javascript]
image:
  feature: abstract-#.jpg
---

从源码角度认识angular，深入理解angular的模块管理，依赖注入，指令，scope，以及重要对象的生存周期，了解angular源码文件的组织方式。

angular.js文件在加载完毕后，会在一个立即执行函数中进行一些系列变量的定义（包括函数），将一个空对象作为angular赋给window，尝试绑定jQuery，将部分定义的变量扩展到angular对象上，最后在文档加载完毕后开始初始化angular app。
代码结构大致为：

{% highlight javascript %}
(function(window, document, undefined) {
  //定义一些变量及函数
  //..
  var angular = window.angular || (window.angular = {}),
      angularModule,
  //..

  bindJQuery();

  publishExternalAPI(angular);

  jqLite(document).ready(function() {
    angularInit(document, bootstrap);
  });

})(window, document);
//..
{% endhighlight %}

<!--more-->

app启动过程中，angular会载入相关模块，针对各个模块，注册相关的directive, controller, service等，以及执行config，run函数，待modules处理完毕后，会生成$rootScope，在$rootScope上对$apply进行首次调用，调用过程中compile app根节点，并调用返回的link函数，最后调用$digest，进入首个digest循环。

可以看到，在这个过程中，包含了对angular很多重要对象的的处理，涉及到很多关键概念，因此，分析这个过程可以对angular是如何工作的，以及它的整个生命周期有深入的理解。下面的内容将围绕angular的启动过程而展开。


### 关于代码

文中的源码以2014.11.28 的 master branch(v1.3.4-14-g30694c8)为参考。从github上clone一份代码到本地以便查看。angular源码的结构可以从Gruntfile.js中了解。构成angular.js的代码文件在angularFiles.js中有列出：

{% highlight javascript %}
var angularFiles = {
  'angularSrc': [
    'src/minErr.js',
    'src/Angular.js',
    'src/loader.js',
    'src/stringify.js',
    'src/AngularPublic.js',
    'src/jqLite.js',
    'src/apis.js',

    'src/auto/injector.js',

    'src/ng/anchorScroll.js',
    'src/ng/animate.js',
    'src/ng/asyncCallback.js',
    'src/ng/browser.js',
    'src/ng/cacheFactory.js',
    'src/ng/compile.js',
    //..

    'src/ng/filter.js',
    'src/ng/filter/filter.js',
    //..

    'src/ng/directive/directives.js',
    'src/ng/directive/a.js',
    'src/ng/directive/attrs.js',
    //..
  ]
  //..
}
{% endhighlight %}

angularSrc数组中所列出的文件，它们所包含的代码都是与ng模块（angular核心模块）相关的，有些会作为angular的属性公开，有些会用来注册到ng模块，有些则用作内部使用。
这些文件中的代码在合并到一起后，全部都处在最外层立即调用函数的作用域下，因此在外部不可见的情况下，同层代码之间可以互相调用。根据api文档中的说法，
angular.\*所涵盖的这些方法、属性（不包括Object原型上的）都是属于ng模块的，但实际上，直接或间接使用angular.module方法来注册的api只有ng路径下的这些，
它们要么是可注入的service，provider，要么是filter、directive。如果一个页面存在多个app根节点，ng模块会被多次处理，但赋给angular对象的方法却不会重复处理。
angular.\*这些方法有种被强行划到ng模块的感觉。。


### angular对象和ng模块的初始化

以下按照angular源码的执行顺序来分析代码。*这里所说的ng模块初始化，仅包括angular对象上的属性定义以及使用angular.module对ng模块的注册*

#### angular对象

由前面的代码结构得知，进入立即函数后，首先是进行一些变量和函数的定义，这些定义好的东西在使用之前几乎都没必要关注，值得注意的是这里定义了angular对象，并将其赋给了window：
```
var angular = window.angular || (window.angular = {})
```

#### 尝试绑定jQuery

`bindJQuery();`

angular初始化过程中会用到操作DOM的方法，于是完成前面那些定义之后，就会调用定义好的bindJQuery方法。如果在加载angular之前没有加载jQuery，就会用一个包含了jQuery api子集的jqLite对象赋给angular.element，
来完成操作DOM等工作，另外这个对象还会扩充一些angular特性相关的方法。如果之前加载了jQuery，这里就会把jqLite对象上angular特性相关的方法扩展到jQuery.fn上，
将jQuery赋给angular.element。jqLite与jQuery有一个重要不同，它只能接受HTML string 或 DOMElement，不能接受选择器或其他类型的对象作为参数。
与angular特性相关的那些方法在描述相关特性时再提。

#### 公开外部api

`publishExternalAPI(angular);`

函数的代码大致如下：
{% highlight javascript %}
function publishExternalAPI(angular) {
  extend(angular, {
    'bootstrap': bootstrap,
    'copy': copy,
    'extend': extend,
    //..
  });

  angularModule = setupModuleLoader(window);
  //..
  angularModule('ngLocale', []).provider('$locale', $LocaleProvider);

  angularModule('ng', ['ngLocale'], ['$provide',
    function ngModule($provide) {/*..*/}
  ]);
}
{% endhighlight %}
这里大概干了这么几件事，公开前面定义的那些外部api到angular；定义module方法并赋给angular.module以及前面声明的angularModule变量。定义了ngLocale和ng模块，
后者依赖于前者。ngLocale是一个默认的本地化模块（'en-us'），如有需要可以用另外的本地化模块来覆盖。

这里值得注意的是module方法的定义。module方法是在setupModuleLoader中定义的。构造module方法的大致如下：
{% highlight javascript %}
function setupModuleLoader(window) {
  //..
  function ensure(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  }
  //..
  return ensure(angular, 'module', function() {
    var modules = {};  // 在这里保存module实例

    return function module(name, requires, configFn) {
      //..
      return ensure(modules, name, function() {
        //..
        var moduleInstance = {/*..*/}
        return moduleInstance;
      });
    };
  });
}
{% endhighlight %}

这里利用闭包将调用angular.module方法所注册的module实例保存到了外部无法访问的modules变量上。这里闭包体现了它的一种作用，保存函数调用所产生的某种状态，且不让外部直接接触。

#### 注册ng模块

定义完module方法后，立即就注册了ngLocale和ng模块。虽然ng模块依赖于ngLocale，但它们的注册顺序其实无关紧要，只要保证在以ng模块来创建injector之前，
相关依赖模块已注册即可。具体在后面分析injector时解释。以ngLocale和ng模块为例来看下模块实例是如何被创建的。module方法的代码大致如下：
{% highlight javascript %}
function module(name, requires, configFn) {
      //..
      return ensure(modules, name, function() {
        var invokeQueue = [];
        var configBlocks = [];
        var runBlocks = [];
        var config = invokeLater('$injector', 'invoke', 'push', configBlocks);
        var moduleInstance = {
          _invokeQueue: invokeQueue,
          _configBlocks: configBlocks,
          _runBlocks: runBlocks,

          requires: requires,
          name: name,

          provider: invokeLater('$provide', 'provider'),
          factory: invokeLater('$provide', 'factory'),
          //..
          controller: invokeLater('$controllerProvider', 'register'),
          directive: invokeLater('$compileProvider', 'directive'),
          config: config,
          run: function(block) {
            runBlocks.push(block);
            return this;
          }
        };

        if (configFn) {
          config(configFn);
        }
        return moduleInstance;

        function invokeLater(provider, method, insertMethod, queue) {
          if (!queue) queue = invokeQueue;
          return function() {
            queue[insertMethod || 'push']([provider, method, arguments]);
            return moduleInstance;
          };
        }
      });
    };
{% endhighlight %}

ngLocale是这样调用注册的，`angularModule('ngLocale', []).provider('$locale', $LocaleProvider);` 在module方法内部，会用对象字面量方式
来构造一个module实例，也就是上面代码中的 `moduleInstance` ，这个对象会将调用时传入的第一个参数，这里是'ngLocale'，作为它的name属性，
将第二个参数作为依赖传给属性requires，如果还有第3个实参，就会作为config来处理。module实例上还提供了一些熟知的方法，如controller，
factory，provider，config，run等。这些方法都具有一种**共同模式**，即把传入的参数（函数，或包含注入信息和函数的数组）与方法相关的信息合在一块（run方法没有这些信息）保存到某个数组中，留待以后调用。
这种只保存函数，暂时不调用的方式也为模块可以任意顺序注册提供了基础。对应不同的方法，一共有3种不同的数组。run 对应 runBlocks数组，config 对应 configBlocks 数组，其他方法对应invokeQueue数组。
这些数组还公开到moduleInstance上，在创建injector时会用到。

在DOMContentLoaded事件触发前（或者是主动调用bootstrap前），angular对象以及ng模块的（部分）初始化就结束了。

### angularInit

在DOMContentLoaded事件触发后，由最前面那段代码可知，angularInit函数会被调用， `angularInit(document, bootstrap);`.
{% highlight javascript %}
function angularInit(element, bootstrap) {
  var appElement,
      module,
      config = {};

  //为求简明，以下略有改动
  if (element.hasAttribute && element.hasAttribute('ng-app')) {
    appElement = element;
  } else {
    appElement = element.querySelector('[ng-app]')
  }
  module = appElement ? appElement.getAttribute(name) : undefined;

  if (appElement) {
    bootstrap(appElement, module ? [module] : [], config);
  }
}
{% endhighlight %}

这个方法使得 angular 可以自动寻找带ng-app特性的节点，并以此为app根节点来调用bootstrap方法。但有时候DOMContentLoaded的触发时机并不适合app启动，例如，
使用第三方模块加载工具（requirejs或seajs等）来加载angular时，就要手动调用bootstrap方法了。

后面可以看到在bootstrap方法中会通过创建一个injector来处理所有需要加载的模块，那些核心服务也会在那时被注册，并创建出相应实例，最后进行指令compile以及首次digest。























