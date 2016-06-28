Vue列表渲染性能优化原理

Vue是一个高效的mvvm框架，这得益于作者已经帮我们框架内部做了足够的优化，比如各个细节的缓存(parseText结果的缓存，compile编译结果的缓存等)。

大列表是容易造成性能问题的地方，一不小心就会造成大量的重绘和重排。Vue的列表渲染实现在for指令的update方法， 性能优化的大部分细节在[diff](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L106)函数。

列表渲染时会为迭代列表的每一份数据并未他们生成各自对应的片段对象frag，frag.node为片段对象的DOM元素。 

##frag缓存和track-by

在列表渲染过程中，当列表数据发生变化时，为了避免frag的重复创建和大规模的重新渲染，Vue会尽可能复用缓存的frag，高效的缓存frag命中率也是DOM元素复用的关键。

以下例子

```
new Vue({
	template: `
		<ul>
			<li v-for='i in model'>
		</ul>
	`,
	data: function(){
		return {
			model: [1, 2, 3]
		}
	}
	
})
```
当这个组件中的列表首次渲染时，Vue 会将[创建](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L264)的 frag 缓存到 [dir.cache](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L80) 。默认通过数据对象的特征来决定对已有作用域和 DOM 元素的复用程度。例如当数据对象为Array时，缓存 id 为数组的 value ，当数据对象为 Object 时，缓存 id 为对象的 $key 。对于这个例子来说三个缓存 id 为1、2、3。

这样在上面的例子中，如果 vm.model 变为 [3, 2, 1]，新的列表的三个片段的缓存 id 分别为3、2、1，因此我们能做到复用全部已创建的frag。

v-for 指令中片段 frag 的缓存 id 计算规则在 [getTrackByKey](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L648) ，从中可以看到，当 track-by 不存在时，缓存 id 将取数组的 value 或对象的 key 。但是这里有一个问题，如果数组出现重复值，会出现缓存 id 冲突的[警告](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L417)。副作用就是会忽略重复的片段，这是因为相同的缓存 id 获取的 frag 将会引用同一个 DOM 元素。当该 DOM 元素在复用算法中处理过一次后会将 frag 的 reused 属性[变为 false ](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L220)，这就导致 v-for 指令会重新尝试将该[插入到DOM](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L215)中，然而因为文档中已经存在该 DOM 元素，就会导致插入失败。

这时 Vue 提示我们可以使用 `track-by='$index'` ，它将使用数据的索引作为缓存 id ，索引作为缓存id一定是唯一的，但同时新旧数据的相同索引的缓存 id 是相同的，所使用的 frag 也是同一份。这回导致新列表的的 frags 失去和数据顺序的映射关系。而 frag.node 的数据在[flushBatcherQueue](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/batcher.js#L37)更新。因为这种更新列表的方式不会移动DOM元素，而是在原位更新，简单地以对应索引的新值刷新，并不会受重复值的影响，这让数据替换非常高效，但同时也会有副作用，比如如果片段存有私有数据或状态，原位更新模式并不会同步状态。如下例

[[track-by='index'副作用]](https://jsfiddle.net/banama/kmgwqtc9/)

在第一个例子中，没有开启 `track-by='$index'` , v-for 指令会根据数组的值作为缓存 id 缓存每项 frag，当反转列表的顺序是，Vue 会根据缓存 id 为列表的每一项取出可复用的frag，但是 frags 的顺序也是反转的，Vue 会通过DOM元素移动算法将每个片段移动到正确的位置，因此当 input 有输入时，因为整个 DOM 节点发生了移动，input 的输入内容并没有错乱，这意味着我们没有丢失 frag 内 DOM 元素的私有状态。

再看第二个例子，开启了 `track-by='$index'` 之后，v-for 指令会根据数据的索引作为缓存 id 缓存每项片段，当反转列表的顺序时，每项的frag 会复用以索引为缓存id所缓存的 frag，所以生成的 frags 和原列表的 frags 是一样的，根据DOM元素移动算法，列表的 DOM 节点并没有移动，每个片段的数据更新会在接下来的流程中更新，所以会发现每个片段的数据更新了，但是因为 DOM 元素节点没有移动，因此每个 DOM 节点中input的输入状态并没有根据元素的变化而更新。

`track-by='$index'` 是一种简洁高效的优化手段，但是使用的时候你必须明白他做了什么。

在一些富交互的列表使用 `track-by=‘$index’` 需要格外谨慎，但是在非 `track-by=‘$index’` 模式我们仍可通过 track-by 尽量优化。有时候 v-for 指令并不能最大化优化，比如

```
vm.model = {
	a: { id: 1, val: "model1"},
	b: { id: 2, val: "model2"},
	c: { id: 3, val: "model2"},
}

列表更新

vm.model = {
	d: { id: 1, val: "model1"},
	e: { id: 2, val: "model2"},
	f: { id: 3, val: "model2"}
}
```

默认情况 v-for 指令对于对象数据会将对象的键作为缓存 id，上面的例子发现列表更新后，对象的键没有重复，所以导致缓存一个都没有命中，而列表更新的结果也是重新渲染整个列表。但上面例子很明显可以看出，如果能够将对象的 a.id、b.id、c.id 作为缓存 id ，那么当列表更新时，所有缓存都能够命中，甚至连DOM元的移动都不需要， `track-by='id'` 就是做这样的事情.


##DOM元素移动和启发算法

diff 算法中另一个重要的优化是 DOM 节点的移动。DOM 节点移动的性能开销非常大，因此减少 DOM 节点移动次数是算法的核心。当然开启 `track-by='$index'`不需要移动DOM元素，只需插入缺少的节点即可。


假如一个列表的 DOM 节点可以全部复用，那么列表的更新的核心就是 DOM 节点移动到合适的位置。简化之后就是下面的场景

old [1, 2, 3, 4, 5, 6, 7]

更新为

new [7, 6, 5, 4, 3, 2, 1]

先声明一下

```
先声明
index		= 索引
target  	= new[index]
targetPrv	= new[index - 1]
current 	= map -> target (target在old里的映射)
currentPrv	= current.prv
move(a, index) 在old中将a元素移动到索引为index的位置
```

让我们来想想怎么移动

```
loop new.length
	move(current, index)
```
这个算法没有问题，可以顺利完成任务，但是这个算法会移动 new.length 次，几乎是最糟糕的情况，有时候节点在正确的地方并不需要移动。

例如

```
old [1, 2, 3, 4, 5, 6, 7]
new [7, 6, 5, 4, 3 ,2 ,1]
完全反转，需要移动7次

old [1, 2, 3, 4, 5, 6, 7]
new [1, 2, 3, 4, 5, 6, 7]
很明显移动次数应该为0，但是还是会移动7次
```

如果在移动之前判断一下，他是不是在正确的位置。而这里对于是否处在正确的判断，是看他们的前一个元素是否相同。

```
loop new.length
	if targetPrv !== currentPrv
		move(current, index)
```

这样做可以在大部分情况下避免不必要的移动，比如对于

```
old [1, 2, 3, 4, 5, 6, 7]
new [7, 1, 2, 3, 4, 5, 6]
```

index => 0

	target 		= 7
	targetPrv 	= undefined
	current 	= 7
	currentPrv	= 6
	
	undefined !== 6
		move(7, 0)
		
	old [7, 1, 2, 3, 4, 5, 6]

我们发现只需移动一次就得到想要结果。

再看一个类似的例子

```
old [1, 2, 3, 4, 5, 6, 7]
new [2, 3, 4, 5, 6, 7, 1]
```

这个例子和上面的例子差不多，很明显只需要移动一次就可以得到想要的结果，然而结果却出乎意料，看一看是怎样移动的


index => 0

	target 		= 2
	targetPrv 	= 1
	current 	= 1
	currentPrv	= undefined
	
	1 !== undefined
		move(1, 0)
		
	old [2, 1, 3, 4, 5, 6, 7]
	
index => 1

	target 		= 3
	targetPrv 	= 2
	current 	= 3
	currentPrv	= 1
	
	2 !== 1
		move(3］ 1)
		
	old [2, 3, 1, 4, 5, 6, 7]

......

old [2, 3, 4, 5, 6, 1, 7]	index => 6

	target 		= 7
	targetPrv 	= 6
	current 	= 7
	currentPrv	= 1
	
	6 !== 1
		move(7, 6)
		
	old [2, 3, 4, 5, 6, 7, 1]
	
直到最后我们的发现，直到移动了7次才得到想要的结果，第一个元素每一次移动都向后冒泡，成功的混淆了每一次判断结果。这个问题在[isseuse#1807](https://github.com/vuejs/vue/issues/1807)有详细说明。

如果我们能忽略第一次移动，那么之后的每一次判断都会成功

old [1, 2, 3, 4, 5, 6, 7]

new [2, 3, 4, 5, 6, 7, 1]

index = 1

	target 		= 3
	targetPrv 	= 2
	current 	= 3
	currentPrv	= 2
	
	2 !== 2
		move(3, 1)
		
	old [2, 3, 4, 5, 6, 7, 1]
	
......

old [2, 3, 4, 5, 6, 7, 1]	index = 6

	target 		= 1
	targetPrv 	= 7
	current 	= 1
	currentPrv	= undefined
	
	7 !== undefined
		move(1, 6)
		
	old [2, 3, 4, 5, 6, 7, 1]
	
这样的结果才是我们想要的。

现在 Vue 判断移动的条件是

```
if	targetPrv !== currentPrv &&
	(!currentPrv || currentPrv.prv !== targetPrv)
	move(current, index)
```

DOM 移动的[核心代码](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/directives/public/for.js#L194-L221)。Vue 声称实现了一些启发算法，以最大化复用 DOM 元素。这里的启发指的是执行 DOM 元素移动的条件，通过判断元素是否在正确的相对位置上，来于评估当前移动是否必要以及是否会造成愚蠢移动，没错，说的正是[1, 2, 3, 4, 5, 6,7] => [2, 3, 4, 5, 6, 7, 1]。

##列表渲染优化实践

Vue 已经为我们做了大量无脑优化，主要在提高 DOM 元素的复用率和减少DOM 元素移动次数算法两方面，具体原理在上文两节中已经分析。DOM 元素的移动算法优化开发者不能做什么，但在提高DOM元素的复用率仍给开发者留有优化余地。

但是优化方式也十分简单，即给v-for列表加一个 `track-by` 属性，提示Vue 如何判断对象时同一份数据，提高缓存命中率。甚至可以直接加 `track-by="$index"` 原位复用DOM元素，这种优化效率最高，但是副作用上文中也说了，会丢失原DOM元素的临时状态组件私有状态，因此在交互复杂的列表中可能会有意想不到的问题，这时使用`非$index的track-by`也可以做到尽可能的性能优化。

	


