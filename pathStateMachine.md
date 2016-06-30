Vue 模板表达式解析和 path 状态机

Vue 模板中数据绑定语法是是采用 "Mustache" 语法（双大括号）， Mustache 标签内的文本称为绑定表达式。在 Vue 中，一段绑定表达式由一个简单的 JavaScript 表达式和可选的一个或多个过滤器构成。在解析组件模板的时候会将这段表达式根据组件数据更新到 DOM 中。

## 表达式解析

Vue 的表达式是通过自己的解析来获取数据的，因此并不是所有的 javascript 表达式都支持。下面是一个表达式的解析的过程

```
{{ message.split('').reverse().join('') }}
```

Vue 首先会将这段文本[解析](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/text.js#L62-L86)为 tokens，

```
var tokens = Vue.parsers.text.parseText("{{ message.split('').reverse().join('') }}")
```

结果为

```
tokens = [
	{
		html: false,
		hasOneTime: false,
		tag: true,
		value: "message.split('').reverse().join('')"
	}
]
```

然后将 token [转化](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/text.js#L107-L113)成表达式 

```
var expression = Vue.parsers.text.tokensToExp(tokens)
```

结果为 

```
expression = "message.split('').reverse().join('')"
```

这个 expression 正是创建 watcher 时所用到的表达式，watcher 在为模板表达式与响应数据建立联系的时候会去解析该表达式并获取值。

```
var res = Vue.parsers.expression.parseExpression(expression)
```

Vue 解析表达式其实是为该表达式定义 getter 和 setter 方法，但并不是所有表达式都可以定义 setter 方法，可以设置 setter 方法的表达式必须是一个合法的数据路径，比如 `model.raw`。 Vue 解析这个表达式，如果他是一个合法的对象访问路径，setter 方法才可以被设置成功，否则 [setter 方法为 undefined](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/expression.js#L178-L181)，但也并不是所有合法数据都可以这样做，必须要保证这个路径的属性值是存在的，否则当执行表达式的 setter 方法时将会[报错](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/path.js#L339-L341)。


表达式的 getter 方法结合组件数据获取表达式的值，通过 `Function` 构造器为表达式的 javscript 字符串创建一个函数，从而访问到 Vue 表达式的真实值。

```
var getter = function(expression){
	return new Function('scope', return 'scope.' + expression + ";")
}
```
获取表达式的值时，执行 getter 方法从作用域对象内取值

```
var model = {
	message: "data from res getter",
	
}
getter.call(model, model) // retteg ser morf atad
```

Vue中的双向绑定原理，正是设置表达式的 setter 方法，在视图中改变组件数据，驱动数据更新。上文中说到并不是所有的表达式都可以设置 setter 方法，必须是一个合法的路径，Vue 通过状态机模式实现了对字符串路径的解析。

## path 状态机

path 指的是一个对象的属性访问路径，比如 `b.c.d` 来访问对象

```
a = {
	b: {
		c: {
			d: "e"
		}
	}
}
```

的属性，很明显该路径对应的值是 `"e"`。Vue 将表达式的访问路径字符串解析成更易于 js 使用的状态。 `b.c.d` 将会被解析成 `['b', 'c', 'd']`，这样如果将该路径的属性值设为 f，执行 `a[pathAry[0]][pathAry[1]][pathAry[2]] = 'f'` 即可为改字符串访问路径下的属性赋值。

Vue 实现了一个 [parse](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/path.js#L164) 方法解析 path。一个合法 path 是有规律的，比如 `a][` 这样一个 path 明显是不合法，因为当路径第一个字符为 a 时，对于对象访问路径来说，第二个字符可能存在的情况 

1. 字符，则仍为属性的名字，拼接到前一个字符串之后作为新的属性名
2. `.` ， 则 a 为第一级属性访问key，接着遍历第三个字符，访问下一级属性
3. `[` ，同上一种情况
4. `undefined` ，没有字符串，解析完毕。

因此当第二个字符串为 `]` 不符合当前状态所期望的输入，因此解析失败。

Vue 的状态机模式解析 path 实际上是将 path 的每个索引的字符视为一个状态，将接下来一个字符视为当前状态的输入，并根据输入进行状态转移以及响应操作，如果输入不是期望的，那么状态机将异常中止。只有状态机正常运行直到转移到结束状态，才算解析成功。

Vue 的 pathStateMachine 有八种状态，例如 [BEFORE_PATH](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/path.js#L33-L38)

BEFORE_PATH 是 pathStateMachine 的初始状态，它的状态模型为

```
pathStateMachine[BEFORE_PATH] = {
  'ws': [BEFORE_PATH],
  'ident': [IN_IDENT, APPEND],
  '[': [IN_SUB_PATH],
  'eof': [AFTER_PATH]
}
```

从状态模型中知道 BEFORE_PATH 接受四种输入

- `ws`，状态转移到 BEFORE_PATH
- `indent`，状态转移到 IN_IDENT，并执行 APPEND 操作
- `[`，状态转移到 IN_SUB_PATH
- `eof`，AFTER_PATH

输入 ws、indent、eof 具体代表什么，可以在 [getPathCharType](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/path.js#L90-L135) 看到定义。其他7种状态模型可在 [vue/src/parsers/path.js](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/path.js#L33-L81) 看到。


状态机运行过程中，Vue 在通过 action 处理每一级 path 的路径值。比如当处于状态 IN_IDENT 时，再次输入字符，会执行 APPEND 操作，将该字符串与之前的字符/字符串拼接。再次输入 `.` 或 `[` 会执行 PUSH 操作，将之间的字符串视为访问对象的一个属性。Vue 的 pathStateMachine 有四种 action，他们主要是根据 path 特征和状态提取出对象的访问属性，并按照层级关系依次推入数组。详细[见代码](https://github.com/vuejs/vue/blob/e9872271fa9b2a8bec1c42e65a2bb5c4df808eb2/src/parsers/path.js#L173-L207)。


下面是是一个详细例子分析状态机的状态转移过程，需要分析的 path 为 `md[0].da["ky"]`

```
先声明
keys 		= 存放对象访问属性的数组
key			= 临时变量
index		= 索引
mode		= 当前状态
input		= 输入
transfer	= 状态转移
action		= 操作
```
现在进入状态极

index = 0

```
mode 		= BEFORE_PATH
input		= 'm'
transfer 	=> IN_IDENT
action		=> APPEND
keys 		= []
key 		= 'm'
```

index = 1

```
mode 		= IN_IDENT
input		= 'd'
transfer 	=> IN_IDENT
action		=> APPEND
keys 		= []
key 		= 'md'
```

index = 2

```
mode 		= IN_IDENT
input		= '['
transfer 	=> IN_SUB_PATH
action		=> PUSH
keys 		= ['md']
key 		= undefined
```

index = 3

```
mode 		= IN_SUB_PATH
input		= '0'
transfer 	=> IN_SUB_PATH
action		=> APPEND
keys 		= ['md']
key 		= '0'
```

index = 4

```
mode 		= IN_SUB_PATH
input		= ']'
transfer 	=> IN_PATH
action		=> INC_SUB_PATH_DEPTH
keys 		= ['md', '0']
key 		= undefined
```


index = 5

```
mode 		= IN_PATH
input		= '.'
transfer 	=> BEFORE_IDENT
action		=> None
keys 		= ['md', '0']
key 		= undefined
```

index = 6

```
mode 		= BEFORE_IDENT
input		= 'd'
transfer 	=> IN_IDENT
action		=> APPEND
keys 		= ['md', '0']
key 		= 'd'
```

index = 7

```
mode 		= IN_IDENT
input		= 'a'
transfer 	=> IN_IDENT
action		=> APPEND
keys 		= ['md', '0']
key 		= 'da'
```

index = 8

```
mode 		= IN_IDENT
input		= '['
transfer 	=> IN_SUB_PATH
action		=> PUSH
keys 		= ['md', '0', 'da']
key 		= undefined
```

index = 9

```
mode 		= IN_SUB_PATH
input		= '"'
transfer 	=> IN_DOUBLE_QUOTE
action		=> APPEND
keys 		= ['md', '0', 'da']
key 		= '"'
```

index = 10

```
mode 		= IN_DOUBLE_QUOTE
input		= 'k'
transfer 	=> IN_DOUBLE_QUOTE
action		=> APPEND
keys 		= ['md', '0', 'da']
key 		= '"k'
```


index = 11

```
mode 		= IN_DOUBLE_QUOTE
input		= 'y'
transfer 	=> IN_DOUBLE_QUOTE
action		=> APPEND
keys 		= ['md', '0', 'da']
key 		= '"ky'
```

index = 12

```
mode 		= IN_DOUBLE_QUOTE
input		= '"'
transfer 	=> IN_SUB_PATH
action		=> APPEND
keys 		= ['md', '0', 'da']
key 		= '"ky"'
```

index = 13

```
mode 		= IN_SUB_PATH
input		= ']'
transfer 	=> IN_PATH
action		=> PUSH_SUB_PATH
keys 		= ['md', '0', 'da', 'ky']
key 		= undefined
```

index = 14

```
mode 		= IN_SUB_PATH
input		= 'eof'
transfer 	=> AFTER_PATH
action		=> None
keys 		= ['md', '0', 'da', 'ky']
key 		= undefined
```

至此状态机结束。最后，更清晰的了解状态转移可以看  @勾三股四 的[图](http://img2.tbcdn.cn/L1/461/1/3acfc1236df2d6cd068dd8540e0b0baeb4b8916b)。

