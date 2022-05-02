# lowcode 低代码前端框架 amis调研

# 背景

### 什么是低代码开发？

> 所谓低代码开发，即无需编码或只需少量代码就可以快速生成应用程序。也就是说，企业的应用开发通过“拖拉拽”的方式即可完成。

### 低代码平台有哪些？

我知道的知名的比如有：

*   阿里的宜搭

*   腾讯的微搭

*   百度的爱速搭

当然对于企业版都是收费的，阿里的宜搭依托于钉钉生态发展的挺好，据消息称字节也正在内部灰度自己的低代码平台，依托于飞书也应该会有比较广泛的应用。

### 我对低代码怎么看？

最近低代码的概念很火，对于业内的人来说，可能感觉就像：技术还是那个技术，新瓶装旧酒，又换了个包装再卖一次一样。

一直以来我都认为低代码是个伪命题，因为它只能针对非定制化或者说非常标准化的功能才有意义，才能够发挥它的价值，但真正的企业内部的需求多数又是定制化非常高的业务，否则每个公司就不用自己招人建团队了，都是标准化的，全部包出去或者随便找几个人做就好了，何必花大钱在技术投入上呢。

直到现在我也是这样认为，可能是因为 lowcode 的概念被很多厂商和公司二次包装的不像样子，有点儿偏了，忽悠不懂行的，宣传的多了，懂行的有些反感，像我就是这样，所以忽视了它本身中立的价值立场。具体来说就是，事物的存在是有它的合理性，那么站在这个角度冷静地思考下，低代码结合我们真实的应用场景到底有没有价值呢？具体来讲是对于一个已有技术团队的公司有什么价值。

在企业内部所有的业务系统自然不在低代码的应用范围，而如果你的团队小，假设是个创业公司，没有那么多人的情况下，像 CRM、OA 这种需求，在像飞书、钉钉这种 IM 中有各种 ISV 提供各种应用，一般情况下也能轻松解决。

但随着团队规模越来越大，岗位分工越来越垂直和精细，就会有越来越多企业内不同领域的需求出现，需要系统来解决，比如：

*   运维团队需要开发自己的运维系统

*   DBA 团队需要开发 SQL 上线审核系统

*   业务团队需要自己的运营系统，甚至会分拆成不同的子系统

*   IT 需要系统维护公司的设备信息

*   ......

而以上所有这些需要的系统需求很大程度上是不能被现有的应用功能（飞书、钉钉）完全满足的，所以需要开发实现，但它们是有相同点的：

*   这些系统多数都是类似 MIS 的信息管理系统

*   功能上相似化程度高

*   也都有一定的定制化功能场景

*   实现上都需要前、后端开发，人员成本高

总结来说就是：**在同质化功能基础上又有部分定制化需求的系统**。

又因为是在企业内部，所以又有两个隐含的需求：

*   成本要低

*   要安全（私有化部署）

现在市面上的 lowcode/nocode 平台提供云端应用和私有化部署，但无论如何是收费的。对于企业来说，**最好是有一个免费又能够部署在自己服务器上（安全）且用起来比较灵活能够满足绝大多数需求的一个平台。**

有这样的东西吗？

没有

那么我们来拆解一下这个需求：

1 免费

花钱能解决的问题都不叫做问题，然而问题是老板不愿意花钱，另外，就算我们愿意掏钱也不一定能解决应用灵活开发的问题，使用人家的东西就要按照人家的逻辑玩儿，你不能即用着 word, 还要求它有个 wps 的新功能。所以免费是个刚需，当然我说的免费只是软件成本，不是说这事儿完全不需要花一分钱成本，私有化部署也要占用服务器资源的。

2 安全

跟第一点有联系，既然不花钱就肯定部署在自己这儿了，那当然安全了

3 灵活开发应用

我们再细拆解一下，现在的应用开发一般都是前后端分离的。

先说**前端**，作为后台的系统，至少需要前端页面来展示，那么就需要有前端开发来做，或者后端自己做。我们都知道团队的前端资源一般都很紧张，业务系统都开发不完，哪有时间帮兄弟团队搞东搞西的，很多情况下都是各团队自已 solo 全栈，虽然都是程序员，但毕竟术业有专攻，那开发出来的页面就五花八门了，且不说好不好看，这个事儿对于开发同学的成本就很高，而且各做各的，有很多重复开发。

再说**后端**，以 java 技术栈为例，后端这边已经有很多的框架和工具可以帮助我们快速的建立一个应用，比如 springboot, 也可以快速的完成 CRUD，比如 springboot+mybatisplus。对于一个熟手来说，写几个 CRUD 接口还是比较快的。另外，要说 lowcode, 一些自动生成代码的工具也可以帮我们在一定程序上实现后端 lowcode, 常见如各种 code generator

还有**契约**，前后端联调 API 最好有个工具，无论是 swagger, 还是 yapi 这种工具能够提高效率，最好是用 yapi 这种能够 mock 数据的，那么就可以在契约+mock 出的数据的基础上实现并行开发。

通过上面的分析，我得出的结论是：

*   后端可以最大限度地灵活开发自定义功能和逻辑部分

*   后端对于通用功能的开发成本也不高

*   前端开发成本高，且复用率低

*   前端学习成本不低，大家的水平参差不齐

*   前端维护成本高，新技术和版本更新较快

看来问题主要集中在前端，如果有个工具能够拥有以下特点就好了

*   不需要什么学习成本，最好都不用懂前端框架

*   能够通过 UI 进行简单配置就可以组合出各种功能

*   对后端的接口调用简单配置就能实现功能

*   不用管各依赖的升级更新，维护成本低

那么有吗？ 有！

## amis

### amis 是什么？

> amis 是一个百度开源的低代码前端框架，它使用 JSON 配置来生成页面，可以减少页面开发工作量，极大提升效率。

有时候其实只想做个普通的增删改查界面，用于信息管理，类似下面这种

![](https://tva1.sinaimg.cn/large/008i3skNly1gu99x2hd7gj615f0u0act02.jpg)

但仔细观察会发现它有大量细节功能
比如：

*   可以对数据做筛选

*   有按钮可以刷新数据

*   编辑单行数据

*   批量修改和删除

*   查询某列

*   按某列排序

*   隐藏某列

*   开启整页内容拖拽排序

*   表格有分页（页数还能同步到地址栏，不过这个例子中关了）

*   有数据汇总

*   支持导出 Excel

*   「渲染引擎」那列的表头有提示文字

*   鼠标移动到「平台」那列的内容时还有放大镜符号，可以展开查看更多

全部实现这些需要大量的代码。

amis 的初衷是：**对于大部分常用页面，应该使用最简单的方法来实现，甚至不需要学习前端框架和工具。**

### amis 的亮点

*   提供完整的界面解决方案：其它 UI 框架必须使用 JavaScript 来组装业务逻辑，而 amis 只需 JSON 配置就能完成完整功能开发，包括数据获取、表单提交及验证等功能，做出来的页面不需要经过二次开发就能直接上线；

*   大量内置组件（100+），一站式解决：其它 UI 框架大部分都只有最通用的组件，如果遇到一些稍微不常用的组件就得自己找第三方，而这些第三方组件往往在展现和交互上不一致，整合起来效果不好，而 amis 则内置大量组件，包括了富文本编辑器、代码编辑器、diff、条件组合、实时日志等业务组件，绝大部分中后台页面开发只需要了解 amis 就足够了；

*   支持扩展：除了低代码模式，还可以通过 自定义组件 来扩充组件，实际上 amis 可以当成普通 UI 库来使用，实现 90% 低代码，10% 代码开发的混合模式，既提升了效率，又不失灵活性；

*   容器支持无限级嵌套：可以通过嵌套来满足各种布局及展现需求；

*   经历了长时间的实战考验：amis 在百度内部得到了广泛使用，在 5 年多的时间里创建了 3.8 万页面，从内容审核到机器管理，从数据分析到模型训练，amis 满足了各种各样的页面需求，最复杂的页面有超过 1 万行 JSON 配置。

### 上手 amis

从官网得知 amis 有两种使用方法

*   JS SDK，可以用在任意页面中

*   React，可以用在 React 项目中

SDK 版本适合对前端或 React 不了解的开发者，它不依赖 npm 及 webpack，可以像 Vue/jQuery 那样外链代码就能使用。

我们选择的是更通用的 JS SDK。

### 搭建 demo

首先创建一个工程目录，比如 amis-demo, 创建 css、js、public、index.html 等目录和文件

然后运行 `npm i amis` 下载 sdk, 在 node\_modules\amis\sdk 目录里就能找到

项目目录结构大概是这样

![](https://tva1.sinaimg.cn/large/008i3skNly1gu9abknvu5j60ca0cyaae02.jpg)

然后编辑 `index.html`

```html
<!DOCTYPE html>
<html lang="zh">
  <head>
    <meta charset="UTF-8" />
    <title>amis demo</title>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <meta
      name="viewport"
      content="width=device-width, initial-scale=1, maximum-scale=1"
    />
    <meta http-equiv="X-UA-Compatible" content="IE=Edge" />
    <link rel="stylesheet" href="sdk.css" />
    <link rel="stylesheet" href="helper.css" />
    <!-- 从 1.1.0 开始 sdk.css 将不支持 IE 11，如果要支持 IE11 请引用这个 css，并把前面那个删了 -->
    <!-- <link rel="stylesheet" href="sdk-ie11.css" /> -->
    <!-- 不过 amis 开发团队几乎没测试过 IE 11 下的效果，所以可能有细节功能用不了，如果发现请报 issue -->
    <style>
      html,
      body,
      .app-wrapper {
        position: relative;
        width: 100%;
        height: 100%;
        margin: 0;
        padding: 0;
      }
    </style>
  </head>
  <body>
    <div id="root" class="app-wrapper"></div>
    <script src="sdk.js"></script>
    <script type="text/javascript">
      (function () {
        let amis = amisRequire('amis/embed');
        // 通过替换下面这个配置来生成不同页面
        let amisJSON = {
          type: 'page',
          title: '表单页面',
          body: {
            type: 'form',
            mode: 'horizontal',
            api: '/saveForm',
            body: [
              {
                label: 'Name',
                type: 'input-text',
                name: 'name'
              },
              {
                label: 'Email',
                type: 'input-email',
                name: 'email'
              }
            ]
          }
        };
        let amisScoped = amis.embed('#root', amisJSON);
      })();
    </script>
  </body>
</html

```

以上是根据官方文档写的，运行会报错，要解决两个问题

1 引入的 js 和 css 路径要写成自己的，所以我从 node\_modules 找到我需要的文件 copy 到自己对应的 css 和 js 目录，所以我的引用代码类似这样

```html
 <link rel="stylesheet" href="css/sdk.css" />
    <link rel="stylesheet" href="css/cxd.css" />
    <link rel="stylesheet" href="css/antd.css" />
    <link rel="stylesheet" href="css/helper.css" />
    <script src="https://cdn.jsdelivr.net/npm/vue@2"></script>
    <script src="https://cdn.jsdelivr.net/npm/history/umd/history.js"></script>
```

2 会报 Cannot read property 'locale' of undefined

解决方法是

先给 amis.embed() 的第四个参数传入一个空对象，例如：

```javascript
 let amisScoped = amis.embed(
            '#root', 
            amisJSON,
            {
                // 这里是初始 props
            },
            {} // 空对象
            
        );

```

为了方便调试我用的 IDE 是 VSCode，安装了 vscode 插件 Live Server，然后右键点击 Open with Live Server 即可在浏览器实时预览页面

接下来就可以根据文档的描述一个个的边写代码边调试体验所有的功能了。

接着你就会发现，原来主要需要开发的是 json, 要是能自动生成 json 就好了。

这个可以有～

不用自己编辑 json，通过编辑器直接拖拽组件自动生成，目前 amis-editor 未开源，但可以免费使用（包括商用）

*   地址：[https://aisuda.github.io/amis-editor-demo/#](https://aisuda.github.io/amis-editor-demo/# "https://aisuda.github.io/amis-editor-demo/#")

*   github 地址：[https://github.com/aisuda/amis-editor-demo](https://github.com/aisuda/amis-editor-demo "https://github.com/aisuda/amis-editor-demo")

为了让页面有个框架（菜单），从 [https://github.com/aisuda/amis-admin](https://github.com/aisuda/amis-admin "https://github.com/aisuda/amis-admin") 项目，参考（抄）了部分代码，让页面有个一个基本的架构，大概是这样：

![](https://tva1.sinaimg.cn/large/008i3skNly1gu9av66vacj621l0u0q4o02.jpg)

我们将每个子页面的具体 json 放到了`pages`目录，所以子页面 json 编辑就从`pages`里找到对应的 json 文件就可以了。

比如，这个`demo.json`

```json
{
    "type": "page",
    "title": "demo 示例页面",
    "body": [
      {
        "type": "tpl",
        "tpl": "这是你刚刚新增的页面。",
        "inline": false
      },
      {
        "type": "chart",
        "config": {
          "xAxis": {
            "type": "category",
            "data": [
              "Mon",
              "Tue",
              "Wed",
              "Thu",
              "Fri",
              "Sat",
              "Sun"
            ]
          },
          "yAxis": {
            "type": "value"
          },
          "series": [
            {
              "data": [
                820,
                932,
                901,
                934,
                1290,
                1330,
                1320
              ],
              "type": "line"
            }
          ]
        },
        "replaceChartOption": true,
        "api": ""
      },
      {
        "type": "tabs",
        "tabs": [
          {
            "title": "tab1",
            "body": [
              {
                "type": "tpl",
                "tpl": "第一个选项卡",
                "inline": false
              },
              {
                "type": "list-select",
                "label": "列表",
                "name": "list",
                "options": [
                  {
                    "label": "选项 A",
                    "value": "A"
                  },
                  {
                    "label": "选项 B",
                    "value": 0
                  },
                  {
                    "label": "options3",
                    "value": true
                  }
                ]
              }
            ]
          },
          {
            "title": "tab2",
            "body": [
              {
                "type": "tpl",
                "tpl": "这是必有项",
                "inline": false
              },
              {
                "type": "input-text",
                "label": "文本",
                "name": "text"
              }
            ]
          }
        ],
        "tabsMode": "chrome"
      },
      {
        "type": "matrix-checkboxes",
        "name": "matrix",
        "label": "矩阵开关",
        "rowLabel": "行标题说明",
        "columns": [
          {
            "label": "列 1"
          },
          {
            "label": "列 2"
          }
        ],
        "rows": [
          {
            "label": "行 1"
          },
          {
            "label": "行 2"
          }
        ]
      },
      {
        "type": "steps",
        "value": "3",
        "steps": [
          {
            "title": "第一步",
            "subTitle": "副标题",
            "description": "描述"
          },
          {
            "title": "第二步"
          },
          {
            "title": "第三步"
          },
          {
            "type": "wrapper",
            "body": "子节点内容",
            "title": "第四步"
          }
        ],
        "status": "wait",
        "source": ""
      },
      {
        "type": "crud",
        "api": {
          "method": "get",
          "url": "https://yapi.gaolvzongheng.com/mock/387/list",
          "replaceData": false,
          "responseData": null,
          "dataType": "json",
          "responseType": "blob",
          "data": null
        },
        "bulkActions": [
          {
            "type": "button",
            "level": "danger",
            "label": "批量删除",
            "actionType": "ajax",
            "confirmText": "确定要删除？",
            "api": "get:/xxx/batch-delete"
          },
          {
            "type": "button",
            "level": "danger",
            "label": "批量编辑",
            "actionType": "dialog",
            "dialog": {
              "title": "批量编辑",
              "size": "md",
              "body": {
                "type": "form",
                "api": "/xxx/bacth-edit",
                "body": [
                  {
                    "label": "字段 1",
                    "text": "字段 1",
                    "type": "input-text"
                  }
                ]
              }
            }
          }
        ],
        "itemActions": [
          {
            "label": "按钮",
            "type": "button",
            "hiddenOnHover": true
          }
        ],
        "features": [
          "create",
          "filter",
          "bulkDelete",
          "bulkUpdate",
          "update",
          "view",
          "delete"
        ],
        "headerToolbar": [
          {
            "label": "新增",
            "type": "button",
            "actionType": "dialog",
            "dialog": {
              "title": "新增",
              "body": {
                "type": "form",
                "api": "xxx/create",
                "body": [
                  {
                    "label": "字段 1",
                    "text": "字段 1",
                    "type": "input-text"
                  }
                ]
              }
            }
          },
          {
            "type": "bulk-actions"
          },
          {
            "type": "pagination"
          },
          {
            "type": "statistics",
            "tpl": "内容"
          },
          {
            "type": "load-more",
            "tpl": "内容"
          },
          {
            "type": "export-excel",
            "tpl": "内容"
          }
        ],
        "filter": {
          "title": "查询条件",
          "body": [
            {
              "type": "input-text",
              "name": "keywords",
              "label": "关键字"
            }
          ],
          "autoFocus": true
        },
        "perPageAvailable": [
          1
        ],
        "messages": {},
        "keepItemSelectionOnPageChange": true,
        "primaryField": "id",
        "labelTpl": "${name}",
        "draggable": true,
        "footable": {
          "expand": "all"
        },
        "title": "demo 表格",
        "mode": "table",
        "columns": [
          {
            "name": "id",
            "label": "ID",
            "type": "text",
            "placeholder": "-",
            "sortable": true,
            "fixed": ""
          },
          {
            "name": "name",
            "label": "name",
            "type": "text",
            "placeholder": "-"
          },
          {
            "type": "operation",
            "label": "操作",
            "buttons": [
              {
                "label": "编辑",
                "type": "button",
                "actionType": "dialog",
                "level": "link",
                "dialog": {
                  "title": "编辑",
                  "body": {
                    "type": "form",
                    "api": "xxx/update",
                    "body": [
                      {
                        "name": "id",
                        "label": "ID",
                        "type": "input-text"
                      },
                      {
                        "name": "engine",
                        "label": "渲染引擎",
                        "type": "input-text"
                      }
                    ]
                  }
                }
              },
              {
                "label": "查看",
                "type": "button",
                "actionType": "dialog",
                "level": "link",
                "dialog": {
                  "title": "查看详情",
                  "body": {
                    "type": "form",
                    "body": [
                      {
                        "name": "id",
                        "label": "ID",
                        "type": "input-text"
                      },
                      {
                        "name": "engine",
                        "label": "渲染引擎",
                        "type": "input-text"
                      }
                    ]
                  }
                }
              },
              {
                "type": "button",
                "label": "删除",
                "actionType": "ajax",
                "level": "link",
                "className": "text-danger",
                "confirmText": "确定要删除？",
                "api": "/xxx/delete"
              }
            ]
          }
        ],
        "footerToolbar": [
          {
            "type": "load-more"
          },
          {
            "type": "pagination"
          }
        ],
        "alwaysShowPagination": true,
        "filterTogglable": false,
        "perPage": 6,
        "checkOnItemClick": true,
        "initFetch": true,
        "quickSaveApi": ""
      }
    ],
    "messages": {},
    "name": "demo",
    "subTitle": "这是副标题",
    "remark": "这是一个提示",
    "aside": []
  }

```

看起来很长对吧，但都是用 editor 自动生成的

![](https://tva1.sinaimg.cn/large/008i3skNly1gu9azinz7kj61pq0u0q9m02.jpg)

最终 html 页面是这样，一模一样

![](https://tva1.sinaimg.cn/large/008i3skNly1gu9b0ipey4j61pg0u043002.jpg)

可以看到页面上有个表格，那么数据是从哪儿来的呢？
我是利用 yapi 先定义好接口契约，然后通过 yapi mock 出来的，当然 mock 规则你可以自定义

![](https://tva1.sinaimg.cn/large/008i3skNly1gu9b2h0pfzj61x00u0dis02.jpg)

接下来我只需要把后端接口真正实现（java、python、go 随你），然后把 API 地址重新配置好就可以了，一个复杂的功能前端通过 editor，后端自己实现，中间契约用 yapi，实现起来是非常快的。

剩下的就是还有很多 editor 的细节，以及 amis 的细节可能要通过文档+实践总结+案例来摸索了，但整体看学习成本并不高。对开发非常友好。

## 参考

*   [https://baidu.gitee.io/amis/zh-CN/docs/index](https://baidu.gitee.io/amis/zh-CN/docs/index "https://baidu.gitee.io/amis/zh-CN/docs/index")
