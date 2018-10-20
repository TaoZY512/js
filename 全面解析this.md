# 全面解析this

this是一个很特别的关键字，也是Javascript中最复杂的机制之一，它被自动定义在所有函数的作用域中。如果你想要熟练的去使用他，那么你将需要了解一下几个问题。

### 1.为什么要用this

下面我们来解释一下为什么要使用`this`：

```
function identify() {
    return this.name.toUpperCase();
}

function speak() {
    var greeting = "Hello, I'm " + identify.call( this );
    console.log( greeting );
}

var me = {
    name: "Kyle"
};
var you = {
    name: "Reader"
};

identify.call( me ); // KYLE
identify.call( you ); // READER

speak.call( me ); // Hello, 我是KYLE
speak.call( you ); // Hello, 我是READER
```

这段代码可以在不同的上下文对象（`me`和`you`）中重复使用函数`identify()`和`speak()`，不用针对每个对象编写不同版本的函数。

如果不使用`this`，那就需要给`identify()`和`speak()`显式传入一个上下文对象。

```
function identify(context) {
    return context.name.toUpperCase();
}

function speak(context) {
    var greeting = "Hello, I'm " + identify( context );
    console.log( greeting );
}

identify( you ); // READER
speak(me); //Hello, 我是KYLE
```

当然，使用显示可以使代码看起来更直观，但是随着你的使用模式越来越复杂，显式传递上下文对象会让代码变得越来越混乱，使用`this`则不会这样。

### 2.它的作用域

`this`指向函数的作用域。这个问题有点复杂，因为在某种情况下它是正确的，但是在其他情况下它却是错误的。  需要明确的是，`this`在任何情况下都不指向函数的词法作用域。在JavaScript内部，作用域确实和对象类似，可见的标识符都是它的属性。但是作用域“对象”无法通过JavaScript代码访问，它存在于JavaScript引擎内部。我们来观察一下下面的代码。

```
function foo() {
    var a = 2;
    this.bar();
}

function bar() {
    console.log( this.a );
}

foo(); // ReferenceError: a is not defined
```

它试图（但是没有成功）跨越边界，使用`this`来隐式引用函数的词法作用域。

首先，这段代码试图通过`this.bar()`来引用`bar()`函数。这是绝对不可能成功的，调用`bar()`最自然的方法是省略前面的`this`，直接使用词法引用标识符。

此外，还试图使用`this`联通`foo()`和`bar()`的词法作用域，从而让`bar()`可以访问`foo()`作用域里的变量`a`。这是不可能实现的，你不能使用`this`来引用一个词法作用域内部的东西。

每当你想要把`this`和词法作用域的查找混合使用时，一定要提醒自己，这是无法实现的。

### 3.调用的位置

在理解`this`的绑定过程之前，首先要理解调用位置：调用位置就是函数在代码中被调用的位置（而不是声明的位置）

通常来说，寻找调用位置就是寻找“函数被调用的位置”，但是做起来并没有这么简单，因为某些编程模式可能会隐藏真正的调用位置。

最重要的是要分析调用栈（就是为了到达当前执行位置所调用的所有函数）。我们关心的调用位置就在当前正在执行的函数的前一个调用中。

下面我们来看看到底什么是调用栈和调用位置：

```
function baz() {
    // 当前调用栈是：baz
    // 因此，当前调用位置是全局作用域

    console.log( "baz" );
    bar(); // <-- bar的调用位置
}

function bar() {
    // 当前调用栈是baz -> bar
    // 因此，当前调用位置在baz中

    console.log( "bar" );
    foo(); // <-- foo的调用位置
}

function foo() {
    // 当前调用栈是baz -> bar -> foo
    // 因此，当前调用位置在bar中

    console.log( "foo" );
}

baz(); // <-- baz的调用位置
```

注意我们是如何（从调用栈中）分析出真正的调用位置的，因为它决定了`this`的绑定。

### 4.绑定的规则

如果要判断一个运行中函数的`this`绑定，就需要找到这个函数的直接调用位置。找到之后就可以顺序应用下面这四条规则来判断`this`的绑定对象。

- 由`new`调用？绑定到新创建的对象。
- 由`call`或者`apply`（或者`bind`）调用？绑定到指定的对象。
- 由上下文对象调用？绑定到那个上下文对象。
- 默认：在严格模式下绑定到`undefined`，否则绑定到全局对象。

##### 4.1  默认绑定

思考一下下面的代码：

```
function foo() { 
    console.log( this.a );
}

var a = 2;

foo(); // 2
```

我们可以看到当调用`foo()`时，`this.a`被解析成了全局变量`a`。为什么？因为在本例中，函数调用时应用了`this`的默认绑定，因此`this`指向全局对象。

那么我们怎么知道这里应用了默认绑定呢？可以通过分析调用位置来看看`foo()`是如何调用的。在代码中，`foo()`是直接使用不带任何修饰的函数引用进行调用的，因此只能使用默认绑定，无法应用其他规则。

如果使用严格模式（`strict mode`），那么全局对象将无法使用默认绑定，因此`this`会绑定到`undefined`

```	
function foo() { 
    "use strict";
     console.log( this.a ); 
}

var a = 2;

foo(); // TypeError: this is undefined
```

##### 4.2 上下文

第三方库的许多函数，以及JavaScript语言和宿主环境中许多新的内置函数，都提供了一个可选的参数，通常被称为“上下文”（context），其作用和`bind(..)`一样，确保你的回调函数使用指定的`this`。

举例来说：

```
function foo(el) { 
    console.log( el, this.id );
}

var obj = {
    id: "awesome"
};

// 调用foo(..)时把this绑定到obj
[1, 2, 3].forEach( foo, obj );
// 1 awesome 2 awesome 3 awesome
```

这些函数实际上就是通过`call(..)`或者`apply(..)`实现了显式绑定，这样你可以少些一些代码。

##### 4.3  `new`绑定 

思考下面的代码：

```
function foo(a) { 
    this.a = a;
} 

var bar = new foo(2);

console.log( bar.a ); // 2
```

使用`new`来调用`foo(..)`时，我们会构造一个新对象并把它绑定到`foo(..)`调用中的`this`上。`new`是最后一种可以影响函数调用时`this`绑定行为的方法，我们称之为`new`绑定。



一定要注意，有些调用可能在无意中使用默认绑定规则。如果想“更安全”地忽略`this`绑定，你可以使用一个DMZ对象，比如`ø = Object.create(null)`，以保护全局对象。

ES6中的箭头函数并不会使用四条标准的绑定规则，而是根据当前的词法作用域来决定`this`，具体来说，箭头函数会继承外层函数调用的`this`绑定（无论`this`绑定到什么）。这其实和ES6之前代码中的`self = this`机制一样。



参考书：《你不知道的Javascript(上卷)》