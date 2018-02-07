---
title: JavaScript完美运动框架的进阶之旅
date: 2016-09-08 07:08:50
tags:
    - JavaScript
---

## 运动框架的实现思路 ##
>运动，其实就是在一段时间内改变left、right、width、height、opactiy的值，到达目的地之后停止。

现在按照以下步骤来进行运动框架的封装：
　　１．匀速运动
　　２．缓冲运动
　　３．多物体运动
　　４．任意值变化
　　５．链式运动
　　６．同时运动
<!--more-->
## （一）匀速运动 ##

### 速度动画 ###
#### 运动基础 ####
思考:如何让`div`动起来
    １.设置元素为绝对定位，只有绝对定位后，left,top等值才生效。
    ２.定时器的使用（动态改变值），这里使用setInterval()每隔指定的时间执行代码。
```javascript
/**
 * 运动框架-1-动起来
 * @param {HTMLElement} element 进行运动的节点
 */
var timer = null;
function startMove(element) {
    timer = setInterval(function () {//定时器
        if (element.offsetLeft === iTarget){
        　　clearInterval(timer);
        }else{
        　　element.style.left = element.offsetLeft + 5 + "px";
        }
    }, 30);
}
```
就这样是不是就完成了呢？已经ok了呢？
no。还有一些Bug需要处理。

当我们将鼠标快速的移入或移出目标块时，运动会加快。

```javascript
/**
 * 运动框架-1-解决Bug
 */
var timer = null;
function startMove(element, iTarget) {
    clearInterval(timer);//在开始运动时,关闭已有定时器
    timer = setInterval(function () {
        var iSpeed = 5;//把速度用变量保存
        //把运动和停止隔开(if/else)
        if (element.offsetLeft === iTarget) {//结束运动
            clearInterval(timer);
        } else {
            element.style.left = element.offsetLeft + iSpeed + "px";
        }
    }, 30);
}
```

这样一个最简单的运动框架就完成了，现在再对速度值进行一些设置：

```javascript
//判断距离目标位置，达到自动变化速度正负
var iSpeed = 0;
if (element.offsetLeft < iTarget) {
    iSpeed = 5;
} else {
    iSpeed = -5;
}
```

### 透明动画 ###
　　１．用变量alpha储存当前透明度。
　　２．把上面的element.offsetLeft改成变量alpha。
　　３．运动和停止条件部分进行更改。如下：
```javascript
//透明度兼容实现
if(alpha === iTarget){
　　clearInterval(time);
}else{
　　alpha +=speed;
　　element.style.filter = "alpha(opacity:" + alpha + ")"//兼容IE
　　element.style.opacity = alpha/100;
}
```

## （二）缓冲动画 ##

　　- 速度值与距离值相关。
　　- Bug：当距离变小时，速度回小于1，会被浏览器默认向下取整。
　　　　- 向下取整。Math.floor()
　　　　- 向上取整。Math.ceil()
```javascript
/**
 * 运动框架-2-缓冲动画
 */
function startMove(element, iTarget) {
    clearInterval(timer);
    timer = setInterval(function () {
    //因为速度要动态改变，所以必须放在定时器中
        var iSpeed = (iTarget - element.offsetLeft) / 10; //(目标值-当前值)/缩放系数=速度
        console.log(element.offsetLeft);
        if (element.offsetLeft === iTarget) {//结束运动
            clearInterval(timer);
        } else {
            element.style.left = element.offsetLeft + iSpeed + "px";
        }
    }, 30);
}
```
在浏览器F12中可以看到不断地console一个不为iTarget的整数，这便是运动速度不取整导致的BUG。**在运动函数中，对运动速度取整这一点非常重要**

改写如下：
```javascript
**
 * 运动框架-3-缓冲动画
 */
function startMove(element, iTarget) {
    clearInterval(timer);
    timer = setInterval(function () {
    //因为速度要动态改变，所以必须放在定时器中
        var iSpeed = (iTarget - element.offsetLeft) / 10; //(目标值-当前值)/缩放系数=速度
        iSpeed = iSpeed > 0 ? Math.ceil(iSpeed) : Math.floor(iSpeed); //速度取整
        if (element.offsetLeft === iTarget) {//结束运动
            clearInterval(timer);
        } else {
            element.style.left = element.offsetLeft + iSpeed + "px";
        }
    }, 30);
}
```

##（三）多物体运动 ##
　　- 直接使用element.timer把定时器变成对象上的一个属性。

```javascript
window.onload=function(){
        var list = document.getElementsByTagName("li");
        for(var i=0; i<list.length;i++){
            list[i].timer=null;
            list[i].onmouseover=function(){
                startMove(this,400);
            }
            list[i].onmouseout=function(){
                startMove(this,200);
            }
        }
    }
    //var timer=null;
    //多物体运动所有东西都不能共用
    function startMove(obj, iTarget){
        clearInterval(obj.timer);
        obj.timer = setInterval(function(){
            var speed=(iTarget-obj.clientWidth)/8;
            speed=speed>0?Math.ceil(speed):Math.floor(speed);
            if(obj.clientWidth==iTarget){
                clearInterval(obj.timer);
            }else{
                obj.style.width=obj.clientWidth+speed+"px"; 
            }
        },30)
    }
```

**多物体运动所有东西都不能共用**

##（四）任意值变化 ##
我们来给div加个1px的边框。boder :1px solid #000

然后来试试下面的代码
```javascript
setInterval(function () {
    oDiv.style.width = oDiv.offsetWidth - 1 + "px";
}, 30)
```
明明speed是-1结果宽度却在不断增大。
这是因为offsetWidth的值是width+2*border，所以其实函数每运行一次width都会+1s。

### 第一步：获取实际样式 ###
>使用offsetLeft..等获取样式时, 若设置了边框, padding, 等可以改变元素宽度高度的属性时会出现BUG.
　　 - 利用getcomputedStyle(非IE)或currentStyle(IE)来获取实际样式。
　　 - 将两个函数封装为getStyle以兼容各浏览器。
```javascript
/**
 * 获取实际样式函数
 * @param   {HTMLElement}   element  需要寻找的样式的html节点
 * @param   {String} attr 在对象中寻找的样式属性
 * @returns {String} 获取到的属性
*/
function getStyle(element, attr){
　　//IE
　　if(element.currentStyle){
　　　　return element.currentStyle[attr];
　　}else{//标准
　　　　return getComputedStyle(element, false);
　　}
}
```
### 第二部：改造原函数 ###
　　１．添加参数，attr表示需要改变的属性值。
　　２．更改element.offsetLeft为getStyle(element, attr)。
　　　　* 注意：getStyle(element, attr)不能直接使用，因为它获取到的字符串。
　　３．element.style.left为element.style[attr]。
```javascript
/**
 * 运动框架-4-任意值变化
 * @param {HTMLElement} element 运动对象
 * @param {string}      attr    需要改变的属性。
 * @param {number}      iTarget 目标值
 */
function startMove(element, attr, iTarget) {
    clearInterval(element.timer);
    element.timer = setInterval(function () {
        //因为速度要动态改变，所以必须放在定时器中
    var iCurrent=0;
    iCurrent = parseInt(getStyle(element, attr));//实际样式大小
        var iSpeed = (iTarget - iCurrent) / 10; //(目标值-当前值)/缩放系数=速度
        iSpeed = iSpeed > 0 ? Math.ceil(iSpeed) : Math.floor(iSpeed); //速度取整
        if (iCurrent === iTarget) {//结束运动
            clearInterval(element.timer);
        } else {
            element.style[attr] = iCurrent + iSpeed + "px";
        }
    }, 30);
}
```

### 第三步：透明度兼容处理 ###
　　１.判断`attr`是不是透明度
　　２.对透明度进行特殊处理`parseInt()`会对小数进行向下取整
```javascript
/**
 * 运动框架-5-兼容透明度
 * @param {HTMLElement} element 运动对象
 * @param {string}      attr    需要改变的属性。
 * @param {number}      iTarget 目标值
 */
function startMove(element, attr, iTarget) {
    clearInterval(element.timer);
    element.timer = setInterval(function () {
        //因为速度要动态改变，所以必须放在定时器中
        var iCurrent = 0;
        if (attr === "opacity") { //为透明度时执行。
            iCurrent = Math.round(parseFloat(getStyle(element, attr)) * 100);
        } else { //默认情况
            iCurrent = parseInt(getStyle(element, attr)); //实际样式大小
        }
        var iSpeed = (iTarget - iCurrent) / 10; //(目标值-当前值)/缩放系数=速度
        iSpeed = iSpeed > 0 ? Math.ceil(iSpeed) : Math.floor(iSpeed); //速度取整
        if (iCurrent === iTarget) {//结束运动
            clearInterval(element.timer);
        } else {
            if (attr === "opacity") { //为透明度时，执行
                element.style.filter = "alpha(opacity:" + (iCurrent + iSpeed) + ")"; //IE
                element.style.opacity = (iCurrent + iSpeed) / 100; //标准
            } else { //默认
                element.style[attr] = iCurrent + iSpeed + "px";
            }
        }
    }, 30);
}
```


##（五）链式运动 ##
>链式动画：顾名思义，就是在该次运动停止时，开始下一次运动。

如何实现呢？
　　- 使用回调函数：运动停止时，执行函数
　　- 添加｀func｀形参（回调函数）。
　　- 在iCurrent === iTarget时，判断是否有回调函数存在，有则执行。
```javascript
if (iCurrent === iTarget) {//结束运动
    clearInterval(element.timer);
    if (func) {
        func();//回调函数
    }
}
```

##（六）同时运动 ##
思考：**如何实现同时运动**？
　　１.使用对象传递多个值
　　２.使用`for in`循环，遍历属性，与值。
　　３.定时器问题!(运动提前停止)
## 完美运动框架 ##
```javascript
/**
 * 获取实际样式函数
 * @param   {HTMLElement}   element  需要寻找的样式的html节点
 * @param   {String]} attr 在对象中寻找的样式属性
 * @returns {String} 获取到的属性
 */
function getStyle(element, attr) {
    //IE写法
    if (element.currentStyle) {
        return element.currentStyle[attr];
        //标准
    } else {
        return getComputedStyle(element, false)[attr];
    }
}
/**
 * 完美运动框架
 * @param {HTMLElement} element 运动对象
 * @param {JSON}        json    属性：目标值      
 *   @property {String} attr    属性值
 *   @config   {Number} target  目标值
 * @param {function}    func    可选，回调函数，链式动画。
 */
function startMove(element, json, fn){
    clearInterval(element.timer);
    element.timer = setInterval(function(){
        var flag = true;//假设所有参数都已到达
        for(var attr in json){
            if(attr === "opacity"){
                icur = Math.round(paraseFloat(getStyle(element,attr)));
            }else{
                icur = parseInt(getStyle(element,attr));
            }

            var speed = (json[attr]-icur)/10;
            speed = speed>0?Math.ceil(speed):Math.floor(speed);
            if (icur != json[attr]) {
                flag = false;
                if (attr === "opacity") {
                    element.style.filter = "alpha(opacity:" + (icur + speed) + ")"; //IE
                    element.style.opacity = (icur + speed) / 100;   
                }else{
                    element.style[attr] = icur + speed + "px";
                }
            }
        }
        if (flag) {
            clearInterval(element.timer);
            if (fn) {
                fn();
            }
        }
    },30) 
}
```