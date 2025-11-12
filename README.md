# algo.tech - 开源博客

## 一、概述

我一直认为，自己是一个热爱技术，走在技术成长之路上的人。在这条路上，我遇到过很多启发和见解，它们就像来自四面八方的光，给人以启迪。

博客网址：<https://aoco.tech>

曾用域名如下

|      域名      |         时间          |  状态  |
| -------------- | --------------------- | ------ |
| blog.jkdev.cn  | 2017-2022             | 已废弃 |
| webcoding.tech | 2022-2028.12.18       | 已废弃 |
| aoco.tech      | 2028.12.18-2025.11.12 | 已废弃 |
| aoco.tech      | 2025.11.12-今天       |        |

## 二、本地运行

### 2.1 安装基础环境

已`debian 12`为例，首先安装 ruby，如下命令

```shell
sudo apt install ruby-full
```

设置 GEM_HOME，如下命令

```shell
echo 'export GEM_HOME="$HOME/.gem"' >> ~/.bashrc
echo 'export PATH="$HOME/.gem/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

安装 bundler 和 jekyll，如下命令

```shell
gem install bundler
gem install jekyll
```

如果希望换成国内源，可以执行如下命令换成中科大的源

```shell
# 查看源
gem sources
# 删除源
gem sources --remove https://rubygems.org/
# 添加源
gem sources -a https://mirrors.ustc.edu.cn/rubygems/
```

同时，在项目的 Gemfile 文件中，将 source 改为中科大的gem源地址。

### 2.2 安装依赖

在当前项目下执行如下命令进行依赖安装

```shell
bundle install
```

### 2.3 运行博客

在当前项目下执行如下命令进行博客运行

```shell
bundle exec jekyll serve
```

在运行过程中，可能会出现相关 gem 的依赖问题，可以按照提示进行安装后再次执行。如缺失 `tzinfo` 可以执行如下命令

```shell
gem install tzinfo
```

运行成功后，访问 <http://127.0.0.1:4000> 即可访问博客。
