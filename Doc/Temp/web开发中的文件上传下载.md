# 文件下载

昨天和同时联调时发现一个问题下载Excel问题，后端下载代码如下：
```java
    response.setContentType("application/vnd.ms-excel");
    response.setHeader("Content-disposition", "attachment;filename=" + reportName);
    out.write(data)
```

后端通过post测试请求都正常返回，
注意post工具不是点击`Send`按钮，而是`Send And Download`按钮，这样postman 才可以正常弹出下载框

但是前端在测试时通过chrome工具可以监听到有响应，就是浏览器没有弹出下载框

解决方法：**改为get请求**

至于原理就不清楚了，估计浏览器是不是有没有规则限定。


# 文件上传

既然提到下载了，那么也顺便重温下上传吧。<br>
很多年前看到李刚的 struts2权威指南，记得有讲解这块原理<br>
所以找了以下，方法第6章-文件下载原理讲的不错。

总结一下：

* 首先了解form表的enctype属性
    1. application/x-www-form-urlencoded：这是默认值，会将表单中元素的value值处理成url编码方式
    2. multipart/form-data：会将表单中的数据以二进制流格式处理，
    3. text/plain：主要用于邮件发送，不常用


## 1. 示例1

当`enctype属性值为sapplication/x-www-form-urlencoded`时

```html
<form enctype='application/x-www-form-urlencoded'>
上传文件：<input name='content' type='file' />
上传人:<input name='user' type='text' />
<input type='submit' name='commit' value='提交' />
</form>
```

后端打印代码如下：
```java
// 去掉注释打印出来正常的信息
// request.setCharacterEncoding("GBK")

//1.直接打印二进制流
InputStream is =request.getInputStream();
BufferedReader br = new BufferedReader(new InputStreamReader(is));
String buffer = "";
while((buffer=br.readLine())!=null)
{
    out.print(buffer+"<br>");
}

//2.采用servlet的方法获取，两种方法是相同的
//request.getParameter("content");
//request.getParameter("user");
```
结果如下(格式如下，数据是编造的)：
```
content=E3%5C%E3%5C%E3%5C%E3%5C%E3%5C%E3%5C%E3%5C%E3%5C%E3%5C%
&user=E3%5C%E3%5C%&commitE3%5C%E3%5C%
```


## 2. 示例2
当`enctype属性值为multipart/form-data`时
打印出来的结果如下：
```html
------------------------------7dad1231dasd123asewq123123asda3213
Content-Disposition:form-data name='content'
filename='C:/1.txt'
Content-Type:text/plain

我只是一个测试文本
------------------------------7dad1231dasd123asewq123123asda3213
Content-Disposition:form-data name='user'

张三
------------------------------7dad1231dasd123asewq123123asda3213
Content-Disposition:form-data name='commit'

提交
```

`7dad1231dasd123asewq123123asda3213`是一个唯一标识，由后端应用服务随机产生


**上传文件的后端解析格式如下**：

------------------------------唯一标识
Content-Disposition:XXXXXX<br>
filename=XX<br>
回车换行
回车换行
文件内容YYYYYYYYYYYYYYYYYYYYYYYYYYY



所以后端可以根据以上格式自己解析，这也是一般常用第三方上传库的原理。

