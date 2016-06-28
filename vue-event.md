Vue的事件解读

这里有一个容易混淆的地方，在Vue中事件有两方面的内容，一方面是自定义事件，一方面是为DOM绑定事件。

#DOM事件

在Vue中为DOM元素绑定事件的具体方法在文章中的[方法与事件处理器](http://cn.vuejs.org/guide/events.html)章节，通过v-on指令或事件语法糖 `@` 为DOM元素绑定事件。Vue解析组件模板后，在绑定更新 v-on 指令时会为DOM元素[绑定事件](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/on.js#L131)(当然如果元素为 iframe ，会等到 iframe 加载完成后再为其绑定事件)。

Vue中为DOM元素绑定事件是采用DOM2级事件的处理方式，因为Vue服务的是IE9以上的现代浏览器，他们也都是支持DOM2级事件。因此下例中 

```
	<div @click="func"></div>
```


实际上相当于

```
	el.addEventListener('click', func)
```

所以 `addEventListener` 支持绑定的事件，`v-on` 指令也都支持。同样的理论上也可以解绑事件，虽然也有相应的 api ，但是Vue文档中并没有显示地告诉我们怎么做。

在代码中可以看到，每个 `v-on` 指令都有一个 [reset](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/on.js#L140) 方法， reset 方法是当指令所绑定方法发生改变时，重新绑定事件之前的解绑操作，我们可以利用这个 api 来解绑事件。因此如果需要解绑事件，我们可以遍历 vm._directives 找到相应该指令，进行解绑。

当然既然是采用DOM2级事件处理，也可以使用 `removeEventListener` 直接进行解绑，看这个 [demo](https://jsfiddle.net/banama/73k5fe13/)。 执行解绑操作后 btn1 的确解绑成功了，但 btn2 没有解绑成功，这要说到 v-on 指令的[修饰符](http://cn.vuejs.org/api/#v-on)，见源码中对带有修饰符的 handler 的[处理](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/on.js#L104-L123)。顾名思义，修饰符修饰过的 handler 做了更多的事情，Vue的处理是包装原 handler 新的 handler 用于向DOM元素绑定，而解绑时仍然解绑原方法当然会失败。

当然这只是分析Vue的事件绑定原理，大多数情况下我们并不需要去解绑事件。合理的利用事件委托可以解决大部分由事件绑定引起的性能问题。

##自定义事件

Vue自定义事件是为组件间通信设计，自定义事件提供了 $on、$off、$once、$emit、$broadcast、$dispatch 几个 api，非常简洁。

首先提两个vm的私有变量，vm._events 和 vm._eventCount。每个vm实例所有的自定义事件都将存储在 vm._events，而 vm._eventsCount 存储的是执行事件广播后子组件触发自定义事件处理程序的数量，这是为了事件广播优化而来的，如果 vm._eventsCount[event] 数量为零，当事件广播时则可断定子组件没有该事件的监听器，就没必要向子组件层层捕获该事件监听器了。

###$on

注册一个自定义事件，注册事件很简单，首先将其挂载到该实例下

```
vm._events[event] = fn
```

然后是向上传播，更新各个组件的 _eventsCount。这里需要注意，我们可以通过 $on 为生命周期注册钩子，[点击](https://jsfiddle.net/banama/w605txu4/)查看demo，但是生命周期不可冒泡和广播，所以需要更新 eventsCount 前需要过滤。 查看[modifyListenerCount](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/api/events.js#L191-L201)

###$once

因为 $once 注册的事件是一次性的，执行完后卸载，所以其实 $once 调用 $on 来注册事件的函数是[包装](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/api/events.js#L28-L32)过的。

###$off

理解了注册事件的流程(其实就是更改 _events 和 _eventsCount)那么卸载事件也就很清晰了。

但是$off支持三种卸载方式

1、[如果没有参数，则删除所有的事件监听器](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/api/events.js#L48-L59)

遍历 _events，冒泡更新每个事件的 _eventsCount，清空 vm._events

2、[	如果只提供了事件，则删除这个事件所有的监听器](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/api/events.js#L65-L69)

冒泡更新每个事件的 _eventsCount，vm._events 中剔除该事件

3、[如果同时提供了事件与回调，则只删除这个回调](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/api/events.js#L73-L80)

遍历 vm._events[event] 的事件处理方法，如果该事件处理方法和回调相同，则从 vm._events[event] 剔除该事件处理方法，并冒泡更新该事件的 _eventsCount

###$emit

触发事件，直接遍历 vm._events[event] 的每个事件处理程序并执行。

$emit 返回 shouldPropagate，shouldPropagate 是一个布尔值，取决于父链上的是否存在该事件的监听器以及，事件处理程序返回的值。他决定 $dispatch 是否停止冒泡。

###dispatch

派发事件。首先在实例上触发该事件，默认情况下将会停止冒泡传播，但如果 $emit 返回的 shouldPropagate 为 true，则该事件会继续沿父链向上传播，即在父组件继续向上派发事件。

###broadcast

事件广播。深度优先遍历子组件，并执行各个子组件的监听器事件处理程序，在绑定和卸载自定义事件时会会每个组件维护一个 vm._eventsCount，而它的作用正是在深度遍历的时候给予提示，避免不必要的深度遍历。


通过自定义事件在组件之间的传播，我们可以利用它进行组件通信。组件通信在应用开发过程中是一个棘手的问题，因为它直接关系到整个应用的健壮和可维护程度，在开发大型项目中建议引入vuex，从应用架构的角度来考虑组件通信相比这种事件形式更容易维护，比如多个子组件都有派发事件与父组件进行通信，如果子组件派发事件不注意命名规范，出现命名重复情况，那么父组件监听器根本不知道这个事件是从哪里派发过来的以技如何处理，这是隐患之一。如果采用这种方式进行组件通信，那么必将导致子组件大量派发事件，那么父组件将要维护大量的事件监听器，如果时间久了，很容易忘记监听器和派发事件子组件的对应关系，这又增加了开发与维护成本。充斥着事件派发的组件维护成本也是一个容易留坑的地方。此外通过事件可以进行父子组件的通信，但兄弟组件的通信有需要增加不少开发成本。

###组件的自定义事件

在上文分析DOM元素绑定事件中，我们用到这个例子

```
	<div @click="func"></div>
```

但是有时候会出现 v-on 为组件绑定事件的情况，如

```
	<demo @myevent="func"></demo>
```

上文中没有分析到，留在这里说，这里有两个明显区别

* 是组件而不是DOM元素
* 自定义事件而不是DOM事件

因此显然 `addEventLisntener` 不适用，而且Vue执行的也是和第一个例子完全不同的处理方式， 对其的处理在 [registerComponentEvents](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/instance/internal/events.js#L20)。它其实是为组件注册自定义事件。这里 v-on 指令绑定的结果是 demoVm._events[myfunc] = [func] 以及更新 _eventsCount。

查看这个 [demo](https://jsfiddle.net/banama/43tswmdo/) 。

可见 v-on 指令既可为DOM元素绑定事件也可为组件绑定自定义事件。明白了这个，这个[issuse](https://github.com/vuejs/vue/issues/3098) 的原因也就很明了了。
