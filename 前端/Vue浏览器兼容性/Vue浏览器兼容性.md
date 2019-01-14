# Vue浏览器兼容性

## 360浏览器内核设置

目前，360浏览器可以使用meta进行设置，强制使用指定内核打开页面，方法如下：

```html
<meta name="renderer" content="webkit|ie-comp|ie-stand">
//content的取值为webkit,ie-comp,ie-stand之一，区分大小写，分别代表用webkit内核，IE兼容内核，IE标准内核。 
//若页面需默认用极速核，增加标签：
<meta name="renderer" content="webkit"> 
//若页面需默认用ie兼容内核，增加标签：
<meta name="renderer" content="ie-comp"> 
// 若页面需默认用ie标准内核，增加标签：
<meta name="renderer" content="ie-stand">
```



注意：如果在360浏览器手动设置某一网页的浏览器模式，则代码不会生效，除非清除360浏览器的缓存内容。



### 各渲染内核的技术细节

| 内核            | Webkit    | IE兼容 | IE标准                        |
| --------------- | --------- | ------ | ----------------------------- |
| 内核版本        | Chrome 45 | IE6/7  | IE9/IE10/IE11(取决于用户的IE) |
| HTML5支持       | YES       | NO     | YES                           |
| ActiveX控件支持 | NO        | YES    | YES                           |



此做法类似于IE中指定所需内核的做法

```html
<meta http-equiv="X-UA-Compatible" content="IE=7,IE=9" />
```

content里可以写1个，也可以写多个，用英文逗号隔开

文档模式（document mode）是IE8引入的一个新概念。页面的文档模式决定了你可以使用哪个级别的CSS，可以使用JavaScript的哪些API，以及如何对待文档类型（doctype）。

“X-UA-Compatible”的值有两种方式：Emulate+IE版本号，单纯版本号。
Edge：始终以最新的文档模式来渲染页面。忽略文档类型声明。对于IE8，始终以IE8标准模式渲染页面。IE9亦如此。
EmulateIE9：如果声明了文档类型，则以IE9标准模式渲染页面，否则将文档模式设置为IE5。
EmulateIE8：如果声明了文档类型，则以IE8标准模式渲染页面，否则将文档模式设置为IE5。
EmulateIE7：如果声明了文档类型，则以IE7标准模式渲染页面，否则将文档模式设置为IE5。
9：强制以IE9标准模式渲染页面，忽略文档类型声明。
8：强制以IE8标准模式渲染页面，忽略文档类型声明。
7：强制以IE7标准模式渲染页面，忽略文档类型声明。
5：强制以IE5标准模式渲染页面，忽略文档类型声明。





## IE浏览器支持

1、安装`babel-polyfill`包；

```
npm install babel-polyfill --save-dev
```

安装完之后，在package.json文件中显示：

2、在`main.js`文件中引入`babel-polyfill`；

![这里写图片描述](https://img-blog.csdn.net/20180817110001651?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1h1TTIyMjIyMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

3、在`webpack.base.config.js`中将`entry`中的`app: './src/main.js'`配置改为：`app: ['babel-polyfill', './src/main.js']`；

![这里写图片描述](https://img-blog.csdn.net/2018081711023877?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1h1TTIyMjIyMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

这样配置之后，重新打包上传，就可以在IE浏览器下正常显示了。