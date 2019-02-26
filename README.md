# mk-dash-ssr

# 介绍

市场后台项目 (SSR 模式)

# 软件架构

1. 支持 SEO
1. Next.js + Ant Design

# 设置

## 开发开发

- 下载代码： git clone https://gitee.com/bigjs/mk-dash-ssr.git
- 安装依赖： npm i
- 设置配置文件： 复制 config.json-dist 为 config.json 并修改内容，设置端口和 api 地址
- 开发模式： node server/server.js
- 生产模式： npm run build && npm run prod
- 生产模式 PM2 启动

```Bash
pm2 start start.json
```

start.json:

```Javascript
{
   "apps": [{
    "name":"yes-dash-ssr",
    "cwd": "/home/yuenshui/mk-dash-ssr",
    "script":"/home/yuenshui/mk-dash-ssr/server/server.js",
    "instances":1,
    "exec_mode":"cluster",
    "env": {
        "NODE_ENV": "production"
    }
  }]
}
```

## 路由设置

1.  默认路由
    正常开发过程中 url 和页面 js 代码映射关系不需要配置，规则如下：
    http://host/pagename => /pages/pagename.js
2.  指定路由
    /router.js

```Javascript
"use strict"
/**
 * 默认路由不需要设置，请尽量使用默认路由进行开发
 * http://host/  =>   /pages/index.js
 * http://host/about  =>  /pages/about/index.js or /pages/about.js
 *
 * 路由配置如下格式：
 * '/index.html': { page: '/' },
 * '/h5': { page: '/single/h5' },
 */
module.exports = {
};
```

## vscode 调试

- 由于 next.js 编译执行会有缓存，程序修改保存后刷新页面才会触发执行，并可以断点调试。
- launch.json 设置如下：

```json
{
  // 使用 IntelliSense 了解相关属性。
  // 悬停以查看现有属性的描述。
  // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Launch Program",
      "program": "${workspaceFolder}/server/server.js"
    }
  ]
}
```

## 服务器配置

#### 安装

- config.json-dist 是服务器配置模板，安装时
  ```Bash
  # cp config.json-dist config.json
  ```
  <font color='red'>注意：`config.json` 不要提交到 git 代码库</font>

#### 设置

- port 服务端口
- API 服务器调用 API 的设置
- API.host 服务器 API 地址，如：http://127.0.0.1:6021
- proxy API 反向代理配置，可以映射多服务器；[文档参考](https://www.npmjs.com/package/http-proxy-middleware)

```Javascript
{
  "port": 3000,
  "API": {
    "host": "http://127.0.0.1:6021"
  },
  "imageDownStr": "/oss/",
  "proxy":  {
    "/api": {
      "target": "http://127.0.0.1:6021",
      "pathRewrite": {
        "^/api": "/"
      },
      "changeOrigin": true
    }
  }
}
```

## 服务器 API

- 服务器 API 封装在/lib/api.js
- 服务器使用 API 数据不要在业务中直接调用 API 接口，将接口封装在/lib/api.js 中，有利于复用和统一管理
- /lib/api.js 中增加接口调用如下：

```javascript
/**
 * 订单列表接口
 * @param token {string} user cookie-token
 * @param role {string} user role
 * @param params {object} filter, orderBy, limit, startPos
 */
API.orderList = (token, role, params) => {
  params = params || {};
  role = role || "mk_admin";
  let url = config.API.host + `/api/mk/${role}/get_order`;
  return API.httpAuth(url, token, {
    method: "POST",
    type: "json",
    data: params
  });
};
```

- 页面（服务器）使用 API，在页面模块的`getInitialProps`方法中引入`/lib/api.js`
  为了避免服务器代码被打包到浏览器 JS 中，在`getInitialProps`内载入 API，不要在文件头部载入，也不要用`import`载入

```Javascript
 const API = require("../../../lib/api");
 let data;
 try {
   data = await API.orderList(user.token, user.role, searchParams);
 } catch (error) {
   console.error("api error:", error);
 }
```

## 用户态

- 系统采用浏览器通过`localStorage`保存`token`和`user`对象，并将`token`存入`cookie`。访问网页时服务器 JS 通过`cookie`获取当前用户`token`，并携带`token`调用 API 接口获取数据。
- 浏览器端的 API 调用，封装在`/pages/functions/api.js`
- 使用 API，页面头部引入

```Javascript
import API from './functions/api.js';
class PageMain extends React.Component {

handleSubmit = async e => {
  e.preventDefault();
  this.props.form.validateFields((err, fieldsValue) => {
    if (err) {
      return;
    }
    fieldsValue.password = fieldsValue.password || "";
    API.login(fieldsValue.account, fieldsValue.password)
      .then(rs => {
        if (rs.status === "success") {
          localStorage.user = JSON.stringify(rs.data.item);
          let role = rs.data.item.role;

          window.location.href = `/${role}/dash`;
          message.success("尚层市场欢迎您的使用");
        } else {
          message.error("登录失败，请重新输入");
        }
      })
      .catch(err => {
        console.error("API login error:", err);
      });
  });
};

render() {
  const { getFieldDecorator } = this.props.form;
  return (
      <Form onSubmit={this.handleSubmit}>
      ......
      </Form>
  );
};
}

export default Form.create()(PageMain);
```

## SEO

- 设置 Layout 的 title、keywords、description 属性即可。
- 具体内容规范[参考文档](http://www.w3school.com.cn/tags/tag_meta.asp) [教程](https://www.cnblogs.com/lijinwen/p/5699443.html)

```Javascript
"use strict";
import Layout from "../../layout/index";

class PageServer extends React.Component {
  render() {
    return (
      <Layout
        title="尚层装饰前台"
        keywords="keywords-keywords"
        description="description-description"
      >
      hello world
      </Layout>
    );
  };
};
```

## 富文本编辑（百度编辑器）

封装[百度编辑器](http://ueditor.baidu.com/website/download.html#ueditor)，参考[文档](http://fex.baidu.com/ueditor/)，组件支持 Next.js

#### demo：

```JSX
"use strict";

import Editor from "./layout/ueditor";

class PageServer extends React.Component {
  constructor(props) {
    super(props);
    this.state = { ...props };
  }

  render() {
    return <MyFORM data={this.state} />;
  }
}

PageServer.getInitialProps = async ctx => {};

class PageForm extends React.Component {
  static async getInitialProps({ req, query }) {
    console.log(req, query);
  }
  constructor(props) {
    super(props);
    this.state = { ...props };
    this.state.context = "hello world";
  }

  callChange = value => {
    console.log("callChange", value);
  };

  clickDemo1 = () => {
    // 主动获取内容，支持的接口参考下面的文档
    console.log(this.refs.demo1.getContent());
  };

  clickDemo2 = () => {
    console.log(this.refs.demo2.getContent());
  };

  render() {
    return (
      <>
        <div onClick={this.clickDemo1}>{this.state.context}</div>
        <Editor
          onChange={this.callChange} // 订阅内容变化
          id="demo1"
          ref="demo1"
          width="1024"
          height="100"
        >
          {this.state.context}
        </Editor>

        <div onClick={this.clickDemo2}>{this.state.context}</div>
        <Editor
          onChange={this.callChange}
          id="demo2"
          ref="demo2"
          width="1024"
          height="100"
        >
          {this.state.context}
        </Editor>
      </>
    );
  }
}

let MyFORM = Form.create()(PageForm);

export default PageServer;

```

#### Ueditor Event

- onChange 消息参数为编辑内容。
- onReady 编辑器初始化完成。

#### Ueditor Method

_使用下列方法，需要设定组件 ref 属性（参考 demo）_

- getContent 获取编辑器 HTML 内容，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:getContent()>)
- setContent 设置编辑 HTML 内容，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:setContent()>)
- getContentTxt 获取编辑器中的纯文本内容,没有段落格式，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:getContentTxt()>)
- getPlainTxt 得到编辑器的纯文本内容，但会保留段落格式，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:getPlainTxt()>)
- hasContents 检查编辑区域中是否有内容，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:hasContents()>)
- focus 让编辑器获得焦点，默认 focus 到编辑器头部，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:focus()>)
- isFocus 编辑器是否得到了选区，[参考文档](<https://ueditor.baidu.com/doc/#UE.dom.Selection:isFocus()>)
- setDisabled 设置当前编辑区域不可编辑，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:setDisabled()>)
- setEnabled 设置当前编辑区域可以编辑，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:setEnabled()>)
- setHide 隐藏编辑器，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:setHide()>)
- setShow 显示编辑器，[参考文档](<https://ueditor.baidu.com/doc/#UE.Editor:setShow()>)
- getText 在当前光标位置插入 html 内容，[参考文档](<https://ueditor.baidu.com/doc/#UE.dom.Selection:getText()>)

- insertHTML 在当前光标位置插入 html 内容，[参考文档](https://ueditor.baidu.com/doc/#COMMAND::inserthtml)
- clear 清空内容，[参考文档](https://ueditor.baidu.com/doc/#COMMAND::cleardoc)
- selectall 选中编辑器内的所有内容，[参考文档](https://ueditor.baidu.com/doc/#COMMAND::selectall)

