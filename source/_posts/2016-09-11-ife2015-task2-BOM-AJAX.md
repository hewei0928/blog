---
title: ife2015-task2 - BOM+AJAX
date: 2016-09-11 12:29:58
tags:
    - JavaScript
    - ife
---

## BOM ##

### 判断是否为IE浏览器 ###
```javascript
// 判断是否为IE浏览器，返回-1或者版本号
function isIE(){
	var user = navigator.userAgent;
	var ieAgent = user.match(/msie (\d+.\d+)/i);
	if (ieAgent) {
		return ieAgent[1];
	}else{
		if (user.match(/Trident\/7.0/i)) {
			ieAgent = user.match(/rv:(\d+.\d+)/i);
			return ieAgent[1];
		}else{
			return -1;
		}
	}
}
```
<!--more-->
### Cookie相关 ###

#### 设置Cookie ####
```javascript
/**
 * 设置cookie
 * @param {String} cookieName  设置cookie名
 * @param {String} cookieValue 对应的cookie名
 * @param {Number} expiredays  过期的时间(多少天后)
 */
function setCookie(cookieName, cookieValue, expiredays) {
    var oDate = new Date();
    oDate.setDate(oDate.getDate() + expiredays);
    document.cookie = cookieName + '=' + cookieValue + ";expiredays =" + oDate.toGMTString();
}

#### 获取Cookie ####
 /**
 * 获取cookie
 * @param   {String} cookieName 待寻找的cookie名
 * @returns {String} 返回寻找到的cookie值,无时为空
 */
function getCookie(cookieName) {
    var arr = document.cookie.split(';');
    for(var i = 0; i < arr.length; i++){
        var arr2 = arr[i].split('=');
        if(arr2[0] == cookieName){
            return arr2[1];
        }
    }
    return "";
}
```

#### 删除Cookie ####
```javascript
/**
 * 删除cookie
 * @param {String} cookieName 待删除的cookie名
 */
function removeCookie(cookieName) {
    setCookie(cookieName, "1", -1)
}

```

## Ajax ##

### 尝试自己封装一个Ajax方法 ###

```javascript
function ajax(url, options) {
    var request = null;
    if (window.ActiveXObject) {
    	request = new ActiveXObject('Microsoft.XMLHTTP');
    }else if(window.XMLHttpRequest){
    	request = new XMLHttpRequest();
    }
    if (!request) {
    	alert('创建XMLHttp对象异常');
    	return false;
    }else{
    	var param = '';
    	if (option.data) {
    		for(var i in option.data){
    			if (option.data.hasOwnProperty(i)) {
    				param = i + "=" + option.data[i] + "&";
    			}
    		}
    		param = param.replace(/&$/,'');
    	}
    	var type = options.type? options.type.toUpperCase():"GET"
    	if (option.type == 'GET') {
    		url = url+"?"+param;
    		request.open("GET", url, true);
    		request.send();
    	}else{
    		oAjax.open("POST", url, true);
        	oAjax.setRequestHeader("Content-type", "application/x-www-form-urlencoded");
        	oAjax.send(param);
    	}

    	request.onreadystatechange = function(){
    		if (request.readyState == 4) {
    			if (request.status == 200) {
    				options.onsuccess(request.responseText, request);
    			}else{
    				if (options.onfail) {
    					options.onfail(request);
    				}
    			}
    		}
    	}
    }
    return request;
}


// 使用示例：
ajax(
    'http://localhost:8080/server/ajaxtest', 
    {
        data: {
            name: 'simon',
            password: '123456'
        },
        onsuccess: function (responseText, xhr) {
            console.log(responseText);
        }
    }
);
//options是一个对象，里面可以包括的参数为：
//type: post或者get，可以有一个默认值
//data: 发送的数据，为一个键值对象或者为一个用&连接的赋值字符串
//onsuccess: 成功时的调用函数
//onfail: 失败时的调用函数
```

