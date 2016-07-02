为什么 Vuex 比 Redux 更适合 Vue.js

这边文章首先分析 Vuex 的原理，从 Vuex 的实现分析 redux 配合 Vue 的问题。对 redux 分析的比较浅显，抛砖引玉，欢迎讨论。

## Vuex 的原理

Vuex 的实现分成两部分，一部分是创建 store，另一部分是在组件中挂载 store，并解析 vuex 属性。

在开始一个 vuex 应用之前，首先要根据 state 和 mutations 为 Vue 生成一个状态管理器。这其中只有一件比较重要的事情，就是通过为 state 构建一个 Vue 实例将 state 变成响应式数据。

在一个 vuex 应用中，每个组件的形式是这样

```
new Vue({
    template: '',
    store: store,
    vuex: {
        getters: {},
        actions: {}
    }
})
```

Vuex 在通过插件形式安装的时候会将 vuexInit 函数加入 vm 生命周期的 init 或 beforeCreate 阶段，因此在组件的对应生命周期会执行 vuexInit 函数来挂载store。对于 store ，Vuex 的处理方式是将它直接挂载到 $options.$store 以供使用，组件的 store 属性其实也并不是必需的，当组件的 store 属性不存在时，Vuex 会去 `vm.$options.parent` 寻找 `$store`，但无论是组件本身有 sore 属性还是直接从父组件获取，vm.$store(不是 vm.store ) 都是必需的。否则 Vuex 会抛出[警告](https://github.com/vuejs/vuex/blob/5385d415edeabb1e991d35678d36a90328ea0553/src/override.js#L37-L40)。

接下来是对 vuex 属性，Vuex 会将 vuex 属性解构成 getters 和 actions。并将 getters 的每个属性都挂载 vm 下(有可能被组件的 $options.data() 的属性覆盖)，同时定义每个值的 getter 方法，但并不会定义 setter 方法，这是因为根据 Vuex 的设计是不允许开发者直接在组件内更改 store.state，而对数据的改动要通过 vuex.actions 内的方法。在这里 getter 方法其实是 `makeComputedGetter`，如果你读过 [Vue数据绑定和响应式原理](https://github.com/banama/aboutVue/blob/master/vue-observe.md) 的计算属性章节，可能会很熟悉这个函数。没错，Vuex 实际上将 vm.vuex.getter 内的属性当作当前 vm 的计算属性来处理。和计算属性的区别是计算属性依赖计算的是 vm.$options.data 内的值，而 vm.vuex.getter 的属性依赖计算的是 store.\_vm.$options.data。这样所有组件的渲染都将都可以直接从状态树拿数据来渲染 UI 。

所有组件也都可以和状态树交互，但不允许直接更改状态树数据，而是要通过 vm.vuex.actions 内的方法。Vuex 会将这些方法绑定到组件的 `$options.methods` ，因此他们和组件的方法一样，可以在模板解析中通过 v-on 指定绑定到 DOM 元素上。而 actions 内的方法将通过 dispatch 触发 mutations 来更新全局状态。

Vuex 是一套类 flux 的架构，以上的用法有时候可能会让人感觉十分冗余，比如为什么组件不允许直接修改 store 实例的状态，而是通过actions、mutations ? 其实也不是那么绝对，我们总有办法能做到，甚至一点不复杂， `vm.$store.state.key = value` 这样就随手更新了全局状态树，只有在开启严格模式后( Strict Mode )，Vuex 才会检测 state 是否是在 mutations 中修改。Vuex 屏蔽了 vm.$store.state.setter 方法，但是仍旧提供了一个 api [replaceState](https://github.com/vuejs/vuex/blob/5385d415edeabb1e991d35678d36a90328ea0553/src/index.js#L87)。其实就像 MVC 一样，架构给我们提供了最佳实践的建议，通过“约定”来提升大型项目的健壮和可维护性。比如通过 action 和 mutations 更新 store，从组件中解耦数据请求和逻辑，虽然“绕弯子”但可以让大型项目更容易理解和维护。我们仍然可以灵活的按照自己的想法进行开发，但是必须明白自己所做事情代价与回报。

再比如 Vuex 要求 mutations 必须事同步的。先看看下面的例子

 [muattions 也可以是异步的](https://jsfiddle.net/banama/q9fgmu5v/)

 会发现如果触发一个异步的操作时，大部分情况异步的 action 和 异步的 mutations 都是没问题的，因为 state 本来就是相当于一个计算属性，只要在异步的回调里修改了 state，都会给订阅器发消息触发视图的更新。但是如果有一个场景，我们需要处理 mutation 和 state 的对象关系，比如 logger 中间件，如果 mutations 中存在异步操作，那么当 mutation 被触发时 logger 中间件已经开始打印，但状态树的状态还未更新，这就失去了日志打印状态的变化。同时 mutations 与 状态树正确的对应关系也正是实现`时间旅行`的关键。因此异步的 mutation 其实是有副作用的，只是有时候并不会对我们造成影响。

虽然我们可以灵活的使用 Vuex ，但个人建议最好遵循官网文档的推荐。

如果理解了 Vuex 的原理，那么来看看 Vue + redux 。redux 是一个泛用的 flux 实现，在 redux 的哲学中推崇 state 是不可修改的，因此在 redux 的 reducer 更新数据一般是这样的

```
state = Object.assign({}, state)
state.count++
return state
```

redux 推荐使用 Object.assign() 新建了一个副本，但是 Vue 定义每一个响应式数据的 __ob__ 都是不可枚举的，因此使用 Object.assign 为 state 创建的副本将会丢失监听器，同时如果使用 redux 作为 Vue 的状态管理器，当 redux 的状态树的更新时，我们需要依赖 redux 实现的监听器来检测 store 的变化，将其重新挂载到 vm，这将导致 Vue 的响应系统重新为 store 创建监听器，每次数据更新 model 的性能开销活生生抵消了 Vue 的优化。

其实 redux 配合 Vue 使用还有另一个取巧的方式，那就是在 reducer 更新 state 的时候直接修改 state，因为 state 已经是响应式数据，因此这样直接更改 state 可以直接触发视图的更新而并不用依赖 redux 本身对 state 的监听。但反过来，这种严重违背 redux 设计哲学的用法可能会引发各种副作用，是为了用 redux 而用 redux。

当然这里只是对 redux 浅显的分析，相信进行深入优化 redux 能够配合 Vue 不损失性能并优雅的使用，但为什么不使用 Vuex 呢？它比 redux 更简单，而且为 Vue 而生。
