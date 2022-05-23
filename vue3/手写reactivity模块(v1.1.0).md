# 手写reactivity模块(v1.1.0)

上一节的代码，存在很多问题，比如直接通过`effect`函数名来作为副作用函数收集到容器里，这种硬编码的方式很不灵活。
比如如果副作用函数名不叫`effect`，那么这段代码将不能正确地工作。
所以我们希望，不管副作用函数的名字叫什么，就算是个匿名函数，也能正确地被收集进容器里。
为了实现这一点，我们需要改一点代码，提供一个可以用来注册副作用函数的机制。

```javascript
<body></body>
<javascript>
// 存副作用的容器
const bucket = new Set()

// 一个全局变量储存被注册的副作用函数（v1.1.0 新增）
let activeEffect

// 原始数据
const data = { foo: 1 }
// 使用Proxy对原始数据进行代理，并返回代理对象
const obj = new Proxy(data, {
    // 拦截读取操作
    get(target, key){
        if(activeEffect){
            // 如果存在注册的副作用函数，则收集进容器
            bucket.add(activeEffect)
        }
        // 返回属性值
        return target[key]
    },
    // 拦截设置操作
    set(target, key, newValue){
        // 设置新属性值
        target[key] = newValue
        // 从容器内取出所有副作用函数，并执行
        bucket.forEach(fn => fn())
    }
})

// effect 函数用于注册副作用函数（v1.1.0 新增）
function effect(fn){
    activeEffect = fn
    fn()
}

// 执行effect 用匿名函数注册副作用函数
effect(() => {
    console.log('run effect')
    document.body.innerText = obj.foo
})

setTimeout(() => {
  obj.bar = '2'
}, 1000)
</javascript>
```

在谷歌浏览器里执行上面的代码，能看到页面一开始渲染为`1`，打印一次`run effect`，然后1秒后继续打印了一次`run effect`，代表副作用函数又执行了一次。
这就出问题了，我们注册的副作用函数并没和`obj.bar`建立关联，但1秒后设置的`obj.bar`值却又触发了次容器里所有的副作用函数的遍历执行。
出现这问题的根本原因是，我们使用了`Set`数据结构作为储存副作用函数的容器，导致副作用函数没和被操作对象的目标字段之间建立明确的联系。
例如现在，不管读取了对象的哪一个属性都会把副作用函数收集进容器，然后不管设置了哪个属性，都会遍历容器里的副作用函数并执行。
解决方法也很简单，要重新设计下容器的数据结构，不能仅仅使用一个`Set`。


```javascript
<body></body>
<javascript>
// 存副作用的容器（新增）
const bucket = new WeekMap()

// 一个全局变量储存被注册的副作用函数（v1.1.0 新增）
let activeEffect

// 原始数据
const data = { foo: 1 }
// 使用Proxy对原始数据进行代理，并返回代理对象
const obj = new Proxy(data, {
    // 拦截读取操作
    get(target, key){
        let depsMap = bucket.get(target)
        if(!depsMap){
            bucket.set(target,(depsMap = new Map()))
        }
        let deps = depsMap.get(key)
        if(!deps){
            deps.set(key, (deps = new Set()))
        }
        if(activeEffect){
            // 如果存在注册的副作用函数，则收集进容器
            deps.add(activeEffect)
        }
        // 返回属性值
        return target[key]
    },
    // 拦截设置操作
    set(target, key, newValue){
        // 设置新属性值
        target[key] = newValue
        // 从容器内取出所有副作用函数，并执行
        const depsMap = bucket.get(target)
        if(!depsMap) return
        const deps = depsMap.get(key)
        deps && deps.forEach(fn => fn())
    }
})

// effect 函数用于注册副作用函数（v1.1.0 新增）
function effect(fn){
    activeEffect = fn
    fn()
}

// 执行effect 用匿名函数注册副作用函数
effect(() => {
    console.log('run effect')
    document.body.innerText = obj.foo
})

setTimeout(() => {
  obj.foo = '2'
}, 1000)

setTimeout(() => {
  obj.bar = '3'
}, 2000)
</javascript>
```

在谷歌浏览器执行，能看到页面一开始渲染为`1`，打印一次`run effect`，然后1秒后继续打印了一次`run effect`，页面渲染为`2`，然后2秒后的`obj.bar = '3'`也不会造成副作用函数的再次执行。