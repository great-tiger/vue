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

​           this._initData()

_this._initEvents()

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

