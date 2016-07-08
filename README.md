## 分析原则

抓住主要逻辑，放弃细枝末节

由粗及细，步步为营！

##编译过程简述
查看dis/compilePath中的例子

## Vue 配置详解

```javascript
var obj=new Vue({
  el:"#root",//指定 Vue 管辖元素
  methods:{"key",fn},//注册方法到obj上,fn中的this就是obj
  events:{"key":fn},//注册事件
  watch:{"key",fn},//注册监听
  data:{//注册数据  访问方式:obj.message===obj._data.message
      message: 'Hello Vue.js!'
  }
});
```

## Dep

1、addSub、removeSub、notify、depend

2、该类虽然很简单，但是很重要

## 执行流程

call init hook

this._initState()

​	   this._initProps()

```javascript
options.el = query(el)     //选择器转成dom元素
this.propsUnlinkFn = el && el.nodeType === 1 && props?compileAndLinkProps(this, el, props, this._scope):null   //这里涉及到编译，暂时略过
```

​           this._initMeta()

​           this._initMethods()

```javascript
//这个太简单了，直接上代码吧
Vue.prototype._initMethods = function () {
      var methods = this.$options.methods;
      if (methods) {
        for (var key in methods) {
          this[key] = bind(methods[key], this);
        }
      }
};
```

​           this._initData()

```javascript
//options.data的处理，主要集中在这里
Vue.prototype._initData = function () {
    /*
        注意：这里的data已经在 _init 方法中处理了一次，入口代码：
        options = this.$options = mergeOptions(this.constructor.options, options,
        this);

        主要功能：
        this._proxy(key);
        observe(data, this);
      */
    this._proxy(key)
    observe(data, this)
}
```



_this._initEvents()

​           主要用来处理options中的events，watch字段

```javascript
Vue.prototype._initEvents = function () {
    var options = this.$options
    registerCallbacks(this, '$on', options.events)
    registerCallbacks(this, '$watch', options.watch)
  }
```



call created hook

## Vue.prototype._proxy

 Proxy a property, so that vm.prop === vm._data.prop

通过get，set 代理 vm 上的属性 到 vm._data

这也就解释了

```javascript
 var vue = new Vue({
        data: {
            message: 'Hello Vue.js!'
        }
 });
 console.log(vue);
 console.log(vue.message===vue._data.message);  //true
```

## Observer

该类的作用：重写对象的get，set。当数据发生变化时，可以向订阅者发送数据变更通知。订阅者，实质上是Watcher对象，下面会涉及到。

下面的部分代码存在疑问？？？？？

```javascript
export function defineReactive (obj, key, val) {
  var dep = new Dep()

  var property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  var getter = property && property.get
  var setter = property && property.set

  var childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        //主要是下面的代码理解不了  ？？？？？？
        if (childOb) {
          childOb.dep.depend()
        }
        if (isArray(value)) {
          for (var e, i = 0, l = value.length; i < l; i++) {
            e = value[i]
            e && e.__ob__ && e.__ob__.dep.depend()
          }
        }
        //主要是上面的代码理解不了  ？？？？？？
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val
      if (newVal === value) {
        return
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = observe(newVal)
      dep.notify()
    }
  })
}
```

## parsers/expression

实现的实质是通过new Function 构造代码

主要功能，可以用下面的代码表示：

```javascript
parse("a.b") => {get:..,set:...}
```
##batcher.js
这个模块中有两个队列
queue
userQueue
这两个队列 queue队列会先被执行

pushWatcher (watcher) 将 watcher 放入到队列，下一次事件循环时，执行watcher.run的方法

watcher 结构:
```javascript
{
  id:...
  user:true||false  //true 放入到userQueue  false 放入到queue队列
  run:...
}
```

##cache.js

实现最近不常用算法
```javascript
put(key,value)
get(key)
```

##watch.js
```javascript
//Vue 对外接口
vm.$watch('a.b.c', function (newVal, oldVal) {
  // 做点什么
})
//内部实现
Vue.prototype.$watch = function (expOrFn, cb, options) {
    var vm = this
    var watcher = new Watcher(vm, expOrFn, cb, {
      deep: options && options.deep,
      sync: options && options.sync,
      filters: parsed && parsed.filters,
      user: !options || options.user !== false
    })
    //取消观察函数
    return function unwatchFn () {
      watcher.teardown()
    }
  }
```
可以看到,$watch内部就是调用的Watcher，下面就具体关注一下watcher.js的实现吧
先看一下构造函数：
```javascript
//保留主要代码 假定expOrFn 为 表达式
export default function Watcher (vm, expOrFn, cb, options) {

  this.vm = vm
  vm._watchers.push(this)

  this.expression = expOrFn
  this.cb = cb
  this.id = ++uid // uid for batching

  var res = parseExpression(expOrFn, this.twoWay)
  this.getter = res.get
  this.setter = res.set

  this.value =this.get()

}
```
通过看上面的代码，也没有发现什么神奇的地方。就是一些初始化工作。接着注意一下$watch的实现，就是调用了一下new Watcher实例化了一个对象，怎么就可以监控变化了呢？玄机在哪里呢？
接着我们看this.get(),对了，上源码
```javascript
//Evaluate the getter, and re-collect dependencies.
Watcher.prototype.get = function () {
  this.beforeGet() //   Dep.target = this
  var scope = this.vm
  var value = this.getter.call(scope, scope)   //这里会调用被监控属性的get方法，然后在get方法中触发watcher的addDep方法，玄机就在这里。具体查看addDep源码很简单。
  this.afterGet() //Clean up for dependency collection.
  return value
}
```
```javascript
//watcher dep 关系  互相保存对方的引用
Watcher.prototype.addDep = function (dep) {
  var id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```

下面看一下依赖收集完成后，做了点什么？
```javascript
Watcher.prototype.afterGet = function () {
  Dep.target = null
  var i = this.deps.length
  while (i--) {
    var dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  /*
      下面主要功能，保存新依赖，清理就依赖：
      this.depIds = this.newDepIds
      this.newDepIds.clear()

      this.deps = this.newDeps
      this.newDeps.length = 0
  */
  var tmp = this.depIds
  this.depIds = this.newDepIds
  this.newDepIds = tmp
  this.newDepIds.clear()
  tmp = this.deps
  this.deps = this.newDeps
  this.newDeps = tmp
  this.newDeps.length = 0
}
```
通过读代码发现，afterGet主要是对deps中保存的dep进行处理。
如果在newDeps中已经没有dep信息了，则从该dep中移除自己。
为什么要这么做，看到这里还不是很明朗，待以后发现原因吧！

如果我们监视的属性被赋值时，会触发dep.notify方法，我们的watcher就会收到通知，触发update方法。下面看一下update方法。


##$mount
这个过程就比较复杂了

第一步编译：this._compile(el);
```javascript
this._initElement(el)  // vm.$el = el  并且  this._callHook('beforeCompile');
this._unlinkFn = compile(el, options)(this, el) //主要功能集中在这里了
this._isCompiled = true;
this._callHook('compiled');
```
compile
```javascript
function compile(el, options, partial) {
    var nodeLinkFn =compileNode(el, options); //编译自己
    var childLinkFn = compileNodeList(el.childNodes, options); //如果有孩子则编译孩子

    //返回link函数
    return function compositeLinkFn(vm, el, host, scope, frag) {
      // cache childNodes before linking parent, fix #657
      var childNodes = toArray(el.childNodes);
      // link
      var dirs = linkAndCapture(function compositeLinkCapturer() {
        if (nodeLinkFn) nodeLinkFn(vm, el, host, scope, frag);
        if (childLinkFn) childLinkFn(vm, childNodes, host, scope, frag);
      }, vm);
      return makeUnlinkFn(vm, dirs);
    };
  }
```
compileNodeList
代码就不往外粘了，就是递归遍历每一个节点调用compileNode

下面看 compileNode
```javascript
  function compileNode(node, options) {
    var type = node.nodeType;
    if (type === 1 && !isScript(node)) {
      return compileElement(node, options);
    } else if (type === 3 && node.data.trim()) {
      return compileTextNode(node, options);
    } else {
      return null;
    }
  }
```
逻辑很清楚，如果是元素就调用compileElement编译，如果是TEXT_NODE则用compileNode编译。
先易后难吧，我们先看compileTextNode,用下面的实例走一下流程
```html
<div id="app">
  {{ message }}
</div>
```
```javascript
new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue.js!'
  }
})
```
```
function compileTextNode(node, options) {
    /*
            [{
                "value": "\n  "
            }, {
                "tag": true,
                "value": "message",
                "html": false,
                "oneTime": false
            }, {
                "value": "\n"
            }]
    */
    var tokens = parseText(node.wholeText);
    //根据token构造文档片段，当中重要的是 processTextToken
    var frag = document.createDocumentFragment();
    var el, token;
    for (var i = 0, l = tokens.length; i < l; i++) {
      token = tokens[i];
      el = token.tag ? processTextToken(token, options) : document.createTextNode(token.value);
      frag.appendChild(el);
    }
    //返回link函数
    return makeTextNodeLinkFn(tokens, frag, options);
  }

   //主要作用是：为token增加descriptor
   function processTextToken(token, options) {
      var el;
      if (token.oneTime) {
        el = document.createTextNode(token.value);
      } else {
        if (token.html) {
          el = document.createComment('v-html');
          setTokenType('html');
        } else {
          // IE will clean up empty textNodes during
          // frag.cloneNode(true), so we have to give it
          // something here...
          el = document.createTextNode(' ');
          setTokenType('text');
        }
      }
      function setTokenType(type) {
        if (token.descriptor) return;
        /*parsed 结果 {expression: "message"}*/
        var parsed = parseDirective(token.value);
        /*指令描述符 new Directive(descriptor,......)*/
        token.descriptor = {
          name: type,
          def: directives[type],
          expression: parsed.expression,
          filters: parsed.filters
        };
      }
      return el;
    }
    //构造link函数并返回。link函数执行的时候，tokens很轻松地可以访问到。(闭包)
    function makeTextNodeLinkFn(tokens, frag) {
        return function textNodeLinkFn(vm, el, host, scope) {
          var fragClone = frag.cloneNode(true);
          var childNodes = toArray(fragClone.childNodes);
          var token, value, node;
          for (var i = 0, l = tokens.length; i < l; i++) {
            token = tokens[i];
            value = token.value;
            if (token.tag) {
              node = childNodes[i];
              if (token.oneTime) {
                value = (scope || vm).$eval(value);
                if (token.html) {
                  replace(node, parseTemplate(value, true));
                } else {
                  node.data = _toString(value);
                }
              } else {
                /*绑定指令，通过指令实现双向绑定  也就是说{{}}绑定，最终会被转换成指令*/
                vm._bindDir(token.descriptor, node, host, scope);
              }
            }
          }
          replace(el, fragClone);
        };
      }
      //文本节点的处理，分析完成
      //对于文本节点来说，编译阶段的成果就是tokens,frag，以闭包的形式传给了link函数
```
下面我们看看，是如何编译元素节点的。compileElement
```javascript
//测试用例
<div id="app">
  <img v-bind:src="message" width="30px" />
</div>

new Vue({
  el: '#app',
  data: {
    message: 'https://www.google.com.hk/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png'
  }
})
```
```javascript
//编译指令，返回linkFn
function compileElement(el, options) {
    var linkFn;
    var hasAttrs = el.hasAttributes();
    var attrs = hasAttrs && toArray(el.attributes);
    // check terminal directives (for & if)
    if (hasAttrs) {
      linkFn = checkTerminalDirectives(el, attrs, options);
    }
    // check element directives  处理元素指令，此例中略过
    if (!linkFn) {
      linkFn = checkElementDirectives(el, options);
    }
    // check component 处理组件，此例中略过
    if (!linkFn) {
      linkFn = checkComponent(el, options);
    }
    // normal directives 此例中，是通过下面处理的
    if (!linkFn && hasAttrs) {
      linkFn = compileDirectives(attrs, options);
    }
    return linkFn;
  }


  function compileDirectives(attrs, options) {
    var i = attrs.length;
    var dirs = [];
    var attr, name, value, rawName, rawValue, dirName, arg, modifiers, dirDef, tokens, matched;
    while (i--) {
      attr = attrs[i];
      name = rawName = attr.name;
      value = rawValue = attr.value;
      tokens = parseText(value);
      // reset arg
      arg = null;

      // attribute interpolations
      if (tokens) {
         // 因为tokens为null，略过
      } else

        // special attribute: transition
        if (transitionRE.test(name)) {
            //跳过
        } else

          // event handlers
          if (onRE.test(name)) {
            //跳过
          } else

            // attribute bindings
            // var bindRE = /^v-bind:|^:/;
            if (bindRE.test(name)) {
              dirName = name.replace(bindRE, '');
              if (dirName === 'style' || dirName === 'class') {
                pushDir(dirName, internalDirectives[dirName]);
              } else {
                arg = dirName;
                pushDir('bind', directives.bind);
              }
            } else

              // normal directives
              if (matched = name.match(dirAttrRE)) {
                //跳过
              }
    }

    /**
     * Push a directive.
     *
     * @param {String} dirName
     * @param {Object|Function} def
     * @param {Array} [interpTokens]
     */

    function pushDir(dirName, def, interpTokens) {
      var hasOneTimeToken = interpTokens && hasOneTime(interpTokens);
      var parsed = !hasOneTimeToken && parseDirective(value);
      dirs.push({
        name: dirName,
        attr: rawName,
        raw: rawValue,
        def: def,
        arg: arg,
        modifiers: modifiers,
        // conversion from interpolation strings with one-time token
        // to expression is differed until directive bind time so that we
        // have access to the actual vm context for one-time bindings.
        expression: parsed && parsed.expression,
        filters: parsed && parsed.filters,
        interp: interpTokens,
        hasOneTime: hasOneTimeToken
      });
    }

    //其实上面在做一件事 调用pushDir收集指令
    if (dirs.length) {
      return makeNodeLinkFn(dirs);
    }
  }
  //直接返回linkFn。通过这里可以看出编译阶段主要做了一件事 就是收集指令 传给linkFn
  function makeNodeLinkFn(directives) {
      return function nodeLinkFn(vm, el, host, scope, frag) {
        // reverse apply because it's sorted low to high
        var i = directives.length;
        while (i--) {
          //绑定指令到元素
          vm._bindDir(directives[i], el, host, scope, frag);
        }
      };
  }

  //对于元素节点的编译分析完成
  //通过上面的分析发现，linkFn中都是通过调用vm._bindDir实现指令与元素关联的。下面我们可以分析一下vm._bindDir
```
```javascript
Vue.prototype._bindDir = function (descriptor, node, host, scope, frag) {
      this._directives.push(new Directive(descriptor, this, node, host, scope, frag));
};
//通过看代码，我们发现真正的魔法，在Directive中。具体参考Directive解析吧。

//总体来说一下compile-->>link 
//compile 收集指令 link 主要是生成scope 调用vm._bindDir
//通过看Directive中的解析我们发现vm._bindDir, 除了push了一个指令实例外，其他的什么都没做。指令到底什么时候起的作用呢？
//我们还得把眼光集中到compile中的linkAndCapture
//在这个方法中分别调用了指令对象的_bind方法。指令这样才起到了真正的作用。
function linkAndCapture(linker, vm) {
    linker();
    var dirs = vm._directives.slice(originalDirCount);
    dirs.sort(directiveComparator);
    for (var i = 0, l = dirs.length; i < l; i++) {
      dirs[i]._bind();
    }
    return dirs;
  }


```
##Directive.js
```
指令对象暴露的一些有用的属性 <div id="demo" v-demo:hello.a.b="msg"></div>
el: 指令绑定的元素。
vm: 拥有该指令的上下文 ViewModel。
expression: 指令的表达式，不包括参数和过滤器。这个表达式用于watch
arg: 指令的参数。 hello
name: 指令的名字，不包含前缀。demo
modifiers: 一个对象，包含指令的修饰符。{"b":true,"a":true}
descriptor: 一个对象，包含指令的解析结果。 指令的描述对象，包括expression name arg modifiers filters等
```
```javascript
//指令对象的构造函数，就是用来初始化的，别的什么都没做
export default function Directive (descriptor, vm, el, host, scope, frag) {
  this.vm = vm
  this.el = el
  // copy descriptor properties
  this.descriptor = descriptor
  this.name = descriptor.name
  this.expression = descriptor.expression
  this.arg = descriptor.arg
  this.modifiers = descriptor.modifiers
  this.filters = descriptor.filters
  this.literal = this.modifiers && this.modifiers.literal
  // private
  this._locked = false
  this._bound = false
  this._listeners = null
  // link context
  this._host = host
  this._scope = scope
  this._frag = frag
}


Directive.prototype._bind = function () {
  var name = this.name
  var descriptor = this.descriptor
  //指令的定义对象 {bind:...,update:...,unbind:...}
  var def = descriptor.def
  extend(this, def)
  // 调用指令定义对象的bind方法
  // initial bind
  if (this.bind) {
    this.bind()
  }

  // wrapped updater for context
  var dir = this
  if (this.update) {
    this._update = function (val, oldVal) {
      if (!dir._locked) {
        dir.update(val, oldVal)
      }
    }
  }
  
  //监控的表达式 与 watcher联系起来了。值更新会调用this._update方法。
  //魔法归于Watcher了
  var watcher = this._watcher = new Watcher(
    this.vm,
    this.expression,
    this._update, // callback
    {
      filters: this.filters,
      twoWay: this.twoWay,
      deep: this.deep,
      preProcess: preProcess,
      postProcess: postProcess,
      scope: this._scope
    }
  )

  this.update(watcher.value)
}

```
##parses/path.js
```javascript
parsePath 对path进行解析，输出是一个数组
parsePath('a.b') -->> ['a', 'b']
parsePath('a[0][1]') -->> ['a', '0', '1']

getPath典型应用
it('get', function () {
    var path = 'a[\'b"b"c\'][0]'
    var obj = {
      a: {
        'b"b"c': [12345]
      }
    }
    expect(Path.getPath(obj, path)).toBe(12345)
    expect(Path.getPath(obj, 'a.c')).toBeUndefined()
})

setPath典型应用
it('set', function () {
    var path = 'a.b.c'
    var obj = {
      a: {
        b: {
          c: null
        }
      }
    }
    var res = Path.setPath(obj, path, 12345)
    expect(res).toBe(true)
    expect(obj.a.b.c).toBe(12345)
})

```
##组件
```
 var MyComponent=Vue.extend({
   template:'<div>A custom component</div>'
 });
 Vue.component('my-component',MyComponent);
 var vue=new Vue({
   el:"#app"
 });

 另外文档中介绍还有一种简洁的写法
 Vue.component('my-component',{
   template:'<div>A custom component</div>'
 });

 那么两种写法是怎样转换的呢？

```

```javascript
//_assetTypes: ['component', 'directive', 'elementDirective', 'filter', 'transition', 'partial'],
config._assetTypes.forEach(function (type) {
      Vue[type] = function (id, definition) {
        if (!definition) {
          //获取组件
          return this.options[type + 's'][id];
        } else {
          //如果是便捷写法的话 调用Vue.extend方法
          if (type === 'component' && isPlainObject(definition)) {
            if (!definition.name) {
              definition.name = id;
            }
            definition = Vue.extend(definition);
          }
          //放置到this.options上
          this.options[type + 's'][id] = definition;
          return definition;
        }
      };
    });
//通过上面的代码，我们知道两种写法之间的关系。我们继续看组件的实现 Vue.extend 方法

//Vue中实现继承的方法 
//官网api中,介绍
//创建基础 Vue 构造器的“子类”。参数是一个对象，包含组件选项。
//这里要注意的特例是 el 和 data 选项在 Vue.extend() 中它们必须是函数。
Vue.extend = function (extendOptions) {
      //下面有两个地方不太了解 mergeOptions ,另外一个就是拷贝_assetTypes。
      //个人觉得没有必要
      extendOptions = extendOptions || {};
      var Super = this;
     
      var name = extendOptions.name || Super.options.name;
      var Sub = createClass(name || 'VueComponent');
      Sub.prototype = Object.create(Super.prototype);
      Sub.prototype.constructor = Sub;
      Sub.cid = cid++;
      //options 的定义在 installGlobalAPI 中
      /**
       * Vue and every constructor that extends Vue has an
       * associated options object, which can be accessed during
       * compilation steps as `this.constructor.options`.
       *
       * These can be seen as the default options of every
       * Vue instance. 说的挺清楚，就不翻译了
       * 查看一下代码，发现options上保存了内置的指令，过滤器
       */
      Sub.options = mergeOptions(Super.options, extendOptions);
      Sub['super'] = Super;
      // allow further extension
      Sub.extend = Super.extend;
      // create asset registers, so extended classes
      // can have their private assets too.
      // 为什么需要拷贝呢，真是理解不了
      config._assetTypes.forEach(function (type) {
        Sub[type] = Super[type];
      });
     
      return Sub;
};
//其中createClass
function createClass(name) {
  return new Function('return function ' + classify(name) + ' (options) { this._init(options) }')();
}

//通过上面的代码，我们知道,原来component,就是一个Vue的子类
//生成的组件最终被放到了this.options['components']中
//接着分析，那么生成的组件，是怎么起作用的呢？通过组件的使用方式<my-component></my-component>
//我们可以猜到肯定与编译有关系，顺着这个思路。我们在compileElement中查到了相应的分支。
function compileElement(el, options) {
    // check component
    if (!linkFn) {
      linkFn = checkComponent(el, options);
    }
    return linkFn;
}

 function checkComponent(el, options) {
    var component = checkComponentAttr(el, options);
    if (component) {
      var ref = findRef(el);
      var descriptor = {
        name: 'component',
        ref: ref,
        expression: component.id,
        def: internalDirectives.component,
        modifiers: {
          literal: !component.dynamic
        }
      };
      var componentLinkFn = function componentLinkFn(vm, el, host, scope, frag) {
        if (ref) {
          defineReactive((scope || vm).$refs, ref, null);
        }
        vm._bindDir(descriptor, el, host, scope, frag);
      };
      componentLinkFn.terminal = true;
      return componentLinkFn;
    }
  }
  //在上面这个方法中，在link函数中，没有发现什么特别的。
  //在编译阶段，internalDirectives.component 应该是真正的魔法。
  //真正的魔法在内置的组件指令定义中，下面仔细看一下internalDirectives.component
```
##mergeOptions
```
文档中说的很清楚，使用在extend 和 _init中。搜索了一下发现 Vue.mixin 也是使用的它
Merge two option objects into a new one.
Core utility used in both instantiation and inheritance.
下面看一下它的具体实现
```
```javascript
//注意返回的是一个新的option对象
export function mergeOptions (parent, child, vm) {
  guardComponents(child)
  guardProps(child)
  if (process.env.NODE_ENV !== 'production') {
    if (child.propsData && !vm) {
      warn('propsData can only be used as an instantiation option.')
    }
  }
  var options = {}
  var key
  if (child.extends) {
    parent = typeof child.extends === 'function'
      ? mergeOptions(parent, child.extends.options, vm)
      : mergeOptions(parent, child.extends, vm)
  }
  if (child.mixins) {
    for (var i = 0, l = child.mixins.length; i < l; i++) {
      var mixin = child.mixins[i]
      var mixinOptions = mixin.prototype instanceof Vue
        ? mixin.options
        : mixin
      parent = mergeOptions(parent, mixinOptions, vm)
    }
  }
  //defaultStrat 默认策略,如果孩子存在则用孩子的，否则用父亲的。
  //注意不是覆盖啊,这和以前接触的处理options的方式是不同的。
  //假如采用默认策略defaultStrat的话，拷贝孩子没有的东西给孩子，孩子有的就不管了。
  //接着我们看看默认的合并策略都有什么，真不少啊。特殊处理的key有components computed 等等。
  //这些信息被保存到了Vue.config.optionMergeStrategies中。具体情况具体分析吧。
  //下面看看对components如何处理的，读源码发现，绕来绕去，还是为了实现访问的时候，孩子有的就用孩子的
  //否则就用父亲的
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {//避免key的重复处理
      mergeField(key)
    }
  }
  function mergeField (key) {
    var strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```
##组件杂想

###组件的定义及存储位置

```
说到组件不可不提到Vue.extend
文档说Vue.extend是用来创建Vue构造器的子类，查过源码之后发现，就是继承。
组件部分又说Vue.extend用来创建一个组件构造器，也就是说Vue的子类可以当做组件构造器。
文档中提示，如果想把这个构造器用作组件，需要用Vue.component(tag, constructor)注册。
```
```javascript
//Vue.component的代码，我们再粘一次
config._assetTypes.forEach(function (type) {
      Vue[type] = function (id, definition) {
        if (!definition) {
          //获取组件
          return this.options[type + 's'][id];
        } else {
          //如果是便捷写法的话 调用Vue.extend方法
          if (type === 'component' && isPlainObject(definition)) {
            if (!definition.name) {
              definition.name = id;
            }
            definition = Vue.extend(definition);
          }
          //放置到this.options上
          this.options[type + 's'][id] = definition;
          return definition;
        }
      };
});
//发现 组件构造器 被放到了this.options["components"]上
```
```javascript
//定义组件方式举例
var MyComponent = Vue.extend({
  template: '<div>A custom component!</div>'
})

// 注册
Vue.component('my-component', MyComponent)

//定义组件的便捷方式，语法糖。Vue.component源码中有所体现
//在Vue.component中封装了扩展与注册，本段代码与上面的定义
//方式是等价的
Vue.component('my-component', {
  template: '<div>A custom component!</div>'
})

//上面这两种写法，component到底被放到哪里了。
//实际上是放到Vue.options上了，MyComponent.options上没有
```

```javascript
//文档中还提到了局部注册
var Child = Vue.extend({ /* ... */ })

var Parent = Vue.extend({
  template: '...',
  components: {
    // <my-component> 只能用在父组件模板内
    'my-component': Child
  }
})

//语法糖写法 局部注册也可以这么做
var Parent = Vue.extend({
  components: {
    'my-component': {
      template: '<div>A custom component!</div>'
    }
  }
})


//我们拿下面的代码分析一下源码
var Parent = Vue.extend({
  components: {
    'my-component': {
      template: '<div>A custom component!</div>'
    }
  }
})


/*
Vue.extend({})  生成一个组件构造器或者说一个Vue子类
关注一下 对 components 字段是如何处理

mergeOptions

guardComponents(child); 作用
保证components转换成如何形式，语法糖的拆解吗
{
   'my-component':组件构造器
}

合并components
算法是用的
//parentVal={}；
//childVal={'my-component':组件构造器}
function mergeAssets(parentVal, childVal) {
    var res = Object.create(parentVal || null);
    return childVal ? extend(res, guardArrayAssets(childVal)) : res;
}
最终合并后的结果，保存到options
最后mergeOptions返回新的options
options["components"]={'my-component':组件构造器}--__protp__-->>parentVal

最最后
我们发现按照上面的方式注册组件被保存到了
Parent.options上
结构
options["components"]={'my-component':组件构造器}--__protp__-->>parentVal

*/
```
