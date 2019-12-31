---
title: 从零开始打造属于自己的UI库
date: 2019-12-30 19:44:01
tags:
  - Vue
categories:
  - 前端
---


目前主流的 Vue UI 框架很多, 每个 UI 框架的风格各有各的特点, 国内目前知名度较高的莫过于 Element 和 iView

大概是 16 年的时候，那时候刚开始上手 Vue2.0, 同时也分别使用了 Element 和 iView 两个 UI 框架来实现一些运营系统的前端开发。

从那时候开始就想着，能写出这么完善和优秀的框架得需要多久啊！内心一直很渴望也能够自己写一写 UI 框架，然后自己设计，自己配色。

不求推广到给别人使用！只求能够自己过把瘾！以及在写 UI 框架的过程中能够学习和沉淀多一丁点前端技术。


![Vue](/images/best-ui-banner.png)

> The best-ui is not the best !important;

__PS: Best UI 大部分学习和借鉴了 Element 和 iView 组件库的源码__

## 项目结构

.
+-- build
+-- examples
+-- lib
+-- packages
    +-- components
    +-- theme
+-- src
    +-- mixins
    +-- utils
    +-- index.js
+-- tests
+-- types
+-- babel.config.js
+-- components.json
+-- gulpfile.js
+-- package.json

看过 Element 的源码的同学估计一目了然，这里主要参考了 Element UI 的项目结构

简单讲一下这里的项目目录和文件的作用

- build 主要是和构建相关
- examples 用于编写一些简单的 demo 代码，开发组件调试所用
- lib 是打包构建完成后代码的存放目录
- packages 是每个组件的源码实现, components 表示每个组件独立的目录，theme 存放组件的样式 .scss 文件
- src 是 UI 框架的入口文件存放位置，以及一些工具函数，mixins，plugins 文件的存放
- tests 单元测试和集成测试（不好意思，写这篇文章的时候一个组件的单元测试都还没写，我是个辣鸡）
- types d.ts 的类型声明文件（写这篇文章的时候所有组件的 d.ts 文件还没完成补齐，我是个辣鸡第二遍）
- babel.config.js babel 的配置声明
- components.json 用来指定具体组件的源码位置，用于构建多个组件单独的文件
- gulpfile 用于 gulp 进行样式构建的运行文件

一个前端工程化项目的结构出来了，按照个人的开发习惯，接下来就是要对项目的构建进行配置了，不然光写代码而运行不了，没什么意义。

## 构建过程配置

对于 Webpack 的配置，本来一开始是使用 vue-cli 来生成对应的构建配置，但是后面发现要去修改适配组件库的构建流程有点鸡肋，所以就干脆自己大致按照 Element 的方式去做一遍配置，每个环节这里都会详细介绍

build 这个构建配置的目录如下：

+-- build
    +-- bin
        +-- build-component.js
        +-- build.js
        +-- watch.js
    +-- utils
    +-- config.js
    +-- webpack.base.js
    +-- webpack.build.js
    +-- webpack.component.js
    +-- webpack.watch.js

bin 目录三个文件代表了三个运行的命令

- build-component 主要是用于构建多个组件的单文件。
- build 表示构建整体的组件库，包含多个组件的合并。
- watch 用于开发构建过程中预览调试

utils 中只有一个 resolve 用来做路径的解析

### Webpack 的 Base 配置  

webpack.base.js:

```js
const HappyPack = require('happypack')
const os = require('os')
const happyThreadPool = HappyPack.ThreadPool({ size: os.cpus().length })
const { VueLoaderPlugin } = require('vue-loader')
const { resolve } = require('./utils/resolve')
const ProgressBarPlugin = require('progress-bar-webpack-plugin')

module.exports = {
  resolve: {
    extensions: ['.vue', '.js', '.json'],
    alias: {
      '@': resolve(process.cwd(), 'examples'),
      '~': resolve(process.cwd(), 'packages'),
      src: resolve(process.cwd(), 'src')
    }
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'happypack/loader?id=eslint',
        enforce: 'pre'
      },
      {
        test: /\.js$/,
        loader: 'happypack/loader?id=babel'
      },
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          loader: {
            js: 'happypack/loader?id=babel'
          }
        }
      },
      {
        test: /\.scss$/,
        use: [
          'style-loader',
          'css-loader',
          'sass-loader'
        ]
      },
      {
        test: /\.(woff2?|eot|ttf|otf)(\?.*)?$/,
        loader: 'url-loader',
        options: {
          limit: 10000,
          name: 'fonts/[name].[hash:7].[ext]'
        }
      }
    ]
  },
  plugins: [
    new HappyPack({
      id: 'eslint',
      loaders: [{
        loader: 'eslint-loader',
        options: {
          formatter: require('eslint-friendly-formatter')
        }
      }],
      verbose: false,
      threadPool: happyThreadPool
    }),
    new HappyPack({
      id: 'babel',
      loaders: [{
        loader: 'babel-loader',
        options: {
          cacheDirectory: true
        }
      }],
      verbose: false,
      threadPool: happyThreadPool
    }),
    new ProgressBarPlugin(),
    new VueLoaderPlugin()
  ]
}

```

使用 happypack 来进行 babel 的转码，配置对应需要的 loader 和 plugins

配置 resolve.alisa 对一些常用的目录文件路径进行缩写配置

这里没有指定 entry 和 output 主要是让其他的环境配置进行指定

### Watch 模式

webpack.base.js: 

```js
const merge = require('webpack-merge')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const baseConf = require('./webpack.base')

const config = merge(baseConf, {
  mode: 'development',
  entry: './examples/main.js',
  plugins: [
    new HtmlWebpackPlugin({
      template: './examples/template/index.html'
    })
  ]
})

module.exports = config

```

bin/watch.js:

```js
const webpack = require('webpack')
const config = require('../webpack.watch')
const middleware = require('webpack-dev-middleware')

const compiler = webpack(config)
const express = require('express')
const app = express()

app.use(
  middleware(compiler, {
  })
)

app.listen(3000, () => console.log('Example app listening on port 3000!'))

```

Watch 模式下使用 webpack-dev-middleware 结合 express 实现一个简单的 dev-server 进行开发预览，对应的入口文件为我们 examples 下的 main.js 文件。

### build 模式

webpack.build.js: 

```js
const merge = require('webpack-merge')
const baseConf = require('./webpack.base')
const { externals } = require('./config')
const { resolve } = require('./utils/resolve')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')

const config = merge(baseConf, {
  mode: 'production',
  entry: resolve(process.cwd(), 'src', 'index.js'),
  output: {
    path: resolve(process.cwd(), './lib'),
    publicPath: '/dist/',
    filename: 'index.js',
    library: 'BEST',
    libraryTarget: 'umd'
  },
  performance: {
    hints: false
  },
  externals,
  plugins: [
    new CleanWebpackPlugin()
  ]
})

module.exports = config

```

指定 mode 为 production，配置对应的 entry 为 src 下的 index.js ， 用于进行全局构建，对应的 output 路径为 lib，指定的 libraryTarget 为 umd 模块。

bin/build.js:

```js
const webpack = require('webpack')
const config = require('../webpack.build')

const compiler = webpack(config)

compiler.run(function (err, stats) {
  if (err) {
    return console.error(err)
  }
})

```

直接借助 webapck 的 compiler 执行构建

### Build Component 模式

这里说明一下为什么要构建单个的组件文件，因为涉及到 __按需加载__ 的使用，我们需要对单个的组件进行构建，这里可以借助 webpack 的多文件构建模式来执行

webapck.component.js:

```js
const merge = require('webpack-merge')
const baseConf = require('./webpack.base')
const { externals } = require('./config')
const { resolve } = require('./utils/resolve')

const components = require('../components.json')

const entrys = {}
Object.keys(components).forEach(component => {
  entrys[component] = components[component]
})

const config = merge(baseConf, {
  mode: 'production',
  entry: entrys,
  output: {
    path: resolve(process.cwd(), 'lib'),
    publicPath: '/dist/',
    filename: '[name].js',
    libraryTarget: 'commonjs2'
  },
  optimization: {
    minimize: false
  },
  performance: {
    hints: false
  },
  externals
})

module.exports = config

```

通过读取 component.json 的配置文件来获取构建的组件对应的入口文件路径，对应的 output 路径为 lib

### 样式构建

Best UI 组件库是用 Sass 来进行样式的编写，我们这里为了和 js 的构建区分开，所以在样式的构建上，单独使用 gulp 对 .scss 的样式文件进行构建

gulpfile.js:

```js
'use strict'

const { series, src, dest } = require('gulp')
const sass = require('gulp-sass')
const autoprefixer = require('gulp-autoprefixer')
const cssmin = require('gulp-cssmin')
const basepath = './packages/theme'

function compile () {
  return src(`${basepath}/src/*.scss`)
    .pipe(sass.sync())
    .pipe(autoprefixer({
      cascade: false
    }))
    .pipe(cssmin())
    .pipe(dest('./lib/theme'))
}

function copyfont () {
  return src(`${basepath}/src/fonts/**`)
    .pipe(cssmin())
    .pipe(dest('./lib/theme/fonts'))
}

exports.build = series(compile, copyfont)

```

这里沿用了 Element 的 gulpfile。这个 gulp 的构建过程很简单，主要做两件事情：

- 将 packages/theme/src 下的每个 .scss 和 .css 的文件进行编译，压缩，最后输出到 lib/theme 的目录下
- 将 packages/theme/src 下的字体文件进行压缩并输出到 lib/theme/fonts 的目录下

### package.json 的 script 配置

通过以上完成了对应的构建配置，为了快速执行以上的命令，我们可以在 npm 的 script 中进行配置:

package.json:

```json
{
  "scripts": {
    "watch": "node ./build/bin/watch.js",
    "build:component": "node ./build/bin/build-component.js",
    "build": "node ./build/bin/build.js",
    "build:all": "npm run build & npm run build:component & npm run build:css",
    "build:css": "gulp build --gulpfile ./gulpfile.js",
    "lint": "eslint --fix --ext .js,.vue src packages"
  }
}
```

## 尝试开发一个组件

以上已经完成了对整个构建过程的配置，当我们把必要的工程化工具都配置好了后，可以尝试上手开发一个组件，我们以开发一个 Button 组件为具体实例


### 组件交互逻辑开发

在 package 的目录下新建一个 button 的目录，然后新建以下的文件：

button/index.js:

```js
import BTButton from './src/button'

BTButton.install = function (Vue) {
  Vue.component(BTButton.name, BTButton)
}

export default BTButton

```

index.js 作为 button 组件的入口文件，这里给 BTButton 定义 install 方法，目的是方便通过 vue 的插件形式进行全局组件注册。

组件的逻辑实现在 button/src/button.vue:

```html
<template>
  <button
  :disabled="disabled"
  type="button"
  :class="[
      'bt-button',
      type ? `bt-button--${type}` : '',
      {
        'is-circle': circle,
        'is-disabled': disabled
      }
    ]"
  @click="handleClick">
    <i :class="icon" v-if="icon"></i>
    <span v-if="$slots.default">
      <!-- @slot 默认插槽 -->
      <slot></slot>
    </span>
  </button>
</template>

<script>
/**
 * Button 按钮
 * @displayName Best Button
 */
export default {

  name: 'BtButton',

  props: {
    /**
     * 类型
     * `default,primary,info,warn`
     */
    type: {
      type: String,
      default: 'default'
    },
    /**
     * 是否圆形按钮
     */
    circle: {
      type: Boolean,
      default: false
    },
    /**
     * 图标类名
     */
    icon: {
      type: String,
      default: ''
    },
    /**
     * 是否禁用
     */
    disabled: {
      type: Boolean,
      default: false
    }
  },

  methods: {
    /**
     * Click 事件
     *
     * @event click
     * @type {object}
     */
    handleClick (event) {
      this.$emit('click', event)
    }
  }

}
</script>
```

button 的组件实现并不复杂，这里用一个很简单的例子来说明添加一个组件开发的过程（牛x的同学可以考虑打造 cli 来快速新建一个组件的模板）

### 将组件添加到全局入口文件和配置文件

我们将我们开发完的 button 组件注册到 src/index.js 中：

```js
import Button from '~/button'

const components = [
  Button,
]

const install = function (Vue, opts = {}) {
  components.forEach(component => {
    Vue.component(component.name, component)
  })
}

export default {
  install,
  Button,
}
```

将开发完成的 button 组件注册到全局的入口文件，提供全量引入的方式。

对于组件的配置文件 componets.json:

```json
{
  "button": "./packages/button",
  "best-ui.common": "./src"
}
```

配置 webapck 在构建多个组件文件的过程中每个组件文件对应 entry 的路径 和 output 的文件名。

### 编写 d.ts 类型声明文件

types/button.d.ts:

```ts
import { BestUIComponent } from './components'

export type ButtonType = 'default' | 'primary' | 'info' | 'warn'

export declare class BTButton extends BestUIComponent {

  type?: ButtonType

  circle?: boolean
  
  icon?: string

  disabled?: boolean
}
```

types/components.d.ts:

```ts
import Vue from 'vue'

export declare class BestUIComponent extends Vue {
  static install (vue: typeof Vue): void
}
```

### 编写单元测试

待补充 !-_-!

