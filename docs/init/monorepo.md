# monorepo

## 什么是 monorepo

monorepo 指的是一中代码项目结构的组织方式，mono 是指单个，repo 是指仓库，总的来说就是单个仓库，顾名思义就是把所有相关的项目放到一个仓库中进行管理

**优点**：

- 公共基础设置，不用重复配置
- 有依赖的项目之间调试开发非常方便
- 第三方库的版本管理更加简单

## 从 Vue3 源码入门级理解 pnpm 的 monorepo

vue3 在模块的拆分和设计上非常合理，模块之间的耦合度非常低，很多模块可以独立安装使用，而不需要完整的以来 vue3 运行时。这种实现除了代码层面的设计之外，最重要的还是 monorepo 对项目代码进行组织管理

### vue3 中的 monorepo

vue3 目前使用的是 pnpm 的 monorepo。
在项目根目录中新建 pnpm-workspace.yaml 文件，并声明对应的工作区就可以了

```yaml
packages:
  - "packages/*"
```

表示 packege/\*这个目录下的所有文件为 workspace 的内容
![Alt text](image-1.png)
从上图可以看到 Vue3 源码整体是通过 monorepo 方式进行管理，并根据功能的不同在 packages 目录下进行划分不同的模块目录。我们可以看到每一个目录下面都一个 package.json 文件，代表每一个目录都是一个 npm 包，每个包有各自的 API、类型定义和测试模块以及 Readme 文档。这样就可以将模块拆分得更细的颗粒度，职责划分也更明确。

由于所有项目都放在一个仓库中，代码逻辑服用非常方便，如果有依赖的代码发生变动，那么用到这个依赖的项目就会立马感知到。这又是怎么做到的呢？普通项目可以通过相对路径进行引用，但我们这里的设想是每一个包都是独立，如果通过相对路径进行引用，那么就会非常耦合。所有，我们可以通过，workspace 协议进行模块之间的相互依赖，达到解耦的目的

### Workspace 协议，模块之间的相互依赖、

vue3 中，响应式方面的功能都是使用`@vue/reactivity`包的,这些包都可以在 npmjs 中下载。当本地的时候，只需要在项目的`package.json`进行下面设置

```js
{
  '@vue/reactivity:"workspace:*"',
  '@vue/runtime-core:"workspace:*"',
  '@vue/runtime-dom:"workspace:*"',

}

```

本地 workspace 包只要进行`workspace`协议，这样就依赖本地的包了，而需要从 npmjs 进行安装下载。还有一个好处就是子包相互引用代码时，使用`workspace:*`的写法来链接子包，而不是具体的版本号，可以防止多人协作时因为修改版本的遗漏而发生冲突。

通过 monorepo 方式进行管理的项目，每一个模块都可以说是一个独立的项目，同时和其他项目复用一套标准的工具和规范，无需切换开发环境。 比如我今天只修改了 reactivity 项目，那么我就可以只对 reactivity 项目进行打包处理。

`pnpm run build` 是打包所有模块，在后面加模块名称则是具体打包所加的模块名称的模块。
单独打包 reactivity 模块：
`pnpm run build reactivity`
这样就可以单独把打包出来的内容单独发布

而对 reactivity 项目进行打包处理的持续集成(CI)流程、构建和发布流程都是和其他项目共用的有的基建流程，即便将来有新的项目接入，依然可以复用现在的基建逻辑代码，这样维护和开发成本就大大降低了。

### workspace 包的版本

workspace:_后面的 _ 表示任意版本，除了 \* 还有其他：~ 、^ 符号。
当 workspace 包打包发布时，将会动态替换这些 workspace: 依赖。
假设我们上面的包的版本都是 1.0.0 ，它们的 workspace 配置如下：

```js
{
  "dependencies": {
    "@vue/reactivity": "workspace:*",
    "@vue/runtime-core": "workspace:~",
    "@vue/runtime-dom": "workspace:^"
  },
}

```

将来发包的时候，使用相关的发包工具，比如使用 changesets 来发包，该工具会帮你自动升级版本、产生 CHANGELOG 、自动替换 workspace:\* 为具体版本、自动保持版本一致性。
比如上面的代码将来发布的时候将会被转化为：

```js
{
  "dependencies": {
    "@vue/reactivity": "workspace:1.0.0",
    "@vue/runtime-core": "workspace:~1.0.0",
    "@vue/runtime-dom": "workspace:^1.0.0"
  },
}
```

[Monorepo - 优劣、踩坑、选型](https://juejin.cn/post/7215886869199896637)
[为什么越来越多的项目选择 Monorepo？](https://juejin.cn/post/7207743145999368229)

## 以 pnpm 进行 monorepo 环境的搭建

### workspace 模式

pnpm 支持 monorepo 模式的工作机制叫做 workspace 工作空间，
他要求在代码仓库的根目录下有`pnpm-workspace.yaml`文件指定那些目录作为独立的工作空间，这个工作空间可以理解为一个子模块或者 npm 包。

```
📦my-project
 ┣ 📂a
 ┃ ┗ 📜package.json
 ┣ 📂b
 ┃ ┗ 📜package.json
 ┣ 📂c
 ┃ ┣ 📂c-1
 ┃ ┃ ┗ 📜package.json
 ┃ ┣ 📂c-2
 ┃ ┃ ┗ 📜package.json
 ┃ ┗ 📂c-3
 ┃   ┗ 📜package.json
 ┣ 📜package.json
 ┣ 📜pnpm-workspace.yaml

```

```js
//pnpm-workspace.yaml
packages:
  - a
  - b
  - c/*
```

pnpm 并不是通过目录名称，而是通过目录下的 package.json 文件的 name 字段来识别仓库的包和模块

### 中枢管理操作

在 workspace 模式下，代码仓库根目录不会作为一个子模块或者 npm 包，而是**主要作为一个管理中枢，执行一些全局操作，安装一些共有的依赖**

### 子包管理操作

在 workspace 模式下，pnpm 通过`--filter`选项过滤子模块，实现对各个工作空间进行精细化操作的目的

### 实战环节，初始化 monorepo 工程

#### 创建项目文件夹并进行初始化

```bash
mkdir monorepo project
cd  monorepo project
pnpm init
```

#### 创建 pnpm-workspace.yaml 文件

`pnpm-workspace.yaml`这个文件的存在本身，会让 pnpm 要使用 monorepo 的模式管理这个项目，他的内容告诉 pnpm 哪些目录将被划分为独立的模块，这些所谓的独立模块被包管理器叫做 workspace(工作空间)。我们在这个文件中写入以下内容。

```yaml
packages:
  # 根目录下的 docs 是一个独立的文档应用，应该被划分为一个模块
  - docs
  # packages 目录下的每一个目录都作为一个独立的模块
  - packages/*
```

接下来就是建立这些工作空间，并将根目录下的`package.json`文件复制到每个工作空间中
接下来就是设置 package.json

#### 设置 package.json

明确每一个模块的属性，设置他们的 package.json 文件

##### 根目录的 package.json

```json
// openx-ui/package.json
{
  "name": "openx-ui",
  "private": true,
  "scripts": {
    // 定义脚本
    "hello": "echo 'hello world'"
  },
  "devDependencies": {
    // 定义各个模块的公共开发依赖
  }
}
```

- `private:true`:根目录在 monorepo 模式下只是一个管理中枢，它不会被发布为 npm 包 -` devDependencies`：所有模块都会有一些公共的开发依赖，例如构建工具、TypeScript、Vue、代码规范等，将公共开发依赖安装在根目录可以大幅减少子模块的依赖声明。

##### 组件包的 package.json

```json
{
  //标识信息
  "name": "@summary-project/cli",
  "version": "1.0.0",

  // 基本信息
  "description": "",
  "keywords": ["vue", "ui", "component library"],
  "author": "aky",
  "license": "MIT",
  "homepage": "git+https://github.com/xxx.git/xxx/README.md"
  "repository": {
    "type": "git",
    "url": "git+https://github.com/xxx.git"
  },
  "bugs": {
    "url" :"git+https://github.com/xxx.git/xxx/issue"
  }

  // 定义脚本，由于还没有集成实际的构建流程，这里先以打印命令代替
  "scripts": {
    "build": "echo build",
    "test": "echo test"
  },

  // 入口文件由于没有实际产物，先设置为空字符串
  "main": "",
  "module": "",
  "types": "",
  "exports": {
    ".": {
      "require": "",
      "module": "",
      "types": ""
    }
  },

  // 发布信息
  "files": [
    "dist",
    "README.md"
  ],
  // "publishConfig": {},

  // 依赖信息
  "peerDependencies": {
    "vue": ">=3.0.0"
  },
  "dependencies": {},
  "devDependencies": {}
}
```

##### 项目文档的 package.json

```json
// openx-ui/docs/package.json
{
  "name": "@openxui/docs",
  "private": true,
  "scripts": {
    // 定义脚本，由于还没有集成实际的构建流程，这里先以打印命令代替
    "dev": "echo dev",
    "build": "echo build"
  },
  "dependencies": {
    // 安装文档特有依赖
  },
  "devDependencies": {
    // 安装文档特有依赖
  }
}
```

至此,我们的 monorepo 项目雏形已经建立完毕.


### monorepo 下集成 Vite

#### .npmrc 文件

安装依赖之前，先熟悉下.npmrc 文件，在根目录下建立.npmrc 文件
这个.npmrc 文件相当于项目级的 npm 配置

```ini
registry=https://registry.npm.taobao.org

```

上面的配置相当于切换 npm 镜像源为 https://registry.npm.taobao.org，只不过配置只在当前项目目录下生效，优先级高于用户设置的本地配置。

这个效果相当于`npm set config registry https://registry.npm.taobao.org`.

将一些必要的配置放在这个项目级的 .npmrc 文件中，并且将这个文件提交到代码仓，这样就可以使后续贡献的其他用户免去许多环境配置的麻烦，尤其是在公司的内网环境下，各式各样的 npm 私仓和代理配置让很多新人头疼。

当然，如果你是开源项目，就不是很推荐你在里面做网络环境相关的配置了，因为每个贡献者的网络环境是多样化的，这时项目级的 .npmrc 就只适合放一些与包管理相关的配置。

#### 安装公共项目构建依赖

接下来，就是在根目录下安装所需的构建工具：`Vite`和`Typescript`

```bash
pnpm i -wD vite typescript
```

因为每个包都需要用到 Vite 和 TypeScript 进行构建，公共开发依赖统一安装在根目录下，是可以被各个子包正常使用
由于我们要构建的是 Vue 项目，Vue 推荐的组件开发范式单文件组件 SFC 并不是原生的 Web 开发语法，所以需要一个编译为原生 js 的过程，所以我们需要引入相关的 Vite 插件`@vitejs/plugin-vue`，这个插件继承了 Vue 编译器的能力，是的构建工具可以理解 vue sfc

```bash
pnpm i -wD @vitejs/plugin-vue

```

另外需要注意，vue 应该被安装到根目录下的 dependencies，因为几乎所有子包的 peerDependencies 中都具有 vue(peerDependencies)，我们结合 pnpm 的`resolve-peers-from-workspace-root` 机制，可以统一所有子包中 vue 的版本

安装 Vue

```bash
pnpm i -wS vue
```

安装 css 预处理

```bash
pnpm i -wD sass
```

#### Vite 集成

为了成功集成 Vite，让我们的集成库构建出产物，这里我们需要完成三个步骤：

- 编写构建目标源码，目前的重点是工程化而非组件库的开发，代码层面能够体现构建要点的 demo
- 准备 vite.config 配置文件
- 在 package.json 中设置构建脚本

在集成 Vite 过程中，我们需要进行非常多的扩展

1. 对于 package 目录下的每一个组件包，我们制定更细致的源码组织规则

- 各种配置文件，如`package.json`、`vite.config.ts(js)`都放在模块根目录下
- src 目录下存放源码，其中`src/index.ts(js)`作为模块的总出口，所有需要暴露给外部供其他模块使用的方法、对象都要在这里声明导出。
- dist 目录作为产物的输出目录

2. 如果是组件库，可以在 packages 目录下新建统一出口包。如 element-plus 主包负责各个子包，并统一导出其中内容
3. 测试模块，可以根目录下新建 demo 模块，用来进行单元测试？
4. typescript，tsconfig 的测试

因为我们规定了每个模块的 dist 都作为产物输出目录，而输出产物是不需要入仓的(clone 代码后执行构建命令就能生成)，所以要注意在根目录的 .gitignore 中添加产物目录 dist：

```diff
# .gitignore
node_modules
+dist
```

#### 公共方法代码预备
我们安排 @summary-project/shared 作为公具方法包，将成为所有其他模块的依赖项。
