---
title: C++语法解析器
date: 2017-03-26 18:40:30
tags: 
  - Node
  - Electron
  - Javascript
categories:
  - Node
---

## 架构分析

1. Node进行文件数据处理
2. Vue.js进行视图的呈现
3. Electron作为客户端

## C++语法解析


### 效果展示

![cppAnalysis](/images/cpp-analysis.gif)

### 读取形式

按照代码行进行读取, 每次读取到一行就进行遍历分析

### 关键字与特殊符号建立映射

```javascript
/**
 * 关键字
 */
let keyWords = [
  'asm', 'auto',
  'bool', 'break',
  'case', 'catch', 'char', 'class', 'const', 'continue', 'cin', 'cout',
  'default', 'delete', 'do', 'double', 'define',
  'else', 'enum', 'except', 'explicit', 'extern', 'endl',
  'false', 'finally', 'float', 'for', 'friend',
  'goto',
  'if', 'inline', 'int', 'include',
  'long',
  'mutable', 'main',
  'namespace', 'new',
  'operator',
  'private', 'protectde', 'public', 'printf',
  'register', 'return',
  'short', 'signed', 'szieof', 'static', 'struct', 'string', 'switch', 'std', 'scanf',
  'template', 'this', 'throw', 'true', 'try', 'typedef', 'typename',
  'union', 'unsigned', 'using',
  'virtual', 'void',
  'while'
]

/**
 * 特殊字符
 */
let specialWords = [
  ',', ';', '(', ')', '{', '}', '#', '^', '?', ':', '.', '[', ']', '+', '-', '*', '/', '%', '=', '>', '<', '!', '~', '|', '&',
  '&&', '||', '==', '>=', '<=', '!=', '++', '--', '::', '<<', '>>', '+=', '-=', '*=', '/=', '%=', '&=', '^=', '->'
]

keyWords.forEach(word => {
  wordsMap.set(word, 'keyWord')
})

specialWords.forEach(word => {
  wordsMap.set(word, 'specialWord')
})
```

### 遍历分析过程

- 代码注释匹配

  - 当读到一行中包含\/*的两个字符时候, 这时把代码注释的标志设置为true, 然后一直读取，知道遇到\*/的时候就把标志重新置为false
  - 判断是否可以构成单词，成立的条件是不以数字开头，并且只包含数字, 下划线, 以及字母, 可以通过正则来/[a-z]|[A-z]|\_/匹配
  - 判断是否为字符串或者字符, 成立条件是以",'开头
  - 判断是否为数字
  - 判断是否为特殊字符, 这时就通过建立的映射进行关键字查找
  - 判断空格

- 代码解释

判断工具函数

```javascript

//判断是否是字母与下划线
function judgeWord(word) {
  let wordPatten = /[a-z]|[A-z]|\_/
  return wordPatten.test(word)
}

//判断是否为数字
function judgeNumber(number) {
  let numberPatten = /[0-9]/
  return numberPatten.test(number)
}

//判断是否为特殊字符
function judgeSpecialWord(letter) {
  return wordsMap.get(letter) === 'specialWord' ? true : false
}

//判断是否为关键词
function judgeKeyWord(letter) {
  return wordsMap.get(letter) === 'keyWord' ? true : false
}

```

行分析函数

```javascript
exports.analysisLine = (line, annotation) => {
  let oneLine = []
  let word = ''
  let i = 0
  let includeFile = false
  while (i < line.length) {
    //注释代码的优先级最高，需要先进行判断
    if (line[i] === '/' && line[i + 1] === '*') {
      word += line[i]
      i++
      annotation = true;
    }
    if (!annotation) {
      //表示不是注释内容
      if (judgeWord(line[i])) {
        //表示是属于字母与下划线
        word += line[i]
        while (i + 1 < line.length && (judgeNumber(line[i + 1]) || judgeWord(line[i + 1]))) {
          i++
          word += line[i]
        }
        if (judgeKeyWord(word)) {
          //判断单词是否属于关键词
          oneLine.push({
            word: word,
            type: 'keyWord'
          })
        } else {
          //不为关键词, 则为标识符
          oneLine.push({
            word: word,
            type: 'identifier'
          })
        }
        //由于include是属于头文件的引入, 需要对头文件进行特殊处理
        if (word === 'include') {
          includeFile = true
        }
        word = ''
        i++
      } else if (line[i] === '"') {
        //字符串判断
        while (i + 1 < line.length && line[i + 1] !== '"') {
          word += line[i]
          i++
        }
        word += (line[i] || '' )+ (line[i + 1] || '')
        oneLine.push({
          word: word,
          type: 'string'
        })
        word = ''
        i += 2
      } else if (line[i] === '\'') {
        //字符判断
        while (i + 1 < line.length && line[i + 1] !== '\'') {
          word += line[i]
          i++
        }
        word += (line[i] || '' )+ (line[i + 1] || '')
        oneLine.push({
          word: word,
          type: 'char'
        })
        word = ''
        i += 2
      } else if (judgeNumber(line[i])) {
        //数字判断
        word += line[i]
        while (i + 1 < line.length && (judgeNumber(line[i + 1]) || line[i + 1] === '.')) {
          i++
          word += line[i]
        }
        oneLine.push({
          word: word,
          type: 'number'
        })
        word = ''
        i++
      } else if (judgeSpecialWord(line[i])) {
        //特殊字符判断
        if (line[i] === '<' && includeFile) {
          //处理头文件的引入
          oneLine.push({
            word: line[i],
            type: 'specialWord'
          })
          i++
          word += line[i]
          while (i + 1 < line.length && line[i + 1] !== '>') {
            i++
            word += line[i]
          }
          oneLine.push({
            word: word,
            type: 'libraryFile'
          })
          word = ''
          i++
        } else {
          //处理//的注释代码
          if (line[i] === '/' && line[i + 1] === '/') {
            i++
            while (i + 1 < line.length) {
              i++
              word += line[i]
            }
            oneLine.push({
              word: word,
              type: 'Annotations'
            })
            word = ''
            i += 3
          } else {
            word += line[i]
            while (i + 1 < line.length && (judgeSpecialWord(word + line[i + 1]))) {
              i++
              word += line[i]
            }
            oneLine.push({
              word: word,
              type: 'specialWord'
            })
            word = ''
            i++
          }
        }
      } else if (line[i] === ' ') {
        oneLine.push({
          word: line[i],
          type: 'space'
        })
        i++
      }
    } else {
      //表示注释内容
      while (i + 1 < line.length && (line[i + 1] !== '*' && line[i + 2] !== '/')) {
        word += line[i]
        i++
      }
      word += line[i] + (line[i + 1] || '') + (line[i + 2] || '')
      oneLine.push({
        word: word,
        type: 'Annotations'
      })
      annotation = false;
      word = ''
      i += 3
    }
  }
  return oneLine
}
```

### 界面实现过程

- Electron, Vue搭建

具体的搭建过程省略, 可查看源码或自行查找资料

- 动态编辑显示

  - 数据绑定, 使用v-model结合watch进行数据监控, 当数据发生变化的时候重新分析每行数据
  - 二维数组保存分析的数据, 使用二维数组保存分析完的词法, 保留原生的代码行
  - 通过v-bind:class对不同类型的词法进行代码高亮的简单呈现

```jsx
<template>
  <div class="container">
    <div class="navigator">
      <button class="menu-btn" @click="open">open</button>
      <button class="menu-btn" @click="save">save</button>
    </div>
    <div class="content">
      <div class="code-input">
        <textarea v-model="codeInput">
        </textarea>
      </div>
      <div class="code-show">
        <div v-for="(line, index) in codeShowArr">
          <p v-for="word in line" :class="word.type">
            <template v-if="word.type==='space'">
              &nbsp
            </template>
            <template v-else>
              {{word.word}}
            </template>
          </p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { analysisLine } from '../controllers/analysis'
import fileController from '../controllers/fileController'

export default {
  watch: {
    codeInput: function() {
      this.codeShowArr = []
      this.codeInput.split(/\r?\n/).forEach(line => {
        let wordArr = []
        analysisLine(line, this.annotation).forEach(word => {
          wordArr.push(word)
        })
        this.codeShowArr.push(wordArr)
      })
    }
  },
  data () {
    return {
      codeInput: '',
      codeShow: '',
      codeShowArr: [],
      annotation: false
    }
  },
  methods: {
    open() {
      fileController.openDialog().then(filePath => {
        return fileController.readPackage(filePath)
      }).then(res => {
        this.codeInput = res
      })
    },
    save() {
      fileController.openDialog().then(filePath => {
        return fileController.writePackage(filePath, this.codeShowArr)
      })
    }
  }
}
</script>
```

- 文件的读取和写入

  - 使用Electron中引入Node的fs模块来进行文件读取写入处理
  - 使用Electron的Dialog控件选取文件

```javascript

import Promise from 'bluebird'
import electron from 'electron'
const {dialog} = electron.remote
const fs = Promise.promisifyAll(require('fs'));

const writePackage = (filePath, data) => {
  return fs.writeFileAsync(filePath, JSON.stringify(data, null, 2)).catch(err => {
    return Promise.reject(err)
  });
}

const readPackage = (filePath) => {
  return fs.readFileAsync(filePath, 'utf-8').catch(err => {
    return Promise.reject(err)
  });
}

const openDialog = () => {
  return new Promise((resolve, reject) => {
    dialog.showOpenDialog({
      properties: [
        'openFile',
      ]
    }, (res) => {
      if(!res) {
        return reject('404')
      }
      return resolve(res[0])
    });
  })
}

export default {
  writePackage,
  readPackage,
  openDialog
}

```

对此, 一个简单的C++词法分析就完成了, 过程并不复杂, 只要理解了优先级和并列情况的区分就比较清晰的写出, 由于
个人匆匆写完这个程序, 并没有对js部分的代码进行优化, 质量过低, 仅供参考

