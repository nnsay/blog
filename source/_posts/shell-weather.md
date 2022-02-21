---
title: 命令行查看天气
date: 2019-10-20 22:03:23
tags: 
- 工具
- Shell
excerpt: 在终端中查询天气预报
---

### 1. 说明介绍
程序开发人员最长用的工具就是命令行的终端了, 如果可以快捷的从命令行获取天气信息则是一件比较极客和有效率操作了. 现在介绍一个极为简单的方式实现命令行获取天气信息, 并且有以下几个特点:
- 无需安装依赖
- 自定义程度较高
- 配置简单

### 2. 安装配置
#### 2.1 添加shell方法
根据自己的需要在\~/.zshrc或者\~/.bashrc中添加如下方法:
```
fweather()
{
  curl -H "Accept-Language: zh" "http://wttr.in/${1-beijing}?m&${2-3}"
}
```
<!-- more -->
没错就是这么简单即完成安装配置, 其中方法名称(fweather)和默认城市(beijing)可以自定义.
#### 2.2 使用演示
如果已经打开了终端, 请执行如下语句, 重新加载shell配置:
```
exec $SHELL
```

##### 2.2.1 最简使用
直接在命令行执行函数, 默认查看北京近三天的天气:
```
fweather
```
结果如下: 
![](https://oscimg.oschina.net/oscnet/155311b956ee7b54882c4090245f61df94a.jpg)

##### 2.2.2 查询任意城市
执行函数空格一下输入要查询的城市名称, 比如合肥
```
fweather hefei
```

##### 2.2.3 查询任意城市今日天气
执行函数空格一下输入要查询的城市名称, 比如合肥
```
fweather hefei 0
```

##### 2.2.4 常见支持设置的查询
- 语言: zh
- 预报天数: 3
- 城市: beijing 
- 具体地区: wangjing+beijing
- 经纬度: -78.46,106.79
- 域名: @stackoverflow.com
还有很多其他高级的使用比如生成图片, 请查看第三节使用说明了解原理后查看相关资料自定义使用.

### 3. [使用说明](https://github.com/chubin/wttr.in)
shell中的方法本质上调用了http://wttr.in获取数据, 并把结果显示出来, 这个函数有两个参数, 第一指定城市, 第二表示预报几天. wttr高级的用法请重点查看 [**wttr github**](https://github.com/chubin/wttr.in), 基本的wttr使用请参考一下几个资源:
- [wttr 帮助](http://wttr.in/:help)
- [wttr 多语言](http://wttr.in/:translation)
- [wttr 原生bash方法](http://wttr.in/:bash.function)

### 4. 参考资料
- [wttr github](https://github.com/chubin/wttr.in)
- [wttr 网站](http://wttr.in)

### 5. 写在最后
- 感谢wttr团队做出这么好玩好用的程序, 万分感谢
- wttr 可以私有部署, 参考其 [github安装的部分](https://github.com/chubin/wttr.in#installation)
