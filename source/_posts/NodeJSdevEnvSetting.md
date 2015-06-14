---
title: NodeJS开发环境配置
tags: [nodejs, dev]
categories: [Tech]
date: 2015-06-13 15:09:27
---


# NodeJS开发环境配置

## 安装nvm、node和npm

nvm是nodejs的多版本管理利器，node是nodejs的解释器，npm是nodejs的包管理工具。

```bash
# ubuntu安装之后会自动添加配置到~/.profile，可以直接cut到自己喜欢的比如~/.bashrc
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.25.4/install.sh | bash
$ nvm install 0.12.4
$ nvm use 0.12.4
$ nvm alias default 0.12.4
$ node --version
```
由于网速问题，国内可以使用[taobao的npm镜像](https://npm.taobao.org/)：

```bash
$ npm install -g cnpm --registry=https://registry.npm.taobao.org
$ cnpm install [package name]
```

## Build工具：grunt/gulp, bower, yeoman

自动编译、依赖管理、自动化测试、打包发布、项目模板工具、文档自动生成…… ，这些基本上属于每个项目（不论语言差异）构建的标配，javascript目前也配齐了，于是乎开发效率直线提升:)

* grunt是javascript的自动化构建工具，类似于Java的gradle/maven/ant，Python的tox，C/C++的make和scons等。
* gulp也是构建工具，它采用类似jQuery的流式配置简化了任务的编写。
* bower是一个依赖管理的工具，可以自动化安装bootstrap、angulajs、jquery等包，解决他们之间的依赖关系。
* yeoman是一个生成项目框架scaffolding工具，遵循Convention over Configuration，用过RoR或者Django的应该都知道快速开发形式.d可以配合generator-webapp、generator-angular等模板快速生成项目结构。

### 安装上述工具到本地：

```bash
$ cnpm install -g yo bower grunt-cli gulp
# 安装yo项目模板
$ cnpm install -g generator-angular
```

### 生成一个HelloWorld

```bash
$ mkdir -p ~/dev/nodejs/helloworld && cd $_
$ yo angular
$ grunt serve
$ grunt test
$ grunt
```
这样就把一个前段项目的框架生成完毕了，可以进入开发阶段。

## 编辑器：Atom

使用chrome和nodejs开发的Atom，几个月前看还是离sublimetext挺远，现在看几乎快要完全超越！——**ubuntu下的中文算是个麻烦事，自定义以后还凑合**。

* 安装可以直接从[官网](https://atom.io/)开始。
* 开源字体从[文泉驿](http://wenq.org/)开始。

参考配置：

```css
@font-family: "DejaVu Sans Mono", "WenQuanYi Zen Hei";

.tree-view, .title, .current-path, .tooltip {
  font-family: @font-family;
}

.terminal {
  font-family: @font-family !important;
  div {
    white-space: nowrap;
  }
}

// style the background and foreground colors on the atom-text-editor-element
// itself
atom-text-editor {
    font-family: @font-family;
}

// To style other content in the text editor's shadow DOM, use the ::shadow
// expression
atom-text-editor::shadow .cursor {
    font-family: @font-family;
}

.markdown-preview {
    font-family: @font-family;
    atom-text-editor {
        font-family: @font-family;
    }
}
```
Atom也有大量的插件可以使用，比如把hexo集成进来，可以少开个term了：）

* VIM开启了编辑器的多模式状态，让敲键盘更尽兴；
* SublimeText方便了编辑器的DIY，性能也很好；
* Atom让Web开发更彻底，它本身就是个基于浏览器内核的工具；此外，它来自于Github:)

## 参考
1. [Atom Offical][node1]
1. [文泉驿官网][node2]
1. [Taobao NPM][node3]
1. [Grunt Offical][node4]
1. [Bower Offical][node5]
1. [Yeoman Offical][node6]

[node1]:https://atom.io/ "Atom Offical site"
[node2]:http://wenq.org/ "文泉驿官网"
[node3]:https://npm.taobao.org/ "Taobao NPM"
[node4]:http://gruntjs.com/ "Grunt Offical"
[node5]:http://bower.io/ "Bower Offical"
[node6]:http://yeoman.io/ "Yeoman Offical"
