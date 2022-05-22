# 手写reactivity模块(v1.0.0)

响应式数据：能检测数据的操作，并做出响应的数据，在vue3中使用proxy创建。
副作用函数：执行后能改变外部状态的函数，比如渲染函数执行后挂载了dom元素。
下面是最基本的案例

```javascript
<body></body>
<javascript>
// 存副作用的容器
const bucket = new Set()

// 原始数据
const data = { foo: 1 }
// 使用Proxy对原始数据进行代理，并返回代理对象
const obj = new Proxy(data, {
    // 拦截读取操作
    get(target, key){
        // 将副作用函数添加进容器内
        bucket.add(effect)
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

function effect(){
    document.body.innerText = obj.foo
}

// 执行副作用函数
effect()

setTimeout(() => {
  obj.foo = '2'
}, 1000)
</javascript>
```

在谷歌浏览器执行，能看到页面一开始渲染为`1`，1秒后执行`obj.foo = '2'`并页面也渲染出`2`。