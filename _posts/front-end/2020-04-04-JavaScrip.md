---
layout: post
title: JavaScript语言
category: 前端技术
tags: 前端
---

## 概述

JavaScript是动态类型与语言，而Java是静态类型语言。  

学习一门编程语言套路基本差不多，首先是基础的数据类型，对基础数据类型有深刻的理解，那就是学成了一半。然后就是基础的运算符与表达式，最后是一些区别的高级特性如面向对象、并发相关的。

## 基础数据类型

一共六类：string、number、boolean、null、undefined、object

### 为什么有null和undefined类型？
### 类型相同使用 === ，必须类型相同
### 类型不同，尝试类型转换做比较

    console.log(null == undefined);
    
    // true string转number
    console.log(1 == "1.0"); 
    
    // true boolean转number
    console.log(1 == true);
    
    // object == string | number 尝试转为基本类型
    new String("hi") == "hi";


### 包装类型
JS的string、number、bollean有对应的包装类，在对一个基础类型调用方法的时候会生成一个临时的包装类。

    console.log(string.length)
    
    string.t = 1;
    
    console.log(string.t)

如上程序，输出6和undefined，原因是第二行生成的为临时对象，第三行又生成个临时对象但是和第二行的对象是不一样的。

### 类型检测方法
- typeof适合基本类型与function检测，遇到null失效
- instanceof适合自定义对象，也可以用来检测原生对象，在不同iframe和window之间检测失效
- [[classs]]通过{}.toString拿到，适合内置对象基元类型，遇到null和undifined失效

```
console.log(typeof 100)
console.log(typeof NaN)
console.log(typeof undefined)
// 历史问题，返回object
console.log(typeof null)
// 无法判断具体类型，为object 
console.log(typeof [1, 2])

console.log([1, 2] instanceof Array)
// false是会比较类型的，与java的instanceOf类似
console.log(new Object instanceof Array)
```

### 运算符
和java大多相似，有个，运算符，使用最右侧的计算结果

```
// tmp计算后为3
var tmp = (1, 2,3)
```

## 语句
### 没有块级的作用域
js的块并没有起到作用域的目的，但是有函数作用域

### var语句
// b是个全局变量，未声明，直接赋值未全局变量，在use strict模式下是不允许的，会报错Reference Error
var a=b=1;

```
function foo(){
    var a=b=1;
}
foo();
console.log(a) // 3
console.log(b) // undifined
```

### 函数声明vs函数表达式
```
// 函数声明作用域为全局
console.log(fe()) // true
function fe(){
    return true;
}

// 函数表达式本质为一个变量的声明，作用域与变量类似
//console.log(fd()); // fd not a function
var fd = function (){
    return true;
}
console.log(fd()); // true
```

### with修改作用域
不在建议使用，js不好优化，可读性差。
```
with({x:1}) {
    console.log(x);
}
```

### 严格模式
use restrict，
- 不允许使用with
- 所有变量必须先声明
- arguments变为参数的静态副本
- delete 参数、函数名会报语法错误
- delete不可配置的属性报错
- 对象字面量重复属性名报错
- 禁止八进制字面量
- eval、arguments变为关键字，不能作为变量函数名
- eval为独立作用域

## 对象
### prototype与属性标签
- prototype属性是属于整个类的，类似于java中的static属性
- 属性标签的作用会对属性的一些特性进行约束，如是否可以修改。起到了类似java对于属性的一些修饰关键字如final的作用
- configurable表示此属性是否能被删除、修改（delete、get、set、修改属性标签）
- writeable表示此属性是否可以修改（通过属性赋值的方式）
- enumrable是否能被遍历
- 如果configurable，writeable为false，那么不能修改任何标签属性、更不能通过赋值的方式，是个彻底的final

```
var descriptor = Object.getOwnPropertyDescriptor(Object, 'prototype')
console.log(descriptor.configurable) // false
```

### 标签修改
```
var obj = {x:1, y:2};
Object.preventExtensions(obj);
// Object {value: 1, writable: true, enumerable: true, configurable: true}
console.log(Object.getOwnPropertyDescriptor(obj, 'x'));
Object.seal(obj);
// Object {value: 1, writable: true, enumerable: true, configurable: false} configurable 为false
console.log(Object.getOwnPropertyDescriptor(obj, 'x'));
Object.freeze(obj)
// Object { value: 1, writable: false, enumerable: true, configurable: false } writable 和 configurable都为false
console.log(Object.getOwnPropertyDescriptor(obj, 'x'));
```

## 数组
JS中的数组是弱类型的，一个数组内可以有不同类型的元素

### 方法
#### join
```
var array = [1, 2, 3]
// 输出1+2+3
console.log(array.join("+"))
// @@@
console.log(new Array(3).join('@'))
```

#### reverse
```
var array = [1, 2, 3]
array.reverse();
// 会对原数组改变
console.log(array)
```

#### sort
```
var arr = [11, 30, 2]
// 字母顺序排序
arr.sort()
console.log(arr)
// 从小到大排序
arr.sort(function(a, b){
    return a - b;
})
console.log(arr)
```

#### concat
```
var arr = [1, 2, 3]
// [1, 2, 3, 4, 5, 6]
console.log(arr.concat(4, [5, 6]))
// 不会改变原数组
console.log(arr)
// [1, 2, 3, 4, Array(2)]， 不会拉平两次
console.log(arr.concat(4, [[5, 6]]))
```

####slice
```
var arr = [1, 2, 3, 4, 5]
// 左闭右开 [2, 3],原数组未修改
console.log(arr.slice(1, 3))
// 第二个参数默认-1
console.log(arr.slice(1))
// 倒数第四个到倒数第三个 2
console.log(arr.slice(-4, -3))
```

#### splice
```
var arr = [1, 2, 3, 4, 5]
// 从第三个元素开始切割，切割原来数组，[3, 4, 5]
console.log(arr.splice(2))
// 对原来数组进行了改变 [1, 2]
console.log(arr)

// 从第三个元素开始切割，切割两个
arr = [1, 2, 3, 4, 5]
// [3, 4]
console.log(arr.splice(2, 2))
// [1, 2, 5]
console.log(arr)

arr = [1, 2, 3, 4, 5]
// 切割两个并加入两个 [3, 4]
console.log(arr.splice(2, 2, 'a', 'b'))
// [1, 2, 'a', 'b', 5]
console.log(arr)
```

#### foreach
```
var arr = [1, 2, 3, 4, 5]
// value,index,数组自身
// 1 | 0 | true
// 2 | 1 | true
// 3 | 2 | true
// 4 | 3 | true
// 5 | 4 | true
arr.forEach(function(x, index, a){
    console.log(x + '|' + index + '|' + (a === arr))
})
```

#### map
```
var arr = [1, 2, 3, 4, 5]
// 对每个元素操作，并返回新数组  [11, 12, 13, 14, 15]
console.log(arr.map(function(x){
    return x + 10;
}))
// 不会对原来数组改变 [1, 2, 3, 4, 5]
console.log(arr)
```

#### filter
```
var arr = [1, 2, 3, 4, 5, 6]
// 元素筛选  [2, 4, 5, 6]
console.log(arr.filter(function(x){
    return x % 2 == 0 || x > 4;
}))
// 不会对原来数组改变 [1, 2, 3, 4, 5, 6]
console.log(arr)
```

#### every & some
```
var arr = [1, 2, 3, 4, 5, 6]
// 任何一个都满足条件，false
console.log(arr.every(function(x){
    return  x > 4;
}))
// 有一个满足条件,true
console.log(arr.some(function (x) {
    return x > 4;
}))
```

#### reduce
```
var arr = [1, 2, 3, 4, 5, 6]
// 第二个参数为第一次迭代时候x的值
var sum = arr.reduce(function(x, y){
    return x + y;
}, 100)
console.log(sum)
var max = arr.reduce(function(x, y){
    return x > y ? x : y
})
console.log(max)
```

#### indexOf
```
var arr = [1, 2, 3, 2, 1]
// 1
console.log(arr.indexOf(2))
// -1
console.log(arr.indexOf(99))
// 4
console.log(arr.indexOf(1, 1))
// 4
console.log(arr.indexOf(1, -3))
// 3
console.log(arr.lastIndexOf(2))
// 1
console.log(arr.lastIndexOf(2, -3))
```

## 函数
### 函数声明、函数表达式、函数构造器
- JS中的函数也是对象，可以像其他对象一样传递和操作。
- 函数声明会被提前，变量声明也会提前。但是赋值操作不会前置
- 函数表达式与函数构造器可以立即执行。而函数声明不会，因为会前置，前置后只剩下一对()。

```
// 函数表达式，有名字的函数对像，可以用在递归的场景
var fun = function nfe(){};

// 可以用函数构造器（更像对象了）, new 也可以省略
var func = new Function('a', 'b', 'console.log(a + b)')
func(1, 2)

// func构造器不能访问函数对象外层的局部变量，能访问全局变量 undefined string
var globalVal = 'global';
(function(){
    var localVal = 'localVal';
    Function('console.log(typeof localVal, typeof globalVal)')();
})();
```

### this
#### 全局的this（浏览器）
```
//浏览器下，为window
console.log(this === window)

this.a = 37
// 37
console.log(window.a)
```

#### 对象方法函数的this
```
// this 代表对象本身
var o = {
    prop : 33,
    f : function() {
        return this.prop;
    }
};
console.log(o.f())

// 还可以这样去赋值、调用
var o = {prop:33}
function func(){
    return this.prop;
}
o.f = func;
console.log(o.f())
```

#### 原型链上的对象this
```
// 与java中继承差不多
var o = {f:function(){ return this.a + this.b}}
var p = Object.create(o)
p.a = 1;
p.b = 2;
console.log(p.f())
```


#### 构造器中的this
```
// 函数对象的构造器，也能当类使用，会返回this，如果返回的是个基本类型，也会返回this
function MyClass(){
    this.a = 33;
}

var o = new MyClass();
console.log(o.a)

// 但是如果返回一个对象，this就是返回的对象
function MyClass(){
    this.a = 33;
    return {a:34}
}

var o = new MyClass();
console.log(o.a)
```

#### call与apply的this
```
// call,apply参数为数组
function add(a, b){
    return this.c + this.d + a + b;
}
var o = {c:1, d:2}
console.log(add.call(o, 3, 4))
console.log(add.apply(o, [3, 4]))
```

#### bind与this
```
// bind方法能把函数中的this与对象绑定在一起,变成两外一个已绑定的函数对象
function f(){
    return this.a;
}

var g = f.bind({a:'test'})
console.log(g())

var o = {a:22, f:f, g:g}
console.log(o.f(), o.g())
```

### arguments
- arguments能获取实际传参的个数，通过函数名.length 能得到函数声明传参的个数
- 严格模式下，arguments[0] = xx这种方式是不能改变参数的实际的值的

#### bind与currying
```
// 通过指定特定参数能够生成其他的函数对象
function foo(a, b, c){
    return a + b + c;
}

var fun = foo.bind(undefined, 100)
console.log(fun(1, 2))

var fun2 = fun.bind(undefined, 200)
console.log(fun2(10))
```

### 闭包
与c++中的不一样，c++中的是拷贝了个变量的副本，但js中的好像是保存了对原来环境中的变量的引用。如下，两次调用后，原来函数内的v会被修改。

```
function foo(){
    var v = 'hello';
    return function(){
        v += ' world';
        return v;
    }
}

var func1 = foo();
// hello world
console.log(func1())
// hello world world
console.log(func1())
```

### 作用域
JS很奇怪，没有块级别的作用域，也就是说在一半的for循环如for（var i=0； i<10; i++）在外部也能访问到i的，所以一半for循环不建议定义在括号内，而是定义在括号外。

没有块级作用域，那像定义局部变量有什么方式？通过函数作用域：
```
// 通过函数
(function(){
    var a = 1;
})();

// ！代替括号，如果没有会变成函数声明，需要用函数表达式方式，并且带个()立即执行
!function(){
    var a = 1;
}();

```

### 执行上下文与变量对象
上下文并不是一个概念，竟然有个对象，如在全局上下文中有个对象GlobalContextVO，在NodeJs中是gloable，在浏览器中是windows。

```
// 为什么能在js中直接调用Math、isNaN等方法（对象）？JS没有静态方法的概念，一切都是对象，因为js的引擎会在第一行初始化全局的上下文对象
String(10); // [[global]].String(10);
var a = 10
// 10,在浏览器中
console.log(window.a) 
```

### VO(变量对象)初始化
VO按照如下顺序进行填充
- 函数传参（若未传，则赋值为undefined）
- 函数声明（若发生冲突，则覆盖）
- 变量声明（初始化为undefined，若冲突，则忽略）

```
function fun(x, y, z){
    function x(){};
    console.log(x)
}
// function x(){ … } 覆盖
fun(100)

function fun1(x, y, z) {
    function x() { };
    var x;
    console.log(x)
}

// function x() { … },变量重复忽略
fun1();
```

```
// function x(){ … }  初始化（全局的无函数参数），先找函数声明
console.log(x)
// 没有局部作用域的概念，所以a为undefined
console.log(a)

var x = 10;
// 在赋值阶段对x进行赋值，赋值后为10
console.log(x)
x = 20;

function x(){};
// 20 因为函数声明提前，是赋值20后的值
console.log(x);

if(true){
    var a = 1;
} else {
    var b = true;
}

// 1
console.log(a)
// undefined
console.log(b)
```

## OOP
#### 原型链
```

// 创建一个类，直接用function可以，但是有些奇怪，如果是函数一为小写开头
function Person(name, age) {
    this.name = name;
    this.age = age;
}

// vscode 建议转化为一个新的类
class PersonNew {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
}

Person.prototype.hi = function() {
    console.log("Hi my name is " + this.name + ", I'm " + this.age + "years old.")
}

Person.prototype.walk = function() {
    console.log(this.name + " is walking.")
}

var p = new Person("bobo", 3);
p.hi();
p.walk();

function Student(name, age, className) {
    Person.call(this, name, age);
    this.className = className;
}

// Object.create创建一个对象，并且对象的原型指向参数，兵器赋值给Student.prototype，原来的Student.prototype指向Object.prototype
Student.prototype = Object.create(Person.prototype);
// 可以不写
Student.prototype.construct = Student;

Student.prototype.hi = function() {
    console.log("Hi my name is " + this.name + ", I'm " + this.age + "years old" + ", and from "
    + this.className)
}

var boboStu = new Student("bobo", 3, "Class 3, grade 2")
boboStu.hi();
// bobo is walking.
boboStu.walk();

// 每个对象都有个prototype的对象，这个对象里面有个__proto__属性（属性叫原型），而这个原型指向的为另外一个对象（父类）的prototype对象
// prototype对象上可以有很多的属性和方法
// 调用的toString、valueOf、hasOwnProperty等方法，都是继承自Object的prototype
// Object的prototype为空
// null
console.log(Object.prototype.__proto__)
// function toString() { … }
console.log(Object.prototype.toString)

function foo() {}
// true 如果是一个新的对象，默认的prototype为Object的prototype
console.log(foo.prototype.__proto__ === Object.prototype)


// 一个对象可以没有原型
var obj = Object.create(null)
// undefine
console.log(obj.toString)

Student.prototype.x = 111
// 动态修改原型上的属性，对于已实例化的对象是会影响的
console.log(boboStu.x)

Student.prototype = {y:2}
// undefined，已经实例化后，在修改原型是不可以的
console.log(boboStu.y)

Person.prototype.ppp = 2;
// true
console.log('ppp' in boboStu)
// false 判断是否为自己的属性
console.log(boboStu.hasOwnProperty('ppp'))
// 父类的非prototype的属性也是 true
console.log(boboStu.hasOwnProperty('name'))
```

### instanceof
```

// instanceof 判断左边对象的原型链上是否有右侧函数出现
// true
console.log([1, 2] instanceof Array)
// false
console.log(new Object instanceof Array)
// true
console.log([1, 2] instanceof Object)
```

## Demo 
```
// 匿名函数限定作用域的作用，this为Window或global
!function(global) {
    function DetectorBase(configs) {
        if(!this instanceof DetectorBase) {
            throw new Error("Do not invoke without new")
        }
        this.configs = configs;
    }

    // 模拟接口
    DetectorBase.prototype.detect = function(){
        throw new Error("not Implemented.")
    }

    // 基类实现
    DetectorBase.prototype.analyze = function(){
        console.log("base analyzing...")
        this.data = "###data###";
    }

    function LinkDetector(links) {
        DetectorBase.apply(this, arguments);
        if(! this instanceof LinkDetector){
            throw new Error("Do not invoke without new")
        }
        this.links = links;
    }

    inherit(LinkDetector, DetectorBase)

    function inherit(subClass, superClass) {
        subClass.prototype = Object.create(superClass.prototype)
        superClass.prototype.constructor = subClass;
    }

    LinkDetector.prototype.detect = function(){
        console.log("loading data: " + this.data)
        console.log("Scaning links " + this.links)
    }

    // 防止修改
    Object.freeze(DetectorBase)
    Object.freeze(DetectorBase.prototype)
    Object.freeze(LinkDetector)
    Object.freeze(LinkDetector.prototype)

    // 必须导出才能使用
    Object.defineProperties(global, {
        // 其他属性默认为false
        DetectorBase : {value : DetectorBase},
        LinkDetector: { value: LinkDetector}
    })

}(this)

// 不能直接调用 ReferenceError: DetectorBase is not defined
// DetectorBase.detect();

// vscode 下必须加个this,浏览器正常
var detectorBase = new this.DetectorBase();
detectorBase.analyze();
var linkDetector = new this.LinkDetector("http://www.baidu.com");
linkDetector.detect();
```
## 参考
[慕课-JavaScript深入浅出](https://www.imooc.com/learn/277)