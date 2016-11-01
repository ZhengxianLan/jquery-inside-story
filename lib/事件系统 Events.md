## 事件系统

jQuery 事件系统按照 3 级事件模型对浏览器事件模型进行了封装和扩展，具有以下特点：

- 通过自定义 jQuery 事件对象，对原生事件对象的差异进行封装和修正，统一了事件对象的属性和方法。
- 提供了灵活、统一的事件绑定和移除机制。
- 支持手动触发事件。
- 支持事件代理和命名空间。
- 提供了常用事件的简写方法。
- 扩展了组合事件 `toggle`、`hover`。
- 基于数据缓存模块实现，避免了 IE 下的内存泄漏。
- 可以为不支持冒泡的事件模拟冒泡过程。
- 提供跨浏览器的 `ready` 事件，以便更早地执行代码。
- 支持自定义事件的绑定、触发和移除。

### 总体结构

![](./img/事件系统总体结构图.png)

### 实现原理

jQuery 并没有将事件监听函数直接绑定到 DOM 元素上，而是基于数据缓存模块来管理监听函数的，对应的数据结构如下图：

![](./img/事件缓存.png)

首先，一个 DOM 对象会按第五章所说的分配一个唯一的 `id` 来关联缓存对象，事件缓存属于内部缓存数据，挂在缓存数据对象的 `events` 属性下面。这个对象被叫做事件缓存对象。它的结构是事件类型 `type` 和监听对象数组 `handles` 的映射。

简单解释一下：每绑定一个监听函数，都会被封装为一个 `handleObj` 对象，加上了图中的属性，用来支持事件模拟、事件代理等。这个事件对象会被加入 `handles` 数组中。每个事件类型 `type`，都会对应一个这样的 `handles`，一个一个元素可以为一种事件绑定多个监听函数。

除了事件缓存对象意外，DOM 元素关联的数据缓存对象还有一个 `handle` 表示主监听函数，负责分发和执行监听函数。对于一个 DOM 元素，jQuery 仅会为其分配一个主监听函数。

###### `.on(events[,selector][,data],handler(eventObject))` 执行过程

1. 监听函数被封装成 `handleObj` 对象，并插入所关联的对象数据 `handles` 中。
2. 如果未绑定过事件，先初始化事件缓存对象为空对象，并为当前元素初始化一个主监听函数。
3. 如果是第一次在元素上绑定某个类型的事件，先把对应的监听对象数组初始化为空数组，并在匹配元素上绑定主监听函数。
4. 底层使用了 `jQuery.event.add`。

###### `.off(events[,selector][,handler(eventObject)])` 执行过程

1. 根据事件类型获取事件监听对象数组 `handles`。
2. 遍历移除匹配的事件监听对象 `handleObj`。
3. 如果 `handles` 变为空数组了，则移除该类型事件的主监听函数。
4. 如果 `events` 为空对象了，那么删除 `events`、`handle` 属性。
5. 底层使用了 `jQuery.event.remove`。

###### 浏览器触发事件执行过程

1. 主监听函数 `handle(event)` 被调用。
2. 取出后代元素匹配的监听代理对象和当前元素的普通监听对象，然后执行。
3. 底层使用了 `jQuery.event.dispatch`。

###### 手动触发事件执行过程

1. `trigger/triggerHandler(eventType, extraParameters)`。
2. 从匹配元素开始向上构造冒泡路径。
3. 触发冒泡路径上的主监听函数、行内监听函数、可选默认行为。
4. 底层使用了 `jQuery.event.trigger`。

### jQuery 事件对象

jQuery 事件系统按照 3 级 DOM 事件模型对浏览器原生事件对象的属性和方法进行了封装和修正，封装后的事件对象成为“jQuery 事件对象”。在事件监听函数执行时，总是会把 jQuery 事件对象作为第一个参数传入。

###### `jQuery.Event(src,props)` 构造函数

1. 运行没有 `new` 的调用。
2. 如果传入的 `src` 为事件对象，那么备份到 `originalEvent` 属性上，然后提取 `type` 并修正 `isDefaultPrevented` 属性。（如果已经被一个更底层的监听函数阻止了默认行为，则为 `true`。）
3. 否则的话 `src` 当做 `event.type` 设置给 `this.type`。
4. `jQuery.extend(this, props)` 扩展自定义事件属性。
5. 修正时间戳 `timeStamp`，IE 9 以下，原生事件对象没有 `timeStamp` 属性。
6. 为当前事件对象设置标记 `jQuery.expando`，这样后面就不需要使用 `instanceof` 来检测是否为 jQuery 事件对象了，高效一点。

###### `jQuery.Event.prototype`

- `isDefaultPrevented()`：默认为始终返回 `false` 的函数。
- `isPropagationStopped()`：默认为始终返回 `false` 的函数。
- `isImmediatePropagationStopped()`：默认为始终返回 `false` 的函数。
- `preventDefault()`：设置 `this.isDefaultPrevented` 为始终返回 `true` 的函数。如果原生事件对象有 `preventDefault` 方法，则调用。对于 IE 9 以下，则设置 `returnValue` 为 `false`。
- `stopPropagation()`：设置 `this.isPropagationStopped` 为始终返回 `true` 的函数。如果原生事件对象有 `stopPropagation` 方法，则调用。对于 IE 9 以下，则设置 `cancelBubble` 为 `true`，因为在 IE 9 以下只有这样才能够组织事件传播。
- `stopImmediatePropagation`：设置 `this.isImmediatePropagationStopped()` 为始终返回 `true` 的函数。然后调用 `this.stopPropagation();` 阻止事件传播。和 `stopPropagation()` 的区别它是会停止执行当前元素上的事件。

### `jQuery.event.fix(event)`

用于封装和修正一个原生事件对象为一个 jQuery 事件对象。

1. 用 `jQuery.expando` 判断是否为 jQuery 事件对象，如果是那么直接返回，无需封装。
2. 获取修正对象，然后合并公共的属性 `props` 和修正对象上的属性 `props`。
3. `event = new jQuery.Event( originalEvent );`。
4. 复制前面合并的事件属性。
5. 修正事件属性 `target`，如果没有 `target` 则从 `srcElement` 中读取。前者是标准，后者是 IE 的。而在 IE 9 以下、Safari 2 中，`load` 事件的 `target`、`srcElement` 都是 `null`，此时修正为 `document`。另外，`target` 不该是一个文本节点，此时修正为父节点。
6. 修正事件属性 `metaKey`，IE 9 以下是 `undefined`，修正为 `false`。在 1.7.1 修正为 ctrl 键，Mac 下是 command 键。
7. 最后如果有 `fixHook`，则需要调用上面的 `filter` 方法来修正键盘或鼠标事件的专属属性。

###### `jQuery.event.keyHooks`

四个专属属性，`char` 和 `key` 在 DOM Level 3 中被标准化，但是除了 IE 9+，其它支持一般，现在应该都已经支持了。而 `charCode`、`keyCode` 是被 IE 9+ 以及其它浏览器都支持的。`which` 是公共属性和 `keyCode` 含义类似，表示“虚拟按键码”。

这里的修正方法就是判断 `which`，如果不存在则从 `charCode` 或 `keyCode` 中读取。 

```
keyHooks: {
	props: "char charCode key keyCode".split(" "),
	filter: function( event, original ) {

		// Add which for key events
		if ( event.which == null ) {
			event.which = original.charCode != null ? original.charCode : original.keyCode;
		}

		return event;
	}
}
```

###### `jQuery.event.mouseHooks`

这个比较长，不贴代码了。

*专属属性：*

- `button`：当事件触发时，那个鼠标按键被按下了。DOM Level 2 规定左中右分别为 0、1、2。但是 IE 9 以下不同，左中右分别为 1、2、4。
- `buttons`：DOM Level 3 提供了更多的按键的支持，默认 0、左中右不变。但是支持如 5 个键的鼠标 1、2、4、8、16。
- `clientX`、`clientY`：相对于窗口左上角的坐标，基本都支持。
- `offsetX`、`offsetY`：相对于事件源元素左上角的坐标。未被标准化，除了 FireFox 都支持，FireFox 应当支持，因为没有修正这个属性。
- `pageX`、`pageY`：相对于整个文档左上角的坐标。未被标准化，IE 9 以下不支持。
- `screenX`、`screenY`：相对于显示器左上角的坐标，基本都支持。
- `fromElement`、`toElement`：前者表示 `mouseover` 中离开的文档元素，后者表示 `mouseout` 中进入的文档元素，没有被标准化，等价于 DOM Level 2 和 3 中的 `relatedTarget`，IE 9 以下仅支持这两个，不支持 `relatedTarget`。

*修正方法：*

1. 针对 `offsetX`、`offsetY`，jQuery 手动计算。
2. 针对 `relatedTarget`，使用 `fromElement` 和 `toElement` 修正。
3. 针对 `which`，IE 9 以下，`which` 不存在。则用 `button` 来修正。左中右分别修正为 1、2、3，默认 0。

### 绑定事件

###### `jQuery.fn.on(types[,selector][,data],fn[,/*INTERNAL*/ one])`

用于为每个匹配元素绑定一个或多个类型的事件监听函数。底层使用了 `jQuery.event.add(elem,types,handler,data,selector)` 来实现，而其他的事件绑定方法都通过调用该方法来实现。

*参数：*

- `types`：事件类型字符串。
 - 多个事件用空格分隔。
 - 命名空间用 `.` 分隔。
 - 还可以是一个 JavaScript 对象，属性是事件类型字符串，值是事件监听函数。
- `selector`：可选的选择器表达式，用于绑定代理事件。事件代理利用的是冒泡机制，由父级来管理子级的事件监听，从而减少绑定的事件数量。
 - 对于普通事件：当直接在匹配元素上触发事件或子级冒泡上来，监听函数被执行。
 - 对于代理事件：直接在代理元素上触发事件，监听函数不执行，只有当事件从子级冒泡上来，才会用 `selector` 匹配子级，然后在匹配成功的子级元素上执行监听函数。
- `data`：传递给监听函数的自定义数据，可以是任何类型。如果是字符串，那么 `selector` 必须为 `null`，不然可能会造成混淆。该参数会被附加到事件监听对象的 `data` 属性上，当事件被触发时会被设置到 jQuery 事件对象的 `data` 属性上。
- `fn`：待绑定的监听函数。这个函数还可以是 `false`，这样会在后面被修正为始终返回 `false` 的函数。
- `one`：内部使用，在 `jQuery.fn.one` 方法绑定监听函数时，会传入 1，后面的代码会把监听函数重新封装为只执行一次的新监听函数。

*执行过程：*

1. 当 `types` 是一个对象。
 1. 修正参数，以 `.on(types, selector, data)` 或 `.on(types, data)` 两种方式。
 2. 遍历对象，递归调用 `this.on( type, selector, data, types[ type ], one );`，调用完直接返回 `this`。 
2. 修正参数。
 1. 如果 `fn` 和 `data` 都是 `null` 或 `undefined`，那么调用方式是 `.on(types, fn)`，把 `selector` 给 `fn`。
 2. 如果 `fn` 是 `null` 或 `undefined`，并且 `selector` 是字符串，那么调用方式是 `.on(types, selector, fn)`，把 `data` 给 `fn`。
 3. 如果 `fn` 是 `null` 或 `undefined`，并且 `selector` 不是字符串，那么调用方式是 `.on(types, data, fn)`，把 `data` 给 `fn`，`selector` 给 `data`。
 4. 如果 `fn` 是布尔值 `false`，那么把它修正为始终返回 `false` 的函数。
3. 如果 `one` 等于 1，那么封装传入的监听函数，在调用原函数前解除事件绑定，达到执行一次的目的。把新旧两个监听函数的 `guid` 设置为相同。
4. 最后遍历匹配元素调用 `jQuery.event.add(this,types,fn,data,selector)` 完成事件绑定。

###### `jQuery.event.add(elem,types,handler,data,selector)`

这个方法为事件绑定提供了底层的支持，参数根据上面方法应该能知道，不重复写一遍，直接写执行过程。

1. `elemData = jQuery._data( elem );`，`elemData` 是缓存数据对象，单参数调用这个方法如果没有缓存数据对象则会初始化一个空对象。
2. 如果 `elemData` 为 `undefined`，说明该元素无法扩展属性，那么直接返回。在 1.7.1 中还派出了文本节点和元素节点。
3. `handler` 是自定义监听对象的情况处理，暂时还不清楚有啥用。
4. 为监听函数分配一个唯一的 `guid`。
5. 取出或初始化事件缓存对象 `events`。
6. 取出或初始化主监听函数。
7. 将 `types` 转为数组，并进行遍历。
 1. 取出 `namespaces` 和 `type`。
 2. 根据事件类型，从 `jQuery.event.special` 获取事件修正对象。关于修正对象下文介绍。
 3. 封装监听函数的监听对象。
 4. 初始化监听对象数组，绑定主监听函数。对于一些特殊事件，会优先调用修正的 `setup` 方法，返回 `false` 时才会调用 `addEventListener\attachEvent` 绑定主监听函数。而在主监听函数内，会调用 `jQuery.event.dispatch` 来触发事件。
 5. 将监听对象插入监听对象数组。修正对象有 `add` 方法，则需要优先调用。
 6. 记录绑定过的事件类型。
8. 解除 `elem` 对 DOM 的引用，避免内存泄漏。

### 移除事件

###### `jQuery.fn.off(types,selector,fn)`

移除匹配元素集合中每个元素上的一个或多个监听函数。参数含义类似 `on`，不介绍了，直接看执行过程吧。

1. 如果 `types` 是一个被分发的 jQuery 事件对象，那么递归调用 `.off` 移除当前正在执行的事件。
2. 如果 `types` 是一个对象，那么遍历移除对象上的属性，类似于 `.on` 方法。
3. 修正参数，支持 `.off(fn)` 的调用，当 `fn` 是 `false` 时，修正为始终返回 `false` 的函数。
4. 遍历匹配元素，调用 `jQuery.event.remove( this, types, fn, selector );` 移除监听函数。

###### `jQuery.event.remove(elem,type,handler,selector,mappedTypes)`

1. 如果该元素没有事件缓存对象，直接返回。
2. 将 `types` 转为数组，并进行遍历。
 1. 取出 `namespaces` 和 `type`。
 2. 如果没有指定事件类型，则移除所有事件或者命名空间下的所有事件。 
 3. 根据事件类型，从 `jQuery.event.special` 获取事件修正对象。
 4. 获取事件对象数组。
 5. 从监听对象数组中移除匹配的监听对象。
 6. 如果监听对象数组为空，则移除该事件类型的主监听函数，优先调用修正对象上的 `teardown` 方法，否则才调用 `removeEventListener/detachEvent`。删除 `events[ type ]`。
3. 如果事件缓存对象为空，移除 `handle` 和 `events` 属性。 

### 事件响应

前面讲到了真正绑定到元素上的只有一个主监听函数，它负责分发事件和执行监听函数。在 `jQuery.fn.on` 方法中我们进行主监听函数的创建和绑定：

```
// 创建
if ( !(eventHandle = elemData.handle) ) {
	eventHandle = elemData.handle = function( e ) {
		return typeof jQuery !== core_strundefined && (!e || jQuery.event.triggered !== e.type) ?
			jQuery.event.dispatch.apply( eventHandle.elem, arguments ) :
			undefined;
	};
	eventHandle.elem = elem;
}
// 绑定
if ( !special.setup || special.setup.call( elem, data, namespaces, eventHandle ) === false ) {
	if ( elem.addEventListener ) {
		elem.addEventListener( type, eventHandle, false );

	} else if ( elem.attachEvent ) {
		elem.attachEvent( "on" + type, eventHandle );
	}
}
```

###### `jQuery.event.dispatch(event)`

从创建主监听函数的代码中我们可以看到底层调用了 `jQuery.event.dispatch` 来分发和执行监听函数。

1. 对于传入的事件对象使用 `jQuery.event.fix` 封装为 jQuery 事件对象。IE 9 以下不会传递事件对象，此时需要通过 `window.event` 来获取。
2. 局部变量定义与初始化。
 1. `handlers`：当前事件类型的缓存监听数组。
 2. `args`：转化为数组的参数。
 3. `special`：事件修正对象。
 4. `handlerQueue`：事件队列。 