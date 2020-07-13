只有刻意练习才能提高。为了更好的理解Vue3源码我计划使用渐进式的方法完成一个简写版的Vue框架。

## 写作计划
欢迎大家持续关注、计划肯定会变的越来越好。

- Step00 NoMVVM
- Step01 MVVM(Mock版)
- Step02 编译函数(Mock)
- Step03 数据劫持(精简版)
    - defineProperty Vue2.0
    - Proxy/Reflect Vue3.0
- Step04 编译、渲染(Mock版)
- Step05 (Ast、Transform、Generate)、渲染(精简版)
- Step06 自定义渲染器(Mock版)
- Step07 Dom Diff原理

[完整的代码](https://github.com/su37josephxia/vue3-study/tree/master/tiny-vue)

## Step00 NoMVVM

![](https://user-gold-cdn.xitu.io/2020/7/8/1732c3aa1015b651?w=330&h=126&f=gif&s=20064)

想象一下如果没有MVVM框架我们要怎么实现一个这样的功能。
- 创建一个数据模型
```js
const data = {
        message: 'Hello Vue 3!!'
    }
```
- 创建一个视图
```js
<div id='app'>
    <input />
    <button></button>
</div>
```
- 创建一个将模型数据更新到视图上的渲染函数 
```js
function update() {
    // 更新视图
    document.querySelector('button').innerHTML = data.message
    document.querySelector('input').value = data.message
}

```
- 执行首次数据更新
```js
// 首次数据渲染
update()
```
- 绑定按钮点击事件
    - 修改模型中数据： 反转字符串
    - 修改模型后重新渲染数据
```js
document.querySelector('button').addEventListener('click', function () {
    data.message = data.message.split('').reverse().join('')
    update()
})
```
- 对输入项变化进行监听
    - 数据项变化时修改模型中数据
    - 修改模型后重新渲染数据
```js
document.querySelector('input').addEventListener('keyup', function () {
    data.message = this.value
    update()
})
```

## Step01 MVVM(Mock版)

![MVVM原理](https://tva1.sinaimg.cn/large/007S8ZIlly1ggp6psq3ugg30dc0a0app.gif)

MVVM框架其实就是在原先的View和Model之间增加了一个VM层完成以下工作。完成数据与视图的监听。我们这一步先写一个Mock版本。其实就是先针对固定的视图和数据模型实现监听。

### 接口定义

我们MVVM的框架接口和Vue3一模一样。

初始化需要确定

- 视图模板
- 数据模型
- 模型行为 - 比如我们希望click的时候数据模型的message会会倒序排列。

```js
const App = {
  // 视图模板
  template: `
<input v-model="message"/>
<button @click='click'>{{message}}</button>
`,
  // 数据模型
  data() {
    return {
      message: 'Hello Vue 3!!'
    }
  },
  // 行为函数
  methods: {
    click() {
      this.message = this.message.split('').reverse().join('')
    }
  }
}
const {
  createApp
} = Vue
createApp(App).mount('#app')
```

### 程序骨架

```js
const Vue = {
  createApp(config) {
    // 编译过程
    const compile = (template) => (observed, dom) => {
    }
    // 生成渲染函数
    const render = compile()

    // 定义响应函数
    let effective
    // 数据劫持
    observed = new Proxy(config.data(), {
    })

    return {
      // 初始化
      mount: function (container) {
      }
    }
  }
}
```

### 编译渲染函数

MVVM框架中的渲染函数是会通过视图模板的编译建立的。

```js
// 编译函数
// 输入值为视图模板
const compile = (template) => {
  //渲染函数
  return (observed, dom) => {
  	// 渲染过程
	}
}
```



简单的说就是对视图模板进行解析并生成渲染函数。

大概要处理以下三件事

- 确定哪些值需要根据数据模型渲染

  ```js
  // <button>{{message}}</button>
  // 将数据渲染到视图
  button = document.createElement('button')
  button.innerText = observed.message
  dom.appendChild(button)
  ```

- 绑定模型事件

  ```js
  // <button @click='click'>{{message}}</button>
  // 绑定模型事件
  button.addEventListener('click', () => {
    return config.methods.click.apply(observed)
  })
  ```

- 确定哪些输入项需要双向绑定

```js
// <input v-model="message"/>
// 创建keyup事件监听输入项修改
input.addEventListener('keyup', function () {
  observed.message = this.value
})
```

完整的代码

```js
const compile = (template) => (observed, dom) => {

    // 重新渲染
    let input = dom.querySelector('input')
    if (!input) {
        input = document.createElement('input')
        input.setAttribute('value', observed.message)
      	
        input.addEventListener('keyup', function () {
            observed.message = this.value
        })
        dom.appendChild(input)
    }
    let button = dom.querySelector('button')
    if (!button) {
        console.log('create button')
        button = document.createElement('button')
        button.addEventListener('click', () => {
            return config.methods.click.apply(observed)
        })
        dom.appendChild(button)
    }
    button.innerText = observed.message
}
```



### 数据监听的实现

Vue普遍走的就是数据劫持方式。不同的在于使用DefineProperty还是Proxy。也就是一次一个属性劫持还是一次劫持一个对象。当然后者比前者听着就明显有优势。这也就是Vue3的响应式原理。

Proxy/Reflect是在ES2015规范中加入的，Proxy可以更好的拦截对象行为，Reflect可以更优雅的操纵对象。
优势在于
- 针对整个对象定制 而不是对象的某个属性，所以也就不需要对keys进行遍历。
- 支持数组,这个DefineProperty不具备。这样就省去了重载数组方法这样的Hack过程。
- Proxy 的第二个参数可以有 13 种拦截方法，这比起 Object.defineProperty() 要更加丰富
- Proxy 作为新标准受到浏览器厂商的重点关注和性能优化，相比之下 Object.defineProperty() 是一个已有的老方法
- 可以通过递归方便的进行对象嵌套。

说了这么多我们先来一个小例子
```js
var obj = new Proxy({}, {
    get: function (target, key, receiver) {
        console.log(`getting ${key}!`);
        return Reflect.get(target, key, receiver);
    },
    set: function (target, key, value, receiver) {
        console.log(`setting ${key}!`);
        return Reflect.set(target, key, value, receiver);
    }
})
obj.abc = 132

```

这样写如果你修改obj中的值，就会打印出来。

也就是说如果对象被修改就会得的被响应。

![image-20200713122621925](https://tva1.sinaimg.cn/large/007S8ZIlly1ggp8d10rhfj30ej06ht9v.jpg)

当然我们需要的响应就是重新更新视图也就是重新运行render方法。

首先制造一个抽象的数据响应函数

```js
// 定义响应函数
let effective
observed = new Proxy(config.data(), {
  set(target, key, value, receiver) {
    const ret = Reflect.set(target, key, value, receiver)
    // 触发函数响应
    effective()
    return ret
  },
})
```

在初始化的时候我们设置响应动作为渲染视图

```js
const dom = document.querySelector(container)
// 设置响应动作为渲染视图
effective = () => render(observed, dom)
render(observed, dom)
```



### 视图变化的监听

浏览器视图的变化,主要体现在对输入项变化的监听上，所以只需要通过绑定监听事件就可以了。
```js
    document.querySelector('input').addEventListener('keyup', function () {
        data.message = this.value
    })
```



### 完整的代码

```js
<html lang="en">

<body>
    <div id='app'></div>
    <script>
        const App = {
            // 视图
            template: `
                <input v-model="message"/>
                <button @click='click'>{{message}}</button>
            `,
            data() {
                return {
                    message: 'Hello Vue 3!!'
                }
            },
            methods: {
                click() {
                    this.message = this.message.split('').reverse().join('')
                }
            }
        }

        const Vue = {
            createApp(config) {

                // 编译过程
const compile = (template) => (observed, dom) => {

    // 重新渲染
    let input = dom.querySelector('input')
    if (!input) {
        input = document.createElement('input')
        input.setAttribute('value', observed.message)
        input.addEventListener('keyup', function () {
            observed.message = this.value
        })
        dom.appendChild(input)
    }
    let button = dom.querySelector('button')
    if (!button) {
        console.log('create button')
        button = document.createElement('button')
        button.addEventListener('click', () => {
            return config.methods.click.apply(observed)
        })
        dom.appendChild(button)
    }
    button.innerText = observed.message
}
                // 生成渲染函数
                const render = compile()

                // 定义响应函数
                let effective

                // 数据劫持
                observed = new Proxy(config.data(), {
                    set(target, key, value, receiver) {
                        const ret = Reflect.set(target, key, value, receiver)
                        // 触发函数响应
                        effective()
                        return ret
                    },
                })

                return {
                    mount: function (container) {
                        const dom = document.querySelector(container)
                        effective = () => render(observed, dom)
                        render(observed, dom)
                    }
                }
            }
        }

        const {
            createApp
        } = Vue
        createApp(App).mount('#app')
    </script>
</body>

</html>
```



OK今天写到这，终于完成了第一步虽然大部分还都是固定的至少把大体结构搞定了。



Step02 编译函数(Mock)



喜欢的点赞👍👍👍👍👍  保持关注

我会持续更新的

