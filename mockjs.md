title: mockJs的应用
speaker: 张淑峰
url: https://github.com/ksky521/nodePPT
transition: slide3
files: /js/demo.js,/css/demo.css,/js/zoom.js
theme: moon
usemathjax: yes

[slide]
## 今天要讲的内容
----
* 开发中遇到的问题
* mockJs的简短介绍
* mockJs使用案例
* 总结

[slide]
 在开发用的接口环境恶劣的情况下，前端和后端的关系如同：
[slide]
<img src="/imgs/foreground.png" />
[slide]
<img src="/imgs/81a215ce36d3d5391bec65643d87e950342ab008.jpg" />

[slide]
<p>前端和后端的开发关系</p>
<img src="/imgs/foreBg.png">

[slide]
为了不通过接口并获取到数据的方案：
[slide]
1. 临时修改service层的代码，直接返回静态数据的promise
<pre><code>
export async function queryDashbord(parmas) {

    // return request(`/treeMap/getData`, {
    //     method: 'POST',
    //     headers: {
    //         'Content-Type': 'application/json',
    //     },
    //     body: JSON.stringify(parmas)
    // })

    return Promise.resolve({result: ture, data: { ... }})
}
</code></pre>

[slide]
<pre>这种方案的缺点：
a. 需要更改功能相关代码，提交的时候需要记得自己改回来；
b. 为了测试不同的情况，需要提供不同种静态数据（业务复杂度而定）；
c. 未必能测试到所有情况
</pre>

[slide]
<h3>2.引入`能拦截请求并返回模拟数据`的<strong>容器</strong></h3>

对容器的要求
1. 能够获取请求参数，方便用户根据参数返回相应的业务数据；
2. 能够根据路径产生不同类型不同返回的数据；
3. 能够描述字段间的关系；
4. 足够灵活

[slide]
### mockJs的简短介绍
mockJs顾名思义就是mock（模拟）数据的js工具
[slide]
- mockJs可以产生的数据类型：
<img src="/imgs/mockJsDataType.png" />

[slide]
- mockJs实例代码1:拦截指定的url

<pre>
<code type="script">
import Mock from 'mockJs'
import $ from 'jquery'

Mock.mock("http://127.0.0.1/getUserInfo", {id: '@id', name: '@cname'});

$.get("http://127.0.0.1/getUserInfo", (response) => {
  console.log(response);
})
</code>
</pre>

[slide]
- 实例代码2：描述字段间的关系
  - 实际接口有时候不同的字段之间有所联系，比如说这两个字段：status和statusName，status为1、2、3时，statusName分别为'未开始'、'已开始'、'已结束'
  [slide]
- 代码：
  <pre>
  <code>
  import Mock from 'mockJs'
  import $ from 'jquery'
  Mock.mock("http://127.0.0.1/getUserInfo", function(params) {
    const fieldMap = {
      1: '未开始',
      2: '已开始',
      3: '已结束'
    }
    const mockData = Mock.mock({
      id: '@id', name: '@cname', 
      'status|1':[1, 2, 3], 
    })

    mockData.statusName = fieldMap[mockData.status];
    return mockData;
  });

  $.get("http://127.0.0.1/getUserInfo", (response) => {
    console.log(response);
  })
  </code>
  </pre>
[slide]

- 实例代码3：读取请求参数，根据请求参数模拟出不同的返回值
  - 实际接口也有这样的情况，比如说：告警列表有一个过滤项为负责人名称，通过这个编号查出来的每一行告警的负责人都是该值或以该值为前缀；
[slide]
- 代码：

<pre><code>
  import Mock from 'mockJs'
  import $ from 'jquery'
  Mock.mock(/http:\/\/127.0.0.1\/getAlerts/, function(params) {
    const bodyJSON = JSON.parse("{\"" + decodeURI(params.body).replace("=", "\":\"") + "\"}"); 
    const mockData = Mock.mock({
      "list|1-10": [
        {
          id: '@id',
          name: '@name',
          createDate: '@date',
          ownerName: bodyJSON.ownerName + "@cword(1)"
        }
      ]
    })
    return mockData;
  });

  $.ajax({
    url: "http://127.0.0.1/getAlerts",
    type: "POST",
    data: {
      ownerName: '张'
    },
    success: (response) => {
      console.log(response);
    }
  })
</code></pre>

[slide]
<h1>如何在项目中干净快捷地引入mockJs?</h1>
```
引入原则：
1.只修改一个或少数个主要文件；
2.只在开发模式下生效，不影响生产模式；
```
[slide]
- 一种架构
<br/>
<img src="/imgs/mockConstruct.png" />
[slide]
<pre>
<code>
[index.js]
// 这个js是开始mock的地方，在全局范围的地方调用mockStart
import rules from './rule'
import Mock from 'mockjs'

export function mockStart(api) {
  let ruleMap = {};
  console.log(rules);
  Object.keys(rules).map((key) => {
    const rule = rules[key];
    if (typeof rule.template === 'function') {
      Mock.mock(rule.regex || api + key, rule.template);
    } else {
      const templateKeys = Object.keys(rule).filter(ruleKey => ruleKey.indexOf("template") >= 0);
      const templateKey = templateKeys.length > 0?templateKeys[0]:undefined;
      Mock.mock(rule.regex || api + key, function() {
        const mockData = templateKey?Mock.mock(rule).template:Mock.mock(rule);
        return mockData;
      });
    }
  })
}
</code></pre>

[slide]
<pre><code>
[rule.js]
// rule.js是用来写规则的

import Mock from 'mockjs'

// 规则的结构为
// {
//   url1: {
//     template: mockjs规则对象 || 返回对象格式为{template: 具体的值}的function,
//     regrex?: 正则表达式 // 如存在，则拦截regrex对应值的，若不存在，则拦截url1完全匹配的地址
//   },
//   url2: {...}
// }
export default {
  '/queryList': {
    // 这种格式表示返回回来是数组而非对象
    'template|1-10': [
      {
        id: '@id',
        name: '@name'
      }
    ]
  },
  // 这个格式表示路径不是固定的，采用regrex匹配路径
  '/viewAlert': {
    'regrex': /\/viewAlert\//,
    'template': {}
  }
}

</code></pre>

[slide]
```
通过mockJs我们可以做什么?
1.规范化前后端的开发方式；
2.提高前端工作效率；
3.加强自测能力
```
