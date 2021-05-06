# 一. 问题描述

​	今天准备破解下公司的定位打卡

# 二. 思路

### 1. 分析请求

1）首先用`fiddler`抓包

主要采用了`fiddler`的`bpu`功能

![image-20210320144342992](https://i.loli.net/2021/03/20/q2OVWsrcYtnJoAH.png)

2）模拟修改包数据

​	虽然`bpu`能够拦截修改，但是手动很耗时，最好能够自动拦截到对应的`url`，自动修改`body`中的数据

这里就用到了`fiddler`的自定义规则，搜索`fiddler customize rules`可以发现不少教程，或者`oSession.RequestBody`

![image-20210320144606642](https://i.loli.net/2021/03/20/x8LyotNWek5JHUh.png)

由于我要修改的是请求数据，所以定位到`OnBeforeRequest`方法，修改如下：

```
	if (oSession.HostnameIs("域名") && oSession.uriContains("地址")){
			//替换前
			var filename = "D:/fiddler-token.log";  
			var curDate = new Date();  
			var logContent =  "[" + curDate.toLocaleString() + "] 替换前：" + oSession.GetRequestBodyAsString() + "\r\n";  
			
			//替换经纬度
			var reponseJsonString=oSession.GetRequestBodyAsString();//获取JSON字符串
			var oJson=Fiddler.WebFormats.JSON.JsonDecode(reponseJsonString);//字符串转化为JSON数据，可编辑
			oJson.JSONObject["longitude"] = "118.817353";
			oJson.JSONObject["latitude"] = "31.919729";
			
			//将json对象序列化成字符串，重新设置
			var reJsonDes=Fiddler.WebFormats.JSON.JsonEncode(oJson.JSONObject);
			oSession.utilSetRequestBody(reJsonDes);//设置RequestBody中的JSON数据


			var logContent2 =  logContent+ "\r\n"+ "[" + curDate.toLocaleString() + "] 替换后：" +oSession.GetRequestBodyAsString() + "\r\n";  
			var sw : System.IO.StreamWriter;  
			if (System.IO.File.Exists(filename)){  
				sw = System.IO.File.AppendText(filename);  
				sw.Write(logContent2);  
			}  
			else{  
				sw = System.IO.File.CreateText(filename);  
				sw.Write(logContent2);  
			}  
			sw.Close();  
			sw.Dispose();  
		}
```

以下是常用的方法

```
var reponseJsonString=oSession.GetRequestAsString();//获取JSON字符串
var responseJSON=Fiddler.WebFormats.JSON.JsonDecode(reponseJsonString);//字符串转化为JSON数据，可编辑

var str='{"key":"value"}';//自定义JSON
responseJSON.JSONObject['data']= Fiddler.WebFormats.JSON.JsonDecode(str).JSONObject ;//转换需要
var myResponseJSON= Fiddler.WebFormats.JSON.JsonEncode(responseJSON.JSONObject);//转换需要

oSession.utilSetResponseBody(myResponseJSON);//设置ResponseBody中的JSON数据

FiddlerObject.alert(oJson);//弹窗显示
```





参考资料

- [ Fiddler Json](https://www.jianshu.com/p/663f015bc42b)
- [ fiddler 脚本](https://blog.csdn.net/qq_37299249/article/details/70558861)
- [fiddler 记录日志](http://www.site-digger.com/html/articles/20170810/137.html)