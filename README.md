## 概述

这是我的个人博客站点仓库，使用静态站点生成器Jekyll进行构建，并使用GitHub Pages进行托管，主题为Chirpy。博客站点主要用于记录我的学习笔记、分享个人经验和心得，以及与读者交流，我会不定期地更新站点，并保持良好的内容质量。


## 如何在本地运行

1. 克隆博客站点仓库

```bash
$ git clone https://github.com/SimenCTan/simenctan.github.io.git
```

2. 安装Jekyll

具体安装方法请参考[Jekyll官网](https://jekyllrb.com/docs/installation/)。

3. 进入博客站点目录

```bash
$ cd simenctan.github.io
```

4. 运行本地服务器

```bash
bundle lock --update #update gemfile.lock
bundle install # install package
bundle exec jekyll s
```

5. 打开浏览器
