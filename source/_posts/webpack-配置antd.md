title: webpack 配置antd
author: Keaper
date: 2018-02-23 23:09:26
tags:
---
## webpack配置antd

webpack配置文件 wepack.config.js
```javascript
//省略
module:{
        rules: [{
            test: /\.js[x]?$/,
            use:['babel-loader?cacheDirectory=true'],
            include:path.join(__dirname,'./src/')
        },{
            test: /\.css$/,
            use: [
                {
                    loader: "style-loader"
                }, {
                    loader: "css-loader"
                },
            ]
        },]
    },
//省略
```

bable配置文件 .bablerc：

```javascript
{
  "presets": [
    "es2015",
    "react",
    "stage-0"
  ],
  "plugins": [[
    "import", {
      "libraryName": "antd",
      "style": "css"
    }]
  ]
}
```

babel-plugin-import 插件用法：[https://github.com/ant-design/babel-plugin-import#usage](https://github.com/ant-design/babel-plugin-import#usage)