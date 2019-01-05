## git操作

### 一. 初次配置
1. 配置署名和邮箱。没有--global则表明只影响当前文件夹
```
git config --global user.name "用户名"
git config --global user.email "邮箱"
```


2. 设置颜色高亮
```
git config --global color.ui true
```

## 配置远程仓库
远程仓库名称 XXX:远程仓库地址
```
git remote add origin XXX
//git remote add 代表新增一个远程仓库，origin：

git push origin master
//将本地master分支跟踪到origin远程分支上
```

## 小技巧
```
git	log	--graph	--pretty=format:'%Cred%h%Creset	-%C(yellow)%d%Creset	%s	%Cgreen(%cr)	%C(bold	blue)<%an>%Creset'	--abbrevcommit	--date=relative

git	config	--global	alias.lg	"log	--graph	--pretty=format:'%Cred%h%Creset	-%C(yellow)%
d%Creset	%s	%Cgreen(%cr)	%C(bold	blue)<%an>%Creset'	--abbrev-commit	--date=relative"

git	config	--global	color.ui	true//着色
git	config	--global	core.quotepath	false	#	设置显示中文文件名
```
参考资料
- [一张图看明白Git的四个区五种状态](https://geektutu.com/post/git-four-areas-five-states.html)
- [团队合作必备的Git操作](https://segmentfault.com/a/1190000015676846)
- [git权威指南](https://git-scm.com/book/zh/v2)