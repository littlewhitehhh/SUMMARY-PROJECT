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

## 以 pnpm 进行 monorepo 环境的搭建
