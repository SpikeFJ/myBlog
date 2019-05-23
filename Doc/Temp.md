* Tomcat--->Mapper
* 建立一个网站
    * 数据源-->爬虫--> [python](https://www.cnblogs.com/happymeng/)
                <br>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   &nbsp;&nbsp;&nbsp;&nbsp;--->
                linux，apt-get
    * 部署环境--->linux--->宝塔
    * NetCore
        * Spike类库-->微信


        sudo ln -s /usr/src/Python-3.7.0 /usr/bin/python


        cd /usr/bin
        sudo rm -rf python
        sudo ln -s  /usr/src/Python-3.7.0/python /usr/bin/python


java算法--后缀表达式
名称解析：如A+B*C；其中A、B、C称之为操作数，+-`*`/称之为操作符
1、操作数：直接输出
2、(：直接入栈
3、)：弹出栈中所有操作符直至遇到(退出
4、操作符(opThis)：
    1)栈为空直接入栈
    2)栈不为空，弹出栈元素(opTop)，比较优先级，如果
        1)opTop < opThis,即优先级大于栈中元素的优先级，不管
        2)opTop >= opThis，输出opTop，循环直至优先级不满足条件或(
        3)推入opThis
5、没有更多项时，弹出所有