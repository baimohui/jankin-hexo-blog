---

title: eslint 和 prettier 配置
categories: 
- eslint
tags: 
- eslint
- prettier
- vscode
---

在团队协作中，为避免低级 Bug、产出风格统一的代码，会预先制定编码规范。使用 Lint 工具和代码风格检测工具，则可以辅助编码规范执行，有效控制代码质量。<!--more-->
## 一、eslint 

ESLint 由《JavaScript 红宝书》作者 Nicholas C. Zakas 编写，2013 年发布第一个版本。NCZ 的初衷不是重复造一个轮子，而是在实际需求得不到 JSHint 团队响应的情况下做出的选择：以可扩展、每条规则独立、不内置编码风格为理念编写一个 lint 工具。

### 1. 项目引入
```node
npm install eslint --save-dev
npx eslint --init
```

执行初始化命令后，项目会生成对应的.eslintrc.js 文件，用来对校验规则进行配置。另外，可以新建.eslintignore 文件，用来添加不受 eslint 检查的文件或目录。
```js
module.exports = {
    "env": {
        "browser": true,
        "es2021": true,
        "node": true
    },
    "extends": [
        //继承 Eslint 中推荐的（打钩的）规则项 http://eslint.cn/docs/rules/
        "eslint:recommended",
        // 此项是用来配置 vue.js 风格
        "plugin:vue/essential"
    ],
    "parserOptions": {
        "ecmaVersion": 13,
        "sourceType": "module"
    },
    "plugins": [
        "vue"
    ],
    "rules": {
    }
};
```
extends 中的规则按排序先后执行。当其中不同配置项中的规则有重复时，默认后面的会替换前面的。rules 里可以配置自定义规则（[eslint 规则大全](http://eslint.cn/docs/rules/)），当和 extends 里的规则有重复时，会默认覆盖掉 extends 中的。

在项目根目录中执行：`npx eslint --ext .vue src/`，这样在运行检查时，如果有不符合规则的代码，则会出现对应的警告或报错。

```node
npx eslint --ext .vue src/
等价于：./node_modules/.bin/eslint --ext .vue src/
--ext:可以指定在指定目录/文件
.vue：vue文件
src/：在src目录下匹配
```

也可以在 package.json 的 script 中配置如下这些运行脚本，这样直接在根目录下执行 `npm run lint`便能检查代码是否规范。另外，运行`npm run lint-fix`可以自动修复不符合 eslint 规范的代码（只有官网上标有扳手图标的规则才会进行修复）。

```json
"lint": "eslint --ext .js --ext .jsx --ext .vue src/",
"lint-fix": "eslint --fix --ext .js --ext .jsx --ext .vue src/"
```

### 2. 插件安装
输入命令才能检查代码是否规范的操作略显繁琐，我们可以在 vscode 上安装 eslint 插件来简化步骤。这个插件会自动检查你的代码是否符合规范，并且会在编辑器中标识出来哪里不符合，底下终端处还会罗列出问题列表。

安装完成后，打开 vscode 的 setting.json 配置文件，增加以下配置后重启 vscode，就能实现保存时自动按照规范修正代码。
```json
 // 配置保存时按照 eslint 文件的规则来处理一下代码
"editor.codeActionsOnSave": {
    "source.fixAll": true,
    "eslint.autoFixOnSave" : true,
  },
```

## 二、prettier
eslint 规范的是代码偏向语法层面上的风格。比如禁用 debugger，禁用稀疏数组等。

而有时候，我们写代码删删改改，或者是个人的代码习惯，就有可能出现这样的代码。也就是说，prettier 规范的是代码偏向于排版层面上的风格。
```vue
<script>
	import HelloWorld from './components/HelloWorld.vue';
        export default {
                name: 'App',
        components: {
    HelloWorld
  }
};
</script>
```
### 1. 项目引入
```node
npm install --save-dev --save-exact prettier
npx prettier --write src/
```

安装完成后运行 `npx prettier --write src/` 就能格式化代码。可以在根目录新建 .prettierrc 的配置文件（格式为 json），或者 .prettierrc.js（则需要 module.export）。
附：[Prettier 规则集](https://www.prettier.cn/docs/options.html)

```js
//此处的规则供参考，其中多半其实都是默认值，可以根据个人习惯改写
module.exports = {
    printWidth: 80, //单行长度
    tabWidth: 2, //缩进长度
    useTabs: false, //使用空格代替 tab 缩进
    semi: false, //句末使用分号
    singleQuote: false, //使用单引号
    arrowParens: "avoid",//单参数箭头函数参数周围使用圆括号-eg: (x) => x avoid：省略括号
    bracketSpacing: true,//在对象前后添加空格-eg: { foo: bar }
    insertPragma: false,//在已被 prettier 格式化的文件顶部加上标注
    jsxBracketSameLine: false,//多属性 html 标签的‘>’折行放置
    rangeStart: 0,
    requirePragma: false,//无需顶部注释即可格式化
    trailingComma: "none",//多行时尽可能打印尾随逗号
  };

```

到目前为止，对于 prettier，我们还无法查看哪里不符合规则，而只是通过自动修复来规范代码风格。如果要像 eslint 一样，会在代码底部出现红色波浪线提示，需要把 prettier 当成 eslint 的插件注入 eslint 中，让 eslint 来报这个错（实际上还是 vscode 的 eslint 插件实现的）。
首先需要安装 `eslint-plugin-prettier`包。

```node
npm i -D eslint-plugin-prettier
```
然后在.eslintrc.js 配置文件中添加这个配置，即通过 eslint 报 prettier 的错误：

```json
// .eslintrc.js
{
  "plugins": ["prettier"],
  "rules": {
    "prettier/prettier": "error"
  }
}
```

当项目没有配置 prettier，但需要美化代码时，可以在 setting.json 文件中配置 prettier 规则：
```json
 /*  prettier 的配置 */
  "prettier.printWidth": 80, // 超过最大值换行
  "prettier.tabWidth": 2, // 缩进字节数
  "prettier.useTabs": false, // 句尾添加分号
  "prettier.singleQuote": false, // 使用单引号代替双引号
  "prettier.proseWrap": "preserve", //  (x) => {} 箭头函数参数只有一个时是否要有小括号。avoid：省略括号
  "prettier.bracketSpacing": true, // 在对象，数组括号与文字之间加空格 "{ foo: bar }"
  "prettier.endOfLine": "auto", // 结尾是 \n \r \n\r auto
  "prettier.eslintIntegration": false, //不让 prettier 使用 eslint 的代码格式进行校验
  "prettier.htmlWhitespaceSensitivity": "ignore",
  "prettier.ignorePath": ".prettierignore", // 不使用 prettier 格式化的文件填写在项目的.prettierignore 文件中
  "prettier.jsxBracketSameLine": false, // 在 jsx 中把'>' 是否单独放一行
  "prettier.jsxSingleQuote": false, // 在 jsx 中使用单引号代替双引号
  "prettier.parser": "babylon", // 格式化的解析器，默认是 babylon
  "prettier.requireConfig": false, // Require a 'prettierconfig' to format prettier
  "prettier.stylelintIntegration": false, //不让 prettier 使用 stylelint 的代码格式进行校验
  "prettier.trailingComma": "none", // 在对象或数组最后一个元素后面是否加逗号（在 ES5 中加尾逗号）
  "prettier.tslintIntegration": false,
  "prettier.arrowParens": "avoid"
```

### 2. 插件安装
同样，我们希望保存代码时能够自动格式化代码，于是又需要安装 Prettier - Code formatter 插件。安装完成后，在 setting.json 文件中添加如下配置：

```json
 //prettier 可以格式化很多种格式，所以需要在这里对应配置下
 "[html]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[css]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[less]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[vue]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[javascript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[jsonc]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[json]": {
      "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  //这个设置在 ctrl+s 的时候，会启用默认的格式化，这里是 prettier
  "editor.formatOnSave": true

```

## 三、解决冲突
在项目中同时配置 eslint 和 prettier，并都安装了对应的 vscode 插件后，两者可能会发生冲突。因为在默认情况下，保存代码时会先进行 eslint 的格式化，然后才是 prettier 的格式化。如果 eslint 和 prettier 的规则有不同，那么最后还是会按照 prettier 的规范进行格式化，从而导致无法通过 eslint 的校验。

### 1. Prettier Eslint
第一种解决方法是安装插件 Prettier Eslint，该 vscode 插件会默认先执行 prettier，而后再执行 eslint。所以当两者有规则冲突时，会由 eslint 来覆盖掉 prettier。

### 2. 添加 prettier 规则到 eslint 中
首先需要安装解决冲突的依赖：
```node
npm i -D eslint-config-prettier
```

然后把 prettier 设置的规则添加到 extends 数组中。因为 extends 中后面的配置会默认覆盖掉前面的，所以就能让前面有冲突的 eslint 规则失效。
```json
  extends: [
    'eslint:recommended', //继承 Eslint 中推荐的（打钩的）规则项 http://eslint.cn/docs/rules/
    'plugin:vue/essential', // 此项是用来配置 vue.js 风格
    'prettier',//把 prettier 中设置的规则添加进来，让它覆盖上面设置的规则。这样就不会和上面的规则冲突了
  ],
```

> 参考资料：
> [eslint+prettier+husky 的配置说明](https://blog.csdn.net/weixin_42349568/article/details/121505460)



