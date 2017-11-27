title:前后端权限管理方案
speaker: 张淑峰
url: https://github.com/ksky521/nodePPT
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

[slide]
## 目前的权限设计方案
<img src="/imgs/permission_old.png" />

[slide]
### 设计方案1

<img src="/imgs/permission_a.png" />

[slide]

### 设计方案2

<img src="/imgs/permission_b.png" />

[slide]

> 问题：两种方式都是通过对角色进行判断来进行权限的管理，但r13开始租户的角色不是定死的两种，而是可以自由增加的，那么这种方式就有必要改进了

[slide]

## 什么是功能？如何描述一个功能？

<img src="/imgs/whatisfunction.png">

> 脱离前端的功能描述：指定接口集合的顺序执行

[slide]
## 什么是功能权限？

> 功能权限个人理解：正常流程下保证功能顺利执行或者一定不执行的代码

> 如何保证功能顺利执行：有权限访问执行功能所需的所有接口；

> 如何保证功能一定不执行：!(如何保证功能顺利执行)

[slide]
# 故功能权限可以用接口的集合表示

> 例子：[

>    "api/v2/dataService/wos",

>    "api/v2/dataService/assignWO"

>  ],

[slide]
### 疑问1：也许有两个功能的接口权限是包含的关系
> A功能：['a']

> B功能：['a', 'b']

> 拥有B功能必定先拥有A功能

[slide]
### 设计方案3

<img src="/imgs/permission_c.png"/>

[slide]
<img src="/imgs/permission_split.png"/>

[slide]
### 功能权限的设计步骤
[slide]

### 1.完成权限文档


<img src="/imgs/permission_wiki.png"/>

> 参考文档：http://kb.uyunsoft.cn/kb/pages/viewpage.action?pageId=35818149

[slide]

### 2.后端为【权限文档涉及的接口】添加“权限控制”

<img src="/imgs/permission_bg.png"/>



[slide]

### 3.前端编写权限组件控制需要权限控制的元素和角色功能点关联页面

<img src="/imgs/permission_fg.png" />

[slide]
### 思考一下：每个产品自己管理权限会有什么问题？

[slide]
### 问题1

<img src="/imgs/permission_problem2.png">

[slide]

## 答案是：无法使用

## 如何解决这个问题？

[slide]

<img src="/imgs/permission_advice.png">

[slide]

> 这种架构逼迫我们必须在提供给其他产品页面时，为这个页面单独创建一个角色，所有想使用这张页面的用户都必须勾上这个角色。

[slide]
# 更理想的做法

> 分享给其他产品的页面发送请求时使用openApi，不做功能权限的设置。

[slide]
# 更更理想的做法

> 不使用iframe的方式嵌套页面，使用原生组件编写页面，并将原生组件分享给其他产品。调用的地方自行通过自己的接口去获取数据，并做功能权限设置。

[slide]
Thank you for listening