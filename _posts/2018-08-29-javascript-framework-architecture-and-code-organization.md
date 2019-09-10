---
layout: post
title: "JS 框架的架构及其代码组织"
description: "JS 框架的架构及其代码组织，详细说明 JS 框架内在扩展逻辑"
date: "2018-08-29 02:20:13"
categories:
    - Weblogs
tags:
    - JavaScript
    - Weblogs
---

* toc
{:toc}


前端感觉就是个大杂烩，每半年都能玩出很多的新花样，实话实说，渐渐感觉力不从心。该收拾收拾一下自己的懒惰的习性，把懂的套路总结总结。前端的世界再怎么变，感觉其核心的东西，从来没有变过。

## 前言

从 IE6 的 `alert(1)` 开始走进前端的世界，到喜欢 Firefox 的 firebug，那时候很天真，觉得 firebug 已经是一切了。从浏览器的 `document.getElementById()` 刀耕火种的年代开始，到因为优秀的动画效果而喜欢 [Mootools](https://mootools.net/) 坚持不肯接受 [jQuery](https://jquery.com/) 到后来每个自己的小项目都带着它，期间还有各种大而全的 JavaScript Framework，比如影响了一代人的 [YUI](https://yuilibrary.com/) (顺便看了一眼其官网，已经停止更新，甚是唏嘘)。后来就有了 [Backbone](http://backbonejs.org/) 让大家知道原来原来前端还有 MVC 这样的方式。同时因为 [Node.JS](https://nodejs.com/) 的兴起，JS 模块化的实现越来越多，可以说是整个前端都因为模块化而蓬勃发展。前端的发展日新月异，各种的编译、打包、发布的工具层出不穷，催生了如 [Webpack](https://github.com/webpack) 这样优秀的工具。而随着手机和 Web 单页面的应用越来越多，也越来越复杂，于是也有了 [Angularjs](https://angularjs.org/) 和 [React](https://reactjs.org/) 这样优秀的前端框架，甚至影响前端的编码的方式。

工具和生态的发展，也促进了 JavaScript 这门语言的发展，人们不再局限在 JS 语言已有的体系当中，而是要求修正这么语言，于是一个又一个版本的 [ECMAScript ](http://www.ecmascript.org/) 规范不断地在刷新着我的认知范围，新的语法和新的语言结构也正在将 JS 变成一个巨大的语法糖的集合。

本文的主题是 JS 框架的架构及其代码组织，但是我并不打算从繁纷复杂的语言细节中去讨论 JS 框架的语言细节，而是从一个比较高的角度去理解其代码的组织，以及为什么这样的组织会有什么的语言细节。


## jQuery 框架

我一直觉得，不是 JavaScript 成就了 [jQuery](https://jquery.com/)，而是 [jQuery](https://jquery.com/) 成就了 JavaScript。即使到今天，很多同学不会 JavaScript 很多原生的方法，但用起 jQuery 来却毫不含糊。jQuery 从使用上就非常之直观，这导致其入门的门槛异常低，并且链式操作更加少写代码的利器，而且，从一个框架的 API 设计来说，jQuery 也设计得异常符合人的直觉，有时候都不用怎么看文档，猜猜就知道怎么用了。

### jQuery 实现基础

[jQuery](https://jquery.com/) 框架的实现基础，特别简单，也特别的有意思


    var jQuery = function(selector, context) {
        return new jQuery.fn.init(selector, context);
    };
    jQuery.fn = jQuery.prototype = {};

    // 重新指向 jQuery.fn.init.prototype 保证原型链的查找
    jQuery.fn.init = function(selector, context, root) {};
    jQuery.fn.init.prototype = jQuery.fn;

这个就是 jQuery 的整个框架的结构了，实现 `jQuery.fn.init` 这个函数就能实现 jQuery 的 `$()` 的功能啦！这里的原型链的处理巧妙到不得了。正常情况下，应该每次都这样来调用 `var jQuery = new jQueryInit()` 来得到一个全新的 jQuery 对象，然而这里内部 `new jQuery.fn.init(selector, context)` 那么势必重新指向 `jQuery.fn.init.prototype` 这里的实现真心巧妙。个中的秘密，自己体会一下。

### jQuery extend 及其代码组织

从这个框架结构来看，jQuery 真实需要扩展的也就是 `jQuery` 和 `jQuery.prototype` 这个地方，那么自然也就得到了 jQuery 整个框架的扩展方法。很多教程里涉及此函数都是说这是 jQuery 插件的调用方法，然而真正的事实是，jQuery 也使用此函数完成自身的扩展。

    jQuery.extend = jQuery.fn.extend = function() {
        var options, name, src, copy, copyIsArray, clone,
            target = arguments[ 0 ] || {},
            i = 1,
            length = arguments.length,
            deep = false;

        // 第一个参数为 boolean 的时候，那就进行 deepcopy
        if ( typeof target === "boolean" ) {
            deep = target;
            target = arguments[ i ] || {};
            i++;
        }

        // 默认是返回一个空的 object 来进行 extend
        if ( typeof target !== "object" && !isFunction( target ) ) {
            target = {};
        }

        // 此处是 jQuery 最为巧妙的实现！
        if ( i === length ) {
            target = this;
            i--;
        }

        for ( ; i < length; i++ ) {
            // 此处是 copy 的过程实现，需要查看具体实现请查看 jQuery 的源代码
        }

        return target;
    };

如果仔细研究，你会发现这段代码写得尤其巧妙。首先 `jQuery.extend` 可以作为工具函数来使用

    // 进行 object merge，尤其常用于 jQuery 的默认参数的传入
    var options = jQuery.extend({default: ''}, options);

    // 进行 object merge deepcopy
    var options = jQuery.extend(true, {default: ''}, options);

然而 `jQuery.extend = jQuery.fn.extend = function(){}` 加上我注释的这句 `target = this`，当传入参数只有一个的时候，那么可以这样来使用

    // 这样是扩展 jQuery 的静态方法，jQuery.example()
    jQuery.extend({
        example: function() {}
    });

    // 这样是扩展 jQuery 的原型方法，jQuery('').example()
    jQuery.fn.extend({
        example: function() {}
    });

这就是 jQuery 框架自己扩展的秘密，整个 jQuery 都是依赖 `jQuery.extend()` 和 `jQuery.fn.extend()` 来实现自身的扩展的。这个 `target = this` 动态实现了谁调用的，那就扩展到谁头上去，写得尤其巧妙，写得让人拍案惊奇！

### jQuery 框架实现之美

jQuery 框架的实现，既简洁，也实现得优雅。

jQuery 的很多地方的实现，都深刻地体现 JavaScript 的核心和本质。一个框架能如此集 JavaScript 精华之大成，我在佩服之余，更多的赞叹，框架的团队设计者们的功底如此深厚，对 JavaScript 的理解程度之深入，不得不让人折服。

当然啦，jQuery 的秘密还有很多，比如实现的链式调用，很多人都知道这是 `return this` 搞出来的语法糖，然而很少人会理解这是 jQuery 内部 `this = array[]` 化调用的结果。这样的处理，真是充分利用了 JavaScript 对于 array 的高效的实现。

jQuery 的秘密还有很多，大概列一下：

- jQuery.Sizzle 选择器的实现
- jQuery.Events 的处理，事件的处理实现得牛逼
- jQuery.Deferred/Queue 异步回调和队列的实现
- jQuery.Data 进行 cache 的实现
- jQuery.ajax 的实现

每个实现都堪称 JavaScript 业界代码实现的典范，如果要评个最优秀，恐怕没有之一。限于篇幅，没法每个都深入说明。如果想深入理解，那就只能自己好好体会一下 jQuery 的源代码啦！

抛开代码层面，jQuery 的代码似乎也有其一套实现的哲学，其 API 的设计几乎达到了 API 可理解性、前后一致性、使用预见性、简单性、保护性 一致的均衡，实属难得。


## Mootools 框架

[Mootools](https://mootools.net/) 作为与 jQuery 同时代的 JavaScript 框架，知道的人恐怕不多。远远不及其前辈 [Prototype](https://github.com/prototypejs/prototype) 框架的名气。对比与 jQuery 的实现，Mootools 的实现得实在不算怎么样，但是 Mootools 却用 `Class` 来组织和构造自己的框架实现，这从代码的组织上，我觉得是比 jQuery 更为先进的理念，虽然这理念现在才开始流行，ECMAScript 已经开始考虑将 `Class` 这个关键字放入到语言实现规范里面去。

从框架的实现上说，我个人理解上，Mootools 是一个很理想化的框架，它的基础代码实现层面上其实类似现在遍地都是的 Polyfill （或者叫 Polyfiller）的代码，Mootools 打算在原生的方法中和自己的框架理念中，有一个理想化的取舍。事实上，我觉得 Mootools 在用它自己的理念在 fix 原生 JavaScript 做得不好的地方，比如原生的 JavaScript 是没有类型的，但是 Mootools 扩展过的原生的类型，是自带 `$family` 和 `$constructor` 这个属性来标示原生的类型是啥、啥对象初始化出来的东西。从这个角度来说，Mootools 更加像是上帝，修复着 JavaScript 这个破碎的世界，对比于 jQuery 则是善于应对的市井小贩，圆滑地与 JavaScript 各种不协调的地方周旋着，却不曾想过从 JavaScript 语法层面去改变这个残缺的世界。


### Mootools 实现基础

因为 JS 所有的魔法几乎都跟 `Function` 有关，而 Mootools 一切实现的魔法，也跟 `Function` 有关。正如前面所说，Mootools 一开始就本着改造 JavaScript 自身的缺陷和不足去的，所以 Mootools 的核心实现，想也没有想就扩展了 JS 原生的 `Function` 增加了下面这两个原型链上的函数。

    Function.prototype.overloadSetter = function(usePlural){
        var self = this;
        return function(a, b){
            if (a == null) return this;
            if (usePlural || typeof a != 'string'){
                for (var k in a) self.call(this, k, a[k]);
                /*<ltIE8>*/
                forEachObjectEnumberableKey(a, self, this);
                /*</ltIE8>*/
            } else {
                self.call(this, a, b);
            }
            return this;
        };
    };

    Function.prototype.overloadGetter = function(usePlural){
        var self = this;
        return function(a){
            var args, result;
            if (typeof a != 'string') args = a;
            else if (arguments.length > 1) args = arguments;
            else if (usePlural) args = [a];
            if (args){
                result = {};
                for (var i = 0; i < args.length; i++) result[args[i]] = self.call(this, args[i]);
            } else {
                result = self.call(this, a);
            }
            return result;
        };
    };

这两个函数，看起来好像没啥，不是什么饱含各种 JS 奇技淫巧的函数，但是却是 Mootools 一切魔法的开始。这两个函数是典型的 JS curry 化的过程，前者 `overloadSetter` 使得 `Function` 传入一个 `object` 的参数并成为可能，即是原来这样调用 `func(a, b)` 的函数，`overloadSetter` 之后，不用修改原来的函数书写的实现，这样调用 `func({a: b, a1: b1})` 成为了可能；后者 `overloadGetter` 则使得 `Function` 传入一个 `array` 的数组并执行函数，返回一个函数执行后结果数组，这个等于封装了 Function 内部调用 `arguments` 这个参数列表的实现，而且内部返回的执行结果的列表。

简单看来，这两个函数并无出彩之处，就是扩展一下 JS 原生的 `Function` 在处理函数参数的时候形式，后者对于 `arguments` 的处理更加是略显多余，因为 JS 函数内部的 `arguments` 几乎等同就是一个 array 来看待的。是的，我给出这样的判断很正确，因为 `overloadSetter` 这个原型方法在框架内部被大量调用，而 `overloadGetter` 方法被调用的次数则少得可怜，而次数少得可怜那些调用则显得可有可无。

基于前面的基础，Mootools 继续扩展 `Function` 的原生方法，也是 Mootools 整个框架的关键函数和关键的代码组织的开始。

    Function.prototype.extend = function(key, value){
        this[key] = value;
    }.overloadSetter();

    Function.prototype.implement = function(key, value){
        this.prototype[key] = value;
    }.overloadSetter();

很简单的两个函数，`extend` 的调用使得 `Function` 能通过 `Function.extend(example, func)` 增加 `Function.example` 这样的静态方法，由于 `overloadGetter()` 的存在也可以 通过 `Function.extend({example: func})` 这样的方法来添加。

而 `implement` 的调用，则是扩展 `Function` 的原型链方法，即实例方法。

这两个 `Function` 原生方法实现，是 Mootools 后面构建整个框架，多次不断地重写 `extend` 和 `implement` 实现的根本基础。可以这样说，从根本上看来，这里的实现，与 jQuery 里面的 `jQuery.extend = jQuery.fn.extend = function() {}` 的实现，并无什么质的区别，唯一的区别就是，Mootools 用了两个名字去区分，而 jQuery 只是内部巧妙地使用了 `this` 关键字去区分谁调用就丢到谁头上而已。`overloadSetter()` 的使用也就是增加函数参数多种类型传入实现的可能吧。

殊途同归，JavaScript 里面的框架也概莫能外，说到根源，其实都是一个卵样。


### Mootools 的类型系统及其对  JS 原生类型的扩展

很有意思的是，很多年前，Mootools 的内部的类型系统的基类叫 `Native` 而现在改叫做 `Type`。说到 `Type` 这货，写 Python 这样的动态语言的同学可能都不用经过脑子都知道是什么意思，但是在这里，我仍然想给一个我觉得简单的理解，到底什么是 `Type`？

`Type` 顾名思义，就是类型，数组 array 是一种类型，字典 object 也是一种类型，布尔值 boolean 更是一种类型。在 JS 中，甚至 `Date` 日期这样的都是一种类型。那么结论显而易见，所谓类型，不过是数据实现的一种方式而已，而这种实现的方式，我们给它起个名字，这个名字就叫这种数据的 `Type`。

绕得有点远，回来 JavaScript 中来。在 JS 中谈类型是一件很困难的事情，为什么？常见的理由不外乎是当初设计成这样子的，或者是 JS 本来就是动态语言等等，这样理解有一定的理由，但是却不全对。JS 里面是有 `typeof` 这样的实现，然而这样的实现近乎残废，要不是后来发现的 `Object.prototype.toString()` 拯救了一下 JS 的类型系统，恐怕 JS 到目前来说，依然混乱不堪。深究其原因，很大程度因为 JS 语言本身是这样子的（语言体系原因），加上当初设计考虑不慎重而导致的。

在 JS 中由于奇技淫巧太多，类型不知所谓也不是什么大不了的事情，但是 Mootools 看来，或者从现在的 ECMAScript 规范发展看来，这样的不知所谓的类型，根本就是无法容忍的事情。

所以 Mootools 打算从上帝视觉来拯救 JS 的行动开始了。首先 Mootools 定义一个所有数据类型的基类，说是基类不是很准确，大概意思是这样吧，应该说定义了所有类型的父母亲。从这个所有数据类型的类型（是不是有点绕口）定义了所有 JS 的类型。

    var Type = this.Type = function(name, object){
        if (name){
            var lower = name.toLowerCase();
            var typeCheck = function(item){
                return (typeOf(item) == lower);
            };

            Type['is' + name] = typeCheck;
            if (object != null){
                object.prototype.$family = (function(){
                    return lower;
                }).hide();
                
            }
        }

        if (object == null) return null;

        object.extend(this);
        object.$constructor = Type;
        object.prototype.$constructor = object;

        return object;
    };

从源代码可以看到，所有的 `Type` 天生就会被带上 `Type.prototype.$family` 用于标示其类型，带上 `Type.$constructor` 和 `Type.prototype.$constructor` 用于标示其实例化的来源是谁。这个几乎就是 Mootools 的类型系统的根本和核心啦！

而接下来，Mootools 会对 JS 原生的 `String, Array, Number, Function, RegExp, Object, Date` 做系列的 `Type` 改造，并对原生的类型的静态方法和原型方法进行扩展。

### Mootools Class 的实现及其模块组织代码的方法

如果说前面的内容是 Mootools 自身对于 JavaScript 语言层面的修补，那么 Mootools Class 则是对 JS 进行模块化、面向对象化的一种理想化的实现。

熟悉 Java 这样的静态面向对象语言的同学，进入到 JS 的世界中，几乎都会在原型链的查找和继承这样的说法中迷失自我，而剔除 JS 原型链的使用，而将其用于模拟静态面向对象编程，Mootools 的 Class 就是其中之一的尝试。

    var Class = this.Class = new Type('Class', function(params){
        if (instanceOf(params, Function)) params = {initialize: params};

        var newClass = function(){
            reset(this);
            if (newClass.$prototyping) return this;
            this.$caller = null;
            this.$family = null;
            var value = (this.initialize) ? this.initialize.apply(this, arguments) : this;
            this.$caller = this.caller = null;
            return value;
        }.extend(this).implement(params);

        newClass.$constructor = Class;
        newClass.prototype.$constructor = newClass;
        newClass.prototype.parent = parent;

        return newClass;
    });

如果你在理解前面的 Mootools 的 `Type` 实现的基础上，还理解 JS 的原型链等一系列的 JS 基本概念，那么这段代码几乎不难理解。唯一你需要知道的就是 `params` 就是你构造的 `Class` 的内容，而这个 `Class` 的方法都是被 `implement` 放到 Class 的原型链方法上去的，等同于是类的实例方法。

而整个代码里，最关键的 `reset(this)` 正是断开前后 Class 原型链之间链接的关键实现，请看 `reset` 函数的实现代码

    var reset = function(object){
        for (var key in object){
            var value = object[key];
            switch (typeOf(value)){
                case 'object':
                    var F = function(){};
                    F.prototype = value;
                    object[key] = reset(new F);
                    break;
                case 'array': object[key] = value.clone(); break;
            }
        }
        return object;
    };

里面的代码简单吧？如果是 `Object` 那么不断地 `new Function()` 来复制，如果是 `Array` 则直接 deepcopy。

Mootools 自己内部实现得略微痛苦，就是为了给外部的实现和调用，能够正常地去模拟那种一本正经的面向对象的编程的实现，包括里面还有 `method.$protected` 和 `method.$hidden` 的属性，都是为了能够正常地去模拟面向对象编程，使得能够保护和隐藏方法不暴露在外。

对于 JS 和 Python 这样的动态语言，子类继承想要完全隐藏父类、保护父类的某些方法或者属性，几乎是不可能的事情，唯一能做的，就是定下一套规矩，这一点，对于 Mootools 来说，无疑的是成功的。

既然 Mootools 搞出了这个 Class，那么毫无疑问的，其插件和后续的代码封装，基本都是基于 Class 来实现的。这点围观一下 Mootools 的代码即可。

### Mootools 框架实现之美

前面说了这么多，其实已经说明了 Mootools 的一切了，重复这个论调，无非是想做个总结吧。Mootools 扩展了 JS 原生的类型和方法，更多地像是在修复 JS 语法层面的不一致；而 Class 的实现更加是 Mootools 给各种打算用 JS 面向对象编程的同学的语法糖。

代码实现层面，Mootools 并没有太多对于 JavaScript 本身太精巧的实现，而对 `Function` 静态方法和原型链方法的 `extend` 更加是与 jQuery 殊途同归。

但不可否认的是，Mootools 在这两方面的努力及其修复 JS 语法层面的不一致的前瞻性，纵然背负着扩展原生类型的罪名，但是要知道，原生类型的方法调用，本来就应该是 JS 这语言尤为出彩的地方，而不应该将其看成负担。


## YUI 框架

说起 [YUI](https://yuilibrary.com/) 大概无人不知其是 Yahoo 前端团队开发的框架，与前面的 jQuery 和 Mootools 不同，jQuery 和 Mootools 可以看作是一个 JS 函数集，一个好用的小扳手，一个好用的螺丝刀，而 YUI 却真正算是一个框架，一个库（library）。

其实同时代的，与 YUI 极其相似的，大而全的框架和库，其实不只是 YUI 一个，很多埋没在时代的潮流中被人忘记的大而全的框架还有很多，比如 [Dojo](https://github.com/dojo/dojo)，这个框架的代码开发还在持续更新。但是我仍然选择已经停止更新的 YUI 库来说明，其原因在于 YUI 在那些年代表的先进的技术理念和模块化的思路。

截至到 2018-08-30 本文写就之际，YUI 的官网和其 Github 仓库都是停止更新的，我好奇跑去 [Yahoo](https://yahoo.com) 的首页围观了一下他们的代码，发现其前端 JS 的实现却依然是 YUI 的，略为尴尬。


### YUI 实现基础

前面说了，YUI 与 jQuery、Mootools 这样的工具函数集不一样的地方，在于 YUI 几乎就是前端 JS 模块化的工程实践，在 JavaScript 还在刀耕火种的年代，提出这样的理念和实现，无疑是领先整个业界几个光年的。YUI 的工程化实践的理念，在 JS 的发展潮流中影响颇深。

YUI 在整个框架的设计之初就引入了模块化引用和模块化加载的实现，并且引入了命名空间（namespacing ）的概念。整个 YUI 框架的骨架，其实特别简单

    if (typeof YUI != 'undefined') {
        YUI._YUI = YUI;
    }

    var YUI = function() {};

    YUI.prototype = {};

而其对于命名空间（namespacing ）的实现，也不过是将 YUI 这个全局的空间，搞成和 window 一样的全局空间而已。其实刚刚开始的时候，这个函数还不是叫 `namespace` 这么高大上的，就是屌丝的命名 `YAHOO.register()`

    namespace: function() {
        var a = arguments, o, i = 0, j, d, arg;

        for (; i < a.length; i++) {
            o = this;
            arg = a[i];
            if (arg.indexOf(PERIOD) > -1) {
                d = arg.split(PERIOD);
                for (j = (d[0] == 'YAHOO') ? 1 : 0; j < d.length; j++) {
                    o[d[j]] = o[d[j]] || {};
                    o = o[d[j]];
                }
            } else {
                o[arg] = o[arg] || {};
                o = o[arg];
            }
        }
        return o;
    },

YUI 就是这样的实现基础，所以你看到 YUI 里面的代码，全部都是 `YUI.xxx.xxx` 这样的。

### YUI 模块化实现及按需加载

YUI 的模块化很有自己的一套，如果当年能再深入一下提出个模块的标准，目测后面就没有 AMD/CMD 这样的模块定义什么事了，将其按需加载的 loader 深入一下单独拿出来搞搞的，目测后面也就没有那一堆的 loader 什么事了。所以从这个角度来说，YUI 真是什么都有，就是什么都没有好好搞。

YUI 里面实现模块化的，除了前面说的命名空间（namespacing ）的实现，剩下的就是这两个函数的事情


    add: function(name, fn, version, details) {
        details = details || {};
        var env = YUI.Env,
            mod = {
                name: name,
                fn: fn,
                version: version,
                details: details
            },
            //Instance hash so we don't apply it to the same instance twice
            applied = {},
            loader, inst, modInfo,
            i, versions = env.versions;

        env.mods[name] = mod;
        versions[version] = versions[version] || {};
        versions[version][name] = mod;

        for (i in instances) {
            if (instances.hasOwnProperty(i)) {
                inst = instances[i];
                if (!applied[inst.id]) {
                    applied[inst.id] = true;
                    loader = inst.Env._loader;
                    if (loader) {
                        modInfo = loader.getModuleInfo(name);
                        if (!modInfo || modInfo.temp) {
                            loader.addModule(details, name);
                        }
                    }
                }
            }
        }

        return this;
    },
    use: function() {
        var args = SLICE.call(arguments, 0),
            callback = args[args.length - 1],
            Y = this,
            i = 0,
            name,
            Env = Y.Env,
            provisioned = true;

        // The last argument supplied to use can be a load complete callback
        if (Y.Lang.isFunction(callback)) {
            args.pop();
            if (Y.config.delayUntil) {
                callback = Y._delayCallback(callback, Y.config.delayUntil);
            }
        } else {
            callback = null;
        }
        if (Y.Lang.isArray(args[0])) {
            args = args[0];
        }

        if (Y.config.cacheUse) {
            while ((name = args[i++])) {
                if (!Env._attached[name]) {
                    provisioned = false;
                    break;
                }
            }

            if (provisioned) {
                if (args.length) {
                }
                Y._notify(callback, ALREADY_DONE, args);
                return Y;
            }
        }

        if (Y._loading) {
            Y._useQueue = Y._useQueue || new Y.Queue();
            Y._useQueue.add([args, callback]);
        } else {
            Y._use(args, function(Y, response) {
                Y._notify(callback, response, args);
            });
        }

        return Y;
    },

这个函数理解起来也简单到不得了，前者相当于是定义一个模块，后者就是调用这个已经定义好的模块。简单的使用例子如下

    YUI.add('module', function (Y) {
        Y.module = function () {
        };
    }, '3.4.0', {
        requires: ['yui-base']
    });

    YUI().use('module', function (Y) {
        module();
    });

这样的代码是不是很有 Node.JS 里面的 `define` 和 `require` 的感觉？目测你已经想到了，其实细化一下，这货真心可堪一用。然而，如此先进的理念，也随着 JavaScript 的发展潮流，被丢入了垃圾堆里。

其实回头翻看 YUI 实现的 YUI loader 的源代码，写得真是好啊，有 Queue 的实现，内含了 Module 的定义，有按需计算依赖的实现。只是这一切，现在看起来，都觉得很可惜。


### YUI 框架实现之美

YUI 这货既没有 jQuery 这样的小而美，也没有 Mootools 的上帝视觉，但其命名空间、模块化、按需加载的理念，堪称 JavaScript 工程化实践的先驱，这也是我特别要提及 YUI 的原因，从代码的组织和实现上说，YUI 的实现，已经打破了从代码语言细节的 `extend` 这样的实现，转而从前端代码工程学的角度，为代码的组织提供了新的做法和思路。这个已经足够它在 JS 界名垂青史了。

而且 YUI 对于工具化整合流程的推动，也是很有一套的，当年没有这么这么高大上的说法，那都是叫代码压缩。现在业界喜欢玩概念，把这些打包压缩的东西整合一下，叫“前端构建工具”，如果真不懂的小白，肯定被吓唬得一愣一愣的。如果你有仔细了解过代码压缩工具的历史，或者你会知道诞生于 2009年前端界还是刀耕火种年代的 [YUI Compressor](http://yui.github.io/yuicompressor/) 是多么的值得大书特书，这货当年是人手必备的，你能想象到的框架或者库啥的，当年都用过这货压缩过再发布！这个贡献，作为 YUI 工程化实践的一部分，真心功不可没。

当然了，从代码实现的细节来看，YUI 内部其实有个 `YUI.mix()` 函数，那也用得不是一般的频繁，其实质也是 `jQuery.extend()` 并没有什么质的区别。还是那句话，殊途但同归，概莫能外。


## Backbone 框架

现在搞前端，玩 JavaScript 的同学可能未必听过这个框架 [Backbone.JS](http://backbonejs.org/)，但是这货确确实实是后面一大波的号称 MVC JS 框架、MVVM JS 框架的鼻祖，其实现的 Events 事件触发的理念，真正契合了 JS 事件驱动那个的本质。

### Backbone 实现基础

Backbone 实现基础就是 Events 事件触发的理念，而这个事件触发的实现，全赖下面这段简单的代码。考虑到现在版本的 `Backbone.Events` 代码写得略为苦涩，当年 Backbone 的这部分代码写得尤为简洁。

    Backbone.Events = {
        bind : function(ev, callback, context) {
            var calls = this._callbacks || (this._callbacks = {});
            var list  = calls[ev] || (calls[ev] = []);
            list.push([callback, context]);
            return this;
        },
        unbind : function(ev, callback) {
            var calls;
            if (!ev) {
                this._callbacks = {};
            } else if (calls = this._callbacks) {
                if (!callback) {
                    calls[ev] = [];
                } else {
                    var list = calls[ev];
                    if (!list) return this;
                    for (var i = 0, l = list.length; i < l; i++) {
                        if (list[i] && callback === list[i][0]) {
                            list[i] = null;
                            break;
                        }
                    }
                }
            }
            return this;
        },
        trigger : function(eventName) {
            var list, calls, ev, callback, args;
            var both = 2;
            if (!(calls = this._callbacks)) return this;
            while (both--) {
                ev = both ? eventName : 'all';
                if (list = calls[ev]) {
                    for (var i = 0, l = list.length; i < l; i++) {
                        if (!(callback = list[i])) {
                            list.splice(i, 1); i--; l--;
                        } else {
                            args = both ? Array.prototype.slice.call(arguments, 1) : arguments;
                            callback[0].apply(callback[1] || this, args);
                        }
                    }
                }
            }
            return this;
        }

    };

这部分代码平淡无奇，就是模拟一个 Events 事件处理的过程，给某些操作加上 callback，然后在某些时候可以触发这些 callback。然而，Backbone 却让这段代码玩出了花！

### Backbone 事件驱动实现的前端 MVC 编程理念的转变

MVC 这个理念本来源自于后端服务器的逻辑开发，但是一直以来总有人想把这个搬到前端来实现，然而在 Backbone 出现之前，前端 MVC 的实现，实现得都不太漂亮，很多甚至是实现得很别扭，也很生硬。

Backbone 实现的前端 MVC 是有一定的历史原因的，原因就是 HTML5 history API 的出现和火热，但是这个原因也不靠谱，因为很久以前就有在 URL 打锚点实现页面路由的实现。

- **Model**：Backbone.Model 和 Backbone.Collection 还有 Backbone.sync 可以说是 Backbone 实现的 Model 层
- **View**：Backbone.View 就是 Backbone 实现的 View 层
- **Controller**：Backbone.Router 和 Backbone.history 可以看作是 Backbone 实现的 Controller 层

这就很明显标示出 Backbone 实现的 MVC 前端开发的编程理念。

前面说了 MVC 这样的编程理念注入到前端开发，并不是特别新奇的事情，MVC 三个层面的分离，在多数的编程条件下是有利于代码的解耦，实现调用的分离，这是很好的事情。但是 Backbone 开创的潮流远不在此，MVC 只是 Backbone 顺带实现的转变，但追究其根本的转变，在于其 Model 绑定到 View 显示的理念。

在 Backbone 之前，我们使用 jQuery 时常是这样写前端的代码的

    <!-- html -->
    <div id="example">Hello</div>

    // javascript
    $('#example').text('Hello, world');

考虑这么一种情况，如果你需要注入的 text 是不断被动态更新的，那么你需要手动编程进行事件的绑定，试想一下，页面上有成千上万这样的数据需要动态更新的呢？我是不是也需要手动去绑定事件？

而 Backbone 恰好带来了这样美好的实现，通过其事件 Events 驱动的 Model 能和 View 进行绑定，当 Model 变化的时候，View 也跟着变化，这样就省去大量的手动绑定事件的代码，并且能实现 MVC 编程进行代码上的解耦。

当然了，这只是理念的转变，事实上 backbone 做得是不够好的，由于 Backbone 实现的 Controller 层面几乎等同于没有，导致大量交互逻辑需要整合在 View 里面，并且，View 更新 Model 之后，Model 再更新 View 是需要自己手动处理的。这些都是 Backbone 没有那么美好的地方，但不妨外是它出现引领了这波 MVC JS 框架、MVVM JS 框架等一系列目前正热门的框架的潮流。


### Backbone 框架实现之美

Backbone 框架的实现，是需要严重依赖 [Underscore](https://underscorejs.org/) 的实现的，正是因为 Underscore 实现，才让 Backbone 的源代码得以如此的简洁、短小精悍。摊开来说，Backbone 实现的自由度是很高的，比如 View 部分的模板的处理，你是需要自行选择的，View 对 HTML DOM 部分的处理也是依赖于 jQuery 强大的实现的。

Backbone 这种站在巨人肩膀上的精神，我觉得很值得大家学习。不必每样东西都是自己造的，但是我能让其他人的东西在我的理念下都整合一起为我所用。很多年前国内很多喜欢折腾轮子的前辈看到我这句话，不知道有没有觉得惭愧一下。

Backbone 对于 JavaScript 工程化的实践，不仅仅是将 MVC 那么漂亮地实现到了前端，而且是它带来的这种数据绑定的理念、事件驱动的理念，深刻地且深远地影响了后面一票的 JS 框架，其影响至今都尚未见有淡化的迹象。


## React 框架


### React 实现基础


### React 框架实现之美
