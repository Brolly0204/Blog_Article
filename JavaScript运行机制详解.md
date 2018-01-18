# JavaScript 运行机制详解

## 一、JavaScript单线程模型

> JavaScript是单线程的，JavaScript只在一个线程上运行，但是浏览器是多线程的，典型的浏览器有如下线程：
- JavaScript引擎线程
- GUI渲染线程
- 浏览器事件触发线程
- 浏览器Http请求线程

## 二、JavaScript为什么是单线程的

> &ensp;&ensp;JavaScript之所以采用单线程 而不是多线程，由于浏览器脚本语言，主要用途是与用户互动，以及操作DOM（文档对象模型）和BOM（浏览器对象模型）， 而多线程需要共享资源，多线程编程经࣡常面临锁、状态同步等问题。<br/>
  &ensp;&ensp;假定JavaScript同时有两个线程，这两个线程同时操作同一个DOM增删修改操作，这时浏览器应该以哪个线程操作为准？无疑会带来同步问题。<br/>
  &ensp;&ensp;既然JavaScript是单线程的，这就意味着，一次只能运行一个任务，其他任务都必须在后面排队等待
  &ensp;&ensp;为了利用多核CPU的计算能力，HTML5提出了Web Worker，它会在当  前JavaScript的执行主线程中利用Worker类新开辟一个额外的线程来加载和运行特定的JavaScript文件，但在HTML5 Web Worker中是不能操作DOM的，任何需要操作DOM的任务都需要委托给JavaScript主线程来执行，所以虽然引入HTML5 Web Worker，但仍然没有改变JavaScript单线程的本质。

## 三、JavaScript如何工作的，首先要理解以下几个概念

- JS Engine(JS引擎)
- Runtime(运行上下文)
- Call Stack(调用栈)
- Event Loop(事件循环)
- Callback(回调)

### 1.JS Engine

JavaScript引擎就是用来执行JS代码的, 通过编译器将代码编译成可执行的机器码让计算机去执行（Java中的JVM虚拟机一样）。
> 常见的JavaScript虚拟机（一般也把虚拟机称为引擎）：
- Chakra(Microsoft Internet Explorer)
- Nitro/JavaScript Core (Safari)
- Carakan (Opera)
- SpiderMonkey (Firefox)
- V8 (Chrome, Chromium)
目前比较流行的就是V8引擎，Chrome浏览器和Node.js采用的引擎就是V8引擎。
引擎主要由堆(Memory Heap)和栈(Call Stack)组成

<div align="center">
<img src="./images/headandstack.png" width="65%" style="margin:atuo">
</div>
<br/>
- Heap（堆） - JS引擎中给对象分配的内存空间是放在堆中的
- Stack（栈）- 这里存储着JavaScript正在执行的任务。每个任务被称为帧（stack of frames）。
主线程运行的时候，产生堆（heap）和栈（stack）,栈中的代码调用个各种外部api。

### 2.RunTime (运行环境)
JS在浏览器环境中运行时，BOM和DOM对象提供了很多相关外部接口（这些接口不是V8引擎提供的），供JS运行时调用，以及JS的事件循环(Event Loop)和事件队列(Callback Queue)，把这些称为RunTime。在Node.js中，可以把Node的各种库提供的API称为RunTime
<br/>
<br/>
<div align="center">
<img src="./images/runtime.png" width="65%" style="margin:atuo">
</div>

### 3.Call Stack
当JavaScript代码执行的时候，执行环境是很重要的，它可能是下面三种情况中的一种：

- 全局 code（Global code）——代码第一次执行的默认环境
- 函数 code（Function code）——执行流进入函数体
- Eval code（Eval code）——代码在eval函数内部执行
JavaScript代码首次被载入时，会创建一个全局上下文，当调用一个函数时，会创建一个函数执行上下文。
栈是一种遵从后进先出（LIFO）原则的结构。函数被调用时，就会被加入到调用栈顶部，执行结束之后，就会从调用栈顶部移除该函数。

代码运行时，首先调用main方法，在里面生成一个Student实例，在Student构造函数里，又调用sayHi方法
```
class Student {
	constructor(age, name) {
		this.name = name;
        this.age = age;
		this.sayName(); // stack 3
	}
	sayName() {
		console.log(`my name is ${this.name}, this year age is ${this.age}`);
	}
}

function main(age, name) {
	new Student(age, name); // stack 2
}

main(23, 'John'); // stack 1
```
<div align="center">
<img src="./images/stack.gif" width="70%" style="margin:atuo">
</div>

> 程序运行时，首先main()函数的执行上下文入栈，再调用Student构造函数添加到当前栈尾，在Student里再调用sayName()方法，添加到此时栈尾。最终main方法所在的位置叫栈底，sayName方法所在的位置是栈顶，层层调用，直至整个调用栈完成返回结果，最后再由栈顶依次出栈。

### 4.Event Loop & Callback
Event Loop 类似于一个while(true)的循环，每执行一次循环体的过程我们成为Tick。每个Tick的过程就是查看是否有事件待处理，当Call Stack里面的调用栈运行完变成空了，就取出事件及其相关的回调函数。放到调用栈中并执行它。
<div align="center">
<img src="./images/eventloop.png" width="70%" style="margin:atuo">
</div>

>调用栈中遇到DOM操作、ajax请求以及setTimeout等WebAPIs的时候就会交给浏览器内核的其他模块进行处理，webkit内核在Javasctipt执行引擎之外，有一个重要的模块是webcore模块。对于图中WebAPIs提到的三种API，webcore分别提供了DOM Binding、network、timer模块来处理底层实现。等到这些模块处理完这些操作的时候将回调函数放入任务队列中，之后等栈中的task执行完之后再去执行任务队列之中的回调函数。


```
console.log('Hi');
setTimeout(function cb1() {
    console.log('cb1');
}, 5000);
console.log('Bye');
```
<div align="center">
<img src="https://user-gold-cdn.xitu.io/2018/1/16/160fcd26f8023a85?imageslim" width="70%" style="margin:atuo">
</div>

## 四、任务队列

> Javascript有一个main thread 主进程和call-stack（一个调用堆栈），在对一个调用堆栈中的task处理的时候，其他的都要等着。当在执行过程中遇到一些类似于setTimeout等异步操作的时候，会交给浏览器的其他模块(以webkit为例，是webcore模块)进行处理，当到达setTimeout指定的延时执行的时间之后，task(回调函数)会放入到任务队列之中。一般不同的异步任务的回调函数会放入不同的任务队列之中。等到调用栈中所有task执行完毕之后，接着去执行任务队列之中的task(回调函数)。

### 1.同步任务

同步任务是指，在主线程上排队执行的任务，只有前一个任务执行完毕，才能执行后一个任务

### 2.异步任务

异步任务是指，不进入主线程，而进入“任务队列”（task queue）的任务。等主栈空闲，然后通过Event Loop机制把排在队列前面的回调压入主栈

## 五、event loop的事件处理机制

> event queue 是用来存储回调函数等待被event loop处理的。但实际上不止一个 event queue队列，事件循环要处理的主要有4个类型的队列：
- Times and Intervals Queue: 保存setTimeout 和 setInterval 中的回调函数
- IO Event Queue: 保存已经完成的I/O回调函数
- Immediates Queue: 保存setImmediate中的回调函数
- Close Handlers Queue: 其他所有close事件的回调。
除了上述四个主要队列外，还有两个比较特殊的队列：
- Next Ticks Queue: 保存process.nextTick中的回调函数
- Other Microtasks Queue: 保存promise等microtask中的回调函数。

>件循环会依次处理timers and intervals queue，IO event queue，immediates queue，close handlers queue这四个队列，如果处理完close hanlers queue后，timers and intervals没有数据再进来，就退出事件循环。
处理其中一个队列的过程称为一个phase。一次事件循环就是处理这四个phase的过程。那另外两个特殊的队列是在什么时候运行的呢？ 答案就是在每个 phase运行完后马上就检查这两个队列有无数据，有的话就马上执行这两个队列中的数据直至队列为空。当这两个队列都为空时，event loop 就会接着执行下一个phase。
这两个队列相比，Next Ticks Queue的权限要比Other Microtasks Queue的权限要高，因此Next Ticks Queue会先执行。
此外要注意的是，如果process.nextTick中出现递归调用没有停止条件的话，Next Ticks Queue将一直有数据进来一直都不会为空，则会阻塞event loop的执行。为了防止该情况，process.maxTickDepth定义了迭代的最大值，不过从NodeJS v0.12版本开始已经移除了。

如果程序中既有setTimeout和setImmediate，两者的执行顺序是什么？
```
setTimeout(function timeout() {
  console.log('timeout');
}, 0);

setImmediate(function immediate() {
  console.log('immediate');
});
```
## 参考
- https://juejin.im/post/5a5e03eef265da3e5033c5b9
