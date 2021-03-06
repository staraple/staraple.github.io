---
published: true
layout: post
title: Making Promises With jQuery Deferred
---

Promises 就像生小孩，怀上容易生出来难。 --佚名

你可能会将处理器指定给元素的 `onclick` 属性，就像 `mywidget.onclick = myhandler` , 从而将处理器和鼠标点击事件联系起来。如果其他方法也想指定给该点击事件就会有问题，因为一个属性只能被指定一个函数。后来，这个问题被DOM函数 `addEventListener` 所解决，因为这个函数允许你为事件添加多个监听器。快进到现在，在使用Ajax调用时出现类一个相似的问题。Ajax的限制是只支持一个回调函数。jQuery在1.5版本时引入类 `Deferred` 对象来解决这个问题。它能够将多个回调注册进回调队列，触发回调队列，并且能推迟发送任何同步或异步函数的成功或失败状态。本文中，我们将学习如何使用 `Deferred` 对象来实现 `Promises`。

## Promise 里包含了什么?
在jQuery 1.5之前，一个典型的Ajax调用是这样的：

```javascript
$.ajax({
    url: "/serverResource.txt",
    success: successFunc,
    error: errorFunc
});
```

1.5之后，Ajax调用返回的对象(jqXHR)实现了[CommonJS Promises/A](http://wiki.commonjs.org/wiki/Promises/A)接口，这带来了极大的灵活性。

```javascript
var promise = $.ajax({
    url: "/serverResource.txt"
});
promise.done(successFunc);
promise.fail(errorFunc);
promise.always(alwaysFunc);
```

`always` 处理器，对应了jQuery1.6之前的 `complete()` ,无论 `Ajax` 调用结果如何，都在 `done()` 或者 `fail()` 事件之后被调用。
`done()`, `fail()` 和 `always` 三者都返回一样的 `jQuery XMLHttpRequest` 对象，所以可以串行使用：

```javascript
$.ajax("example.php")
    .done(function(){alert("success");})
    .fail(function(){alert("error");})
    .always(function(){alert("complete")});
```

你也可以将jqXHR对象保存到一个变量中：

```javascript
var jqxhr = $.ajax("example.php")
    .done(function(){alert("success")})
    .fail(function(){alert("fail")})
    .always(function(){alert("complete")});
//做点其他的事情
//设置另一个完成调用后要的处理器
jqxhr.always(function(){alert("another complete")});
```

另一个结合处理器的方式时使用Promise接口中的 `then()` 方法，它可以接受三个处理器作为参数。对于jQuery,在1.8之前，你可以传一个函数数组给这个方法：

```javascript
$.ajax({url:'/serverResource.txt'})
    .then(
        [successFunc1, successFunc2, successFunc3],
        [errorFunc1, errorFunc2]
);

//same as
var jqxhr = $.ajax({url:'/serverResource.txt'});
jqxhr.done(successFunc1);
jqxhr.done(successFunc2);
jqxhr.done(successFunc3);
jqxhr.fail(errorFunc1);
jqxhr.fail(errorFunc2);
```

1.8之后，`then()` 方法返回一个新的 `promise`, 它可以通过函数过滤一个 `deferred` 对象的状态和数值。如果不需要对特定的事件类型指定处理器，可以传 `null` 值。

```javascript
var promise = $.ajax({url:'/serverResource.txt'});
promise.then(successFunc, errorFunc);

var promise = $.ajax({url:'/serverResource.txt'});
promise.then(successFunc);
```


## 串联then()函数

```javascript
var promise = $.ajax('/serverScript1');
function getStuff(){
    return $.ajax('/serverScript2');
}
promise.then(getStuff).then(function(serverScript2Data){
    // 
});
```

## 合并 Promises
`Promise` 方法 `$.when()` 方法等同于逻辑和操作符。传一堆 `Promise` 给它，它会返回一个新的 `Promise` :
- 当所有给定的 `Promises` 都解决了，新的 `Promise` 就被解决了。
- 有任一 `Promise` 被拒绝，新的 `Promise` 就被拒绝。

下面的代码使用 `when()` 方法来同时进行两个 `Ajax` 调用，并在两个调用都完成后执行一个函数：

```javascript
var jqxhr1 = $.ajax('/serverResource1.txt');
var jqxhr2 = $.ajax('/serverResource2.txt');
$.when(jqxhr1, jqxhr2).done(function(result1, result2){
    //处理两个返回值
    //
    alert("all complete");
})
```


## Promise的状态
任何时候，`promises` 可能处于三种状态之一: `unfulfilled` , `resolved` 或者 `rejected`。`promise` 的默认状态是 `unfulfilled` 。任何定义到队列中的处理器会在稍后被执行。当一个 `Ajax` 调用成功后，`$.resolved` 就会被调用，它的状态被设置成`resolved`，所有分配给 `done` 的处理器会被一一执行。同样，如果调用失败，`$.rejected` 就会被调用，所有分配给 `fail` 的处理器会被执行。

## 创建自己的Deferred过程
通过 `jQuery.Deferred()` 方法创建一个新的 `Deferred` 对象，我们可以配置我们自己的 `deferred` 过程。在下面的例子中，一个 `<div>` 或者 `<span>` 元素会基于过程的状态而被更新。

```javascript
var timer;
$('#result').html('waiting...');
var promise = process();
promise.done(function(){
    $('#result').html('done.');
});
promise.progress(function(){
    $('#result').html($('#result').html() + '.');
});
function process(){
    var deferred = $.Deferred();
    timer = setInterval(function(){
        deferred.notify();
    }, 1000)
    setTimeout(function(){
        clearInterval(timer);
        deferred.resolve();
    }, 10000);
    return deferred.promise();
}
```

使用 `then()` 方法后可以这样写：

```javascript
var timer;
(function (){
    $('result').html('waiting...');
    var deferred = $.Deferred();
    timer = setInterval(function(){
        deferred.notify();
    }, 1000)
    setTimeout(function(){
        clearInterval(timer);
        deferred.resolve();
    }, 10000);
    return deferred.promise();
})().then(function(){
    $('#result').html('done.');
},
null,
function(){
    $('#result').html($('#result').html() + '.');
});
```

## 结论
`Promises` 是一种全新的方式来处理异步过程。忘记传统的回调，试试 `Promise` 接口。这个朝着正确方向的一大步。

翻译自一篇旧文，我觉得关于 `jQuery.Deferred` 对象的使用总结得很好。在今后很长一段时间内，有些项目依然会依赖 `jQuery`，也同样绕不开 `ajax` 调用，全面的掌握这种方式来处理 `promise` 在实际项目中很有用。

[原文地址](http://www.htmlgoodies.com/beyond/javascript/making-promises-with-jquery-deferred.html)


--- 
## 提问
### deferred 有哪些作用？
- 可以为 `ajax` 调用指定多个回调
- 解决多重回调函数的嵌套问题
- 异步状态可以进行自主的控制

### done 和 then 有哪些异同？
- 相同：两者都能串联使用，为 `resolved` 状态指定多个回调
- 不同：`done` 可以对传入的参数进行过滤，将改变过的值往后传递；而`done` 不会进行过滤，传入回调的参数都是原始的 `resolved` 结果

### deferred.promise() 返回的对象有什么特点?
返回的 `promise` 对象是 `deferred` 对象的子集，只包含可以指定回调函数的方法，而不包含可以改变状态的方法。