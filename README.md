## 分析原则

抓住主要逻辑，放弃细枝末节

由粗及细，步步为营！

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
parse("a+b",{a:1,b:2})  =>  3
```
##batcher.js
这个模块中有两个队列
queue
userQueue
这两个队列 queue队列会先被执行

pushWatcher (watcher) 将 watcher 放入到队列，下一次事件循环时，执行watcher.run的方法

watcher 结构:
{
  id:...
  user:true||false  //true 放入到userQueue  false 放入到queue队列
  run:...
}

##cache.js

实现最近不常用算法
put(key,value)
get(key)


