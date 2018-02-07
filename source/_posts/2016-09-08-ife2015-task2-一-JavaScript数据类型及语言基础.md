layout: w
title: ife2015-task2(1)-JavaScript数据类型及语言基础
date: 2016-09-08 11:27:25
tags:
    - JavaScript
    - ife
---

## 1.第一个页面交互 ##
　　- 　这里最需要学习的老师的代码中，每一部分功能都由函数控制，没有创建一个全部变量。

## 2. JavaScript数据类型及语言基础 ##
  　- 详情见[另一篇博客](https://hewei0928.github.io/2016/09/07/JavaScript%E7%B1%BB%E5%9E%8B%E8%AF%86%E5%88%AB/)
  
### 2.1 深度克隆 ###
**浅度克隆**:原始类型按值传递， 对象类型仍为引用传递
**深度克隆**：所有元素或属性均完全复制，与原对象完全脱离，也就是说所有对于新对象的修改都不会反映到原对象中。
<!--more-->
#### 浅克隆的表现 ####
1.原始类型
```javascript
//数值克隆的表现
var a="1";
var b=a;
b="2";
console.log(a);// "1"
console.log(b);// "2"

//字符串克隆的表现
var c="1";
var d=c;
d="2";
console.log(c);// "1"
console.log(d);// "2"

//布尔值克隆的表现
var x=true;
var y=x;
y=false;
console.log(x);// true
console.log(y);// false
```
以上代码可以看出：原始类型即使我们采用普通的克隆方式仍能得到正确的结果，原因就是原始类型存储的是对象的实际数据。

2.对象类型

函数也是对象，但是函数的克隆通过浅克隆即可实现
```javascript
var m=function(){
　　alert(1);
};
var n=m;
n=function(){
　　alert(2);
};
 
console.log(m());//1
console.log(n());//2
```
网上解释说是函数的克隆会在内存单独开辟一块空间，互不影响。

下面就来说说普通对象类型，设置oPerson如下
```javascript
var oPerson={
    oName:"rookiebob",
    oAge:"18",
    oAddress:{
        province:"beijing"
    },    
    ofavorite:[
        "swimming",
        {reading:"history book"}
    ],
    skill:function(){
        console.log("bob is coding");
    }
};
function clone(obj){
    var result={};
    for(key in obj){
        result[key]=obj[key];
    }
    return result;
}
var oNew=clone(oPerson);
console.log(oPerson.oAddress.province);//beijing
oNew.oAddress.province="shanghai";
console.log(oPerson.oAddress.province);//shanghai
```
通过上面的代码，可以看到，经过对象克隆以后，修改oNew的地址，发现原对象oPerson也被修改了。这说明对象的克隆不够彻底，那也就是说深度克隆失败！

#### 深度克隆的实现 ####
>为了保证对象的所有属性都被复制到，我们必须知道如果for循环以后，得到的元素仍是Object或者Array，那么需要再次循环，直到元素是原始类型或者函数为止。为了得到元素的类型，我们定义一个通用函数，用来返回传入对象的类型。
```javascript
function isClass(o){
　　return Object.prototype.toString.call(o).slice(8,-1);
}

function deepclone(obj){
　　var result, oClass = isClass(obj)
　　if(oClass === "Object"){
　　　　result = {};
　　}else if(oClass === "Array"){
　　　　result = [];
　　}else{
　　　　return obj;
　　}
　　
　　for(var key in obj){
　　　　var copy = obj[key];
　　　　if(isClass(copy) === "Object"||"Array"){
　　　　　　result[key] = deepclone(copy);
　　　　}else{
　　　　　　result[key] = copy;
　　　　}
　　}
　　
　　return result;
}
```

### 2.2 数组、字符串、数字等相关方法 ###

#### 数组去重操作 ####
```javascript
// 对数组进行去重操作，只考虑数组中元素为数字或字符串，返回一个去重后的数组
function uniqArray(arr) {
    // your implement
    var result = [];
    for(var i =0; i < arr.length; i++){
    　　if(result.indexOf(arr[i]) == -1){
    　　　　result.push(arr[i]);
    　　}
    }
    return result;
}
// 使用示例
var a = [1, 3, 5, 7, 5, 3];
var b = uniqArray(a);
console.log(b); // [1, 3, 5, 7]
```

#### 实现`trim`函数，去除字符串首尾空白 ####
　　- 实现一个简单的`trim`函数，用于去除一个字符串，头部和尾部的空白字符(包括全角和半角)
```javascript
function trim(str){
　　var result = "";
　　result = str.replace(/^(\s|\u00A0)+|(\s|\u00A0)+$/g, '');
　　return result;
}
```

#### 实现`each`函数，实现一个遍历数组 ####
　　- 针对数组中每一个元素执行fn函数，并将数组索引和元素作为参数传递
```javascript
function each(arr,fn){
　　 for (var i = 0, l = arr.length; i < l; i++) {//遍历传参
        fn(arr[i], i);
    }
}
```

#### 实现`getObjectLength`函数，获取一个对象里面第一层元素的数量，返回一个整数 ####
```javascript
function getObjectLength(obj){
　　var i = 0;
　　for(var key in obj){//for in循环会遍历来自原型的属性和方法
　　　　if (obj.hasOwnProperty(i)) {//
            count++;
        }
　　}
　　return i;
}
```

#### 正则表达式 ####
```javascript
// 判断是否为邮箱地址
function isEmail(emailStr) {
    var reg = /^(\w+)@(\w+)(\.\w+)$/;
    return reg.test(emailStr);
}

// 判断是否为手机号
function isMobilePhone(phone) {
    var reg = /^1[3578][0-9]{9}/;
    return reg.test(phone);
}
```






