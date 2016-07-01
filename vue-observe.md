Vue数据绑定和响应式原理


当实例化一个Vue构造函数，会执行 Vue 的 init 方法，在 init 方法中主要执行三部分内容，一是初始化环境变量，而是处理 Vue 组件数据，三是解析挂载组件。以上三部分内容构成了 Vue 的整个执行过程。


Vue 实现了一个 `观察者-消费者(订阅者)` 模式来实现数据驱动视图。通过设定对象属性的 setter/getter 方法来监听数据的变化，而每个属性的 setter 方法就是一个观察者， 当属性变化将会向订阅者发送消息，从而驱动视图更新。


Vue 的订阅者 watcher 实现在 [/src/watchr.js](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/watcher.js#L36) 。构建一个 watcher 最重要的是 expOrFn 和 cb 两个参数，cb 是订阅者收到消息后需要执行的回调，一般来说这个回调都是视图指令的更新方法，从而达到视图的更新，但是这也不是必须的，订阅回调也可以是一个和任何无关的纯函数。一个订阅者最重要的是要知道自己订阅了什么，watcher 分析 expOrFn 的 getter 方法，从而间接获得订阅的对象属性。


##数据订阅


Vue 的数据订阅主要在上述的第二个阶段。在生命周期 init 和 created 之间执行，这部分实现了对[ options.data 的处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L48)，[ $options.props 的处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L45)，[ $options.computed 的处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L49)，[ $options.methods 的处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L45)，[ $options.events 的处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/events.js#L18)，[ $options.watch 的处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/events.js#L18)等。


这里主要讲对 options.data 的处理 [_initData](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L79) 方法。在 _initData 这个方法中 $options.data 并不是原来的 $options.data , 而是在一个`mergedInstanceDataFn`，这是因为在合并父组件和子组件 options 的 mergeOptions 方法中，对 options 做了特殊处理，对于 $options.data 而言新的 $options.data 是一个封装过的合并父子 $options.data 的新函数 mergeInstanceDataFn。我们遵循的开发实践是倾向于将一个复用组件的 data 属性设为返回原生对象的函数而不是纯对象，因为如果 data 为纯对象，一个组件的多个实例的 data 属性将是同一个对象的引用，者可能会导致意想不到的 bug ，这个在[文档](http://cn.vuejs.org/guide/components.html#组件选项问题)里也有说明。


从 observe 函数正式进入了对数据对象的观察，Vue 中响应式数据都有一个 `__ob__` 属性属性作为标记，这个属性其实就是该对象的观察器。如果数据已经是响应式的，将会跳过对该对象的重新观察，直接返回观察器。 在接下来的处理流程中会根据对象和数组分别处理，因为对象可以定义属性的 setter 方法，对于数组遍历每项的对象递归递归执行 observe 。观察数据的最终处理是 [defineReactive](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/observer/index.js#L194) 方法。 ES5 定义属性的setter和getter方法本身很简单，需要理解的地方是 Vue 定义处理订阅者和观察的关系。


defineReactive 方法中看到每一个数据的 setter 和 getter 函数都是闭包，因此对于每一个数据都存在一个私有变量 dep 用于存放订阅器。简单来说 Vue 在执行属性 getter 方法时收集依赖(收集订阅者)，执行属性的 setter 方法时给订阅者发消息。那依赖收集具体是怎么做的？从[这段代码](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/observer/index.js#L212-L223)中看到并不是每次执行属性的 getter 方法都会触发依赖收集，而是只有当 Dep.target 存在时才会触发依赖收集。 Dep.target 是一个“全局变量”，从后文中可以知道 Dep.target 变量存的是一个订阅者对象。这样就可以理解了，观察器收集订阅对象必然要知道是否有依赖可收集，而不是盲目收集。


Vue 解析组件模板的时候将解析出来的指令绑定到 vm 上，这里还涉及组件解析、指令绑定、表达式解析等部分内容，先忽略这部部分内容细节。Vue 中视图的更新其实就是指令的更新，为了做到数据驱动视图更新，需要注册一个订阅者订阅数据的变化，通过回调来进行指令更新。

```
	<div id="#app">
		<span v-text="model"></span>
	</div>

	new Vue({
		el: "#app",
		data: {
			model: "the model"
		}
	})
```

上例。Vue 解析模板当解析出 v-text 指令时，会为该 DOM 元素注册指令，并将其绑定到 vm 上。绑定指令时会根据指令的信息为 v-text 指令注册一个订阅器

```
new Watcher(vm, 'model', dir.update)
```
订阅器在创建的时候会根据指令的 expression 分析出该表达式的 getter 方法，并执行 getter 方法。getter 方法在真正处理取值之前 watcher 会将 Dep.target 设为他自己。这就告诉 Vue 现在在整个系统中，他才是主角。那么可以预见在接下来 getter 取值过程中，如果该表达式的数据涉及到获取 vm 的响应式数据将会触发该响应数据的依赖收集，而且订阅者一定是自己。这时该 watcher 会把自己加入该响应数据的依赖，并将响应数据的依赖对象存到自己的 deps ( deps 里存的其实是各个响应数据依赖对象的引用因此可以手动改动响应数据的订阅依赖，但是最好不要这么做，这在计算属性中大有用处)。


这样就建立起了观察者和消费者的关系，当 vm.model 发生改变时，model 的 getter 方法将会向其所有的订阅者发送消息，触发指令的更新，从而做到视图的更新。


订阅器的回调并不一定是更新指令，只有在解析模板过程中注册的订阅者回调才是更新指令。Vue 组件的 $options.watch 也可以创建订阅者，如下例

```
	new Vue({
		data: function(){
			return {model: 'the model'}
		},
		watch: {
			'model': function(){
				console.log('hello' + this.model)
			}
		}
	})
```

[_initEvents](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/events.js#L23) 在对 $options.watch 进行处理的时候，也是构造了一个订阅者，只是和在解析指令过程中构造的订阅者不同，这里构建的订阅者不用通过解析指令来获得回调，而是直接一个纯函数。



## 计算属性原理

如果清楚了Vue的响应式原理，那么理解计算属性也会变的很容易。

`$options.computed` 的处理在 [_initComputed](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L208) ，在这个方法中 Vue 将计算属性的 key 挂载到 vm 下(并没有任何重复 key 的检测以及警告，又因为 _initComputed 是 _initState的最后一个方法 ，所以会直接覆盖，因此这里要自己避免 vm 下挂载的属性的重复，见最后章节)，并定义了其 getter 和 setter 方法，	setter 方法没有复杂的地方，详细可看[代码](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L225)， getter 是通过 [makeComputedGetter](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L234) 生成的一个函数，当生成 getter 的时候，这里为该计算属性构建了一个 watcher ，这个 watcher 和之前见到的都不一样，因为它构建的时候[回调为 null ](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L235,c12)，这意味着即使其他响应式数据将该订阅者收集为依赖，那么当数据响应时，该订阅者也不会有任何作为。的确是这样，因为这个订阅器其实是一个"中间订阅器"，它的存在并不是为了触发订阅者做什么，而是为了帮助监听该计算属性变化的订阅器订阅消息。

看下面的例子

```
new Vue({
	template: 'computed',
	data: {
		raw: 1
	},
	computed: {
		model: function(){
			return this.raw + 1
		}
	},
	watch: {
		'model': function(){
			console.log('the computed')
		}
	}
})
```

在计算属性处理完成后，会发现在 vm 下挂载了一个 key 为 model 的属性，该属性的 getter 为 `makeComputedGetter` ，如果没有接下来的watch，那么该计算属性用法和

```
vm.model = function(){
	return this.raw + 1
}
```

差不多，虽然在 vm.model 的闭包 getter 里已经构建了一个watcher，并且一旦调用 getter 方法，该 watcher 就会被收集到 vm.raw 的私有变量 dep 里。但是这样对于响应式是没有任何意义的，因为并没有谁会因为 vm.raw 的变化而做什么。直到有订阅者开始订阅 vm.model 变化的消息，计算属性才变的有意义。而计算属性的核心 `makeComputedGetter` 正是来处理这个事情。

当为 vm.model 创建一个订阅者的时候，watcher 会执行 vm.model 的 getter 方法，同样的在执行 getter 之前，会先将 Dep.target 设为该 watcher，makeComputedGetter 做的事情其实是分析计算属性的依赖，有两个步骤。

一是[evalute](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L240)。该方法做的是

* 缓存当前 Dep.target
* 为执行 makeComputedGetter 闭包内的 watcher (正是之前所说没什么实际意义的订阅者，)的 get 方法。因为层层关系，最终会执行到 vm.raw 的 getter 方法，因为 Dep.target 不为空，所以这时会进行依赖收集，将该 Dep.target 收集为 vm.raw 为订阅者，同时将该响应数据的 dep 加入该 watcher 的 deps 中
* 还原Dep.target


二是 [watcher.depend](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L243) ，depend 方法将目前的订阅者的订阅对象"分享"给 Dep.target


经过以上两步，通过"中间订阅者"这个 watcher 就实现了真正意义的计算属性，计算属性也变为响应式属性。


## 追踪(订阅)


看这个 [demo](https://jsfiddle.net/xcd8b8yd/1)
<iframe width="100%" height="500" src="https://jsfiddle.net/xcd8b8yd/1/embedded/result,html,js" allowfullscreen="allowfullscreen" frameborder=0></iframe>


上文中讲到 Vue 是先定义响应式数据 ，然后再解析指令的过程中收集依赖。但在上面例子中的 demo1 组件在声明周期 ready 的时候，依赖收集的过程已经执行完毕，这时候为 vm 挂载一个属性，这个属性一定不是响应式的。但是 Vue 提供了动态添加响应的 api ，上面例子中 demo2 组件就是这样一个例子，但是`$set`到底做了什么。

* [解析表达式](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/api/data.js#L48)
* [解析表达式的path](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/expression.js#L173)
* [设置path的值](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/expression.js#L176)
* [为vm添加响应式属性](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/util/lang.js#L64)
* [依赖收集](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/util/lang.js#L65)


经过以上几个步骤就可以动态添加响应式属性。


不同于对象，数组还存在一些 api 可以改变数组本身，而不是通过 setter 方法来改变，这种情况采用常规的观察器是不能观察到的。Vue 对于这种数据变动，[采用Monkey patching扩展Array原型链上的方法](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/observer/array.js#L19)手动给订阅者发消息。例如 pop 方法

```
var pop = Array.prototype.pop
Array.prototype.pop = function(){
	var i = arguments.length
	var args = new Array(i)
	while(i--){
		args[i] = arguments[i]
	}
	var result = pop.apply(this, args)
	var ob = this.__ob__ // 所有被监测的数据都会添加__ob__属性，https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/observer/index.js#L43
	ob.dep.notify()
	return result
}
```

可知被检测的数组发生变化也会触发更新。但在 js 中数组的方法分为两种，一种是变异方法，即方法调用会使数组本身发生变化，例如 pop、push 等，这些方法会直接给订阅者发消息。另一种是非变异方法，即方法调用会返回一个新的数组，原数组本身并不会发生变化，这时如果想要给订阅者发消息只需要将该数组赋给原数组就可以。这里并不用担心 Vue 会重新渲染整个列表，因为 Vue 为 v-for 指令做了巧妙的优化，即通过缓存、track-by以及所谓的[启发算法](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L648)优化过的 DOM 元素移动算法来实现 DOM 元素的最大化复用，从最大程度上避免重新渲染所带来性能消耗。

##其他

在 _initData 这个私有方法中，Vue 将 $options.data 所返回的数据全部代理到了 options._data 上。$options.props 和 $options.data 被观察后都会将该属性挂载到 vm 根节点上

$options.props 的处理流程

* [_initProps](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L45)
* [compileAndLinkProps](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L71)
* [compileProps](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/compiler/compile.js#L184)
* [initProp](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/compiler/compile-props.js#L261)
* `defineReactive(vm, prop.path, value)` 最后经props挂载在 vm 上

$options.data 的处理流程

* [_initData](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L48)
* [vm._proxy](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L159)
* 通过 `Object.defineProperty` 将 data 挂载到 vm 上


因此有可能会出现 props 和 data key 重复了的情况，这时会有[警告](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/state.js#L103-L107)，因为 _initProps 在 _initData 之前执行，所以也并不会覆盖，然而还有其他的情况可能并不会这么友好，比如 $options.methods 和 $options.computed 有相同的 key。
