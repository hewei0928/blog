---
title: JavaScript类型识别
date: 2016-09-07 17:21:18
tags: 
	- JavaScript
---

## 类型系统 ##

> javascript 类型系统可以分为标准类型和对象类型，进一步标准类型又可以分为原始类型和引用类型，而对象类型又可以分为内置对象类型、普通对象类型、自定义对象类型。

![img](http://i2.buimg.com/567571/57e3b460b69b40fc.jpg)

## 类型判断 ##

 - `typeof`
 - `Object.prototype.toString`
 - `constructor`
 - `instanceof`

`typeof`
    1.可以识别标准类型（`null`除外）
    2.不可识别具体的对象类型（`Function`除外）
<!--more-->
例：
```javascript
//1.可以识别基本包装类型（'null'除外）
typeof(1);//'number'
typeof('a');//'string'
typeof(undenfined);//'undefined'
typeof(true);//'boolean'
typeof(null);//'object'

//不可识别具体对象类型
typeof([]);//'object'
typeof({});//'object'
typeof(function(){})//'function'
```

`instanceof`
　　1.列表项能够判别内置对象类型
　　2.不能判别原始类型
　　3.能够判别自定义类型

例：
```javascript
//1.列表项能够判别内置对象类型
[] instanceof Array;//true
/\d/ instanceof RegExp;//true

//2.不能判别原始类型
1 instanceof Number;//false
'a' instanceof String;//false

//3.能够判别自定义类型
function Point(x, y) {
	this.x = x;
	this.y = y;
}
var c = new Point(2,3);
c instanceof Point;//true
```
`Object.prototype.toString.call()`方法
　　１.可以识别标准类型,及内置对象类型
　　２.不能识别自定义类型
　　
例：
```javascript
//1. 可以识别标准类型,及内置对象类型
Object.prototype.toString.call(21);//"[object Number]"
Object.prototype.toString.call([]);//"[object Array]"
Object.prototype.toString.call(/[A-Z]/);//"[object RegExp]"

//2. 不能识别自定义类型
function Point(x, y) {
	this.x = x;
	this.y = y;
}
var c = new Point(2,3);//c instanceof Point;//true
Object.prototype.toString.call(c);//"[object Object]"
```

 - 为了方便使用，封装函数如下：
```javascript
function typeProto(obj){
　　return Object.prototype.toString.call(obj).slice(8,-1);
}

typeProto('a');//"String"
typeProto({});//"Object"
typeProto([]);//"Array"
```

`constructor`
>constructor指向构造这个对象的构造函数本身。
　　１．可识别原始类型
　　２．可识别内置对象类型
　　３．可识别自定义类型
　　
例：
```javascript
//1. 可识别原始类型
"guo".constructor === String;//true
(1).constructor === Number;//true
true.constructor === Boolean;//true
({}).constructor === Object;//true
//2. 可识别内置对象类型
new Date().constructor === Date;//true
[].constructor === Array;//true
//3. 可识别自定义类型
function People(x, y) {
	this.x = x;
	this.y = y;
}
var c = new People(2,3);
c.constructor===People;//true
```
　　

 - 为了方便使用，封装函数如下：
```javascript
function getConstructorName(obj){
　　 return obj &&　obj.constructor　&&　obj.constructor.toString().match(/function\s*([^(]*)/)[1];
}
```
封装原理：
　　如果obj存在，obj.constructor存在则执行obj.constructor.toString().match(/function\s*([^(]*)/)[1]
　　
## 类型判断对比表 ##
　　
 　　- 其中红色的单元格表示判断方式不支持的类型。
 ![img](http://i2.buimg.com/567571/995fde5248ac0e07.png)

## 综合练习 ##

```javascript
var result=function(){
    //以下为多组测试数据
            var cases=[{
                    arr1:[1,true,null],
                    arr2:[null,false,100],
                    expect:true
                },{
                    arr1:[function(){},100],
                    arr2:[100,{}],
                    expect:false
                },{
                    arr1:[null,999],
                    arr2:[{},444],
                    expect:false
                },{
                    arr1:[window,1,true,new Date(),"hahaha",(function(){}),undefined],
                    arr2:[undefined,(function(){}),"okokok",new Date(),false,2,window],
                    expect:true
                },{
                    arr1:[new Date()],
                    arr2:[{}],
                    expect:false
                },{
                    arr1:[window],
                    arr2:[{}],
                    expect:false
                },{
                    arr1:[undefined,1],
                    arr2:[null,2],
                    expect:false
                },{
                    arr1:[new Object,new Object,new Object],
                    arr2:[{},{},null],
                    expect:false
                },{
                    arr1:null,
                    arr2:null,
                    expect:false
                },{
                    arr1:[],
                    arr2:undefined,
                    expect:false
                },{
                    arr1:"abc",
                    arr2:"cba",
                    expect:false
                }];
//使用for循环, 通过arraysSimilar函数验证以上数据是否相似，如相似显示“通过”,否则"不通过"。
//相似条件：
//1.传入的数据是数组， 数组中的成员类型相同，顺序可以不同。例如[1, true] 与 [false, 2]是相似的。
//2. 数组的长度一致。
//3. 类型的判断范围，需要区分:String, Boolean, Number, undefined, null, 函数，日期, window.
            for(var i=0;i<cases.length;i++){
                if(arraysSimilar(cases[i].arr1,cases[i].arr2)!==cases[i].expect) {
                    document.write("不通过！case"+(i+1)+"不正确！arr1="+JSON.stringify(cases[i].arr1)+", arr2="+JSON.stringify(cases[i].arr2)+" 的判断结果不是"+cases[i].expect);
                    return false;
                }                
            }
            return true;
        }();
    document.write("判定结果:"+(result?"通过":"不通过"));
    
    
/*
* param1 Array 
* param2 Array
* return true or false
*/
function arraysSimilar(arr1, arr2){
            if(!Array.isArray(arr1) || !Array.isArray(arr2) ||arr1.length!=arr2.length){return false;}
            var arr3=[];
            var arr4=[];
            var x;
            for(var i in arr1){
                arr3.push(type(arr1[i]));
                arr4.push(type(arr2[i]));
            }
            if(arr3.sort().join()==arr4.sort().join()){
                return true;
            }else{
                return false;
            }
        }
        
/**
 * 类型判断方法
 * param item 
 * return type(string,function,boolean,number,undefined,null,window,Date,Array,object)
 */
 function type(a){
 	  var b 
      a = Object.prototype.toString.apply(a); //hack ie678
      return b = a.slice(8,-1);
}
```

　　


 
