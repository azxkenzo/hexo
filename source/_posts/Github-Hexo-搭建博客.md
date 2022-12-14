---
title: Github + Hexo 搭建博客
date: 2022-05-26 07:56:14
tags: 
- Hexo
- Github
---

## 一、准备工作

### 1. 安装 Node.js
Node.js 版本需不低于 10.13，建议使用 Node.js 12.0 及以上版本
点此下载[安装程序](https://nodejs.org/en/download/)

### 2. 安装 Git

### 3. 安装 Hexo
在 Node.js 安装好后，使用 npm 安装 Hexo
``` bash
$ npm install -g hexo-cli
```
注意：在 macOS 上，该命令需要在 root 环境下运行


## 二、建站
安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
``` bash
$ hexo init <folder>
$ cd <folder>
$ npm install
```

新建完成后，指定文件夹的目录如下：
```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

以下是目录下各个文件的描述：
#### _config.yml
网站的 配置 信息，可以在此配置大部分的参数。

#### package.json
应用程序的信息。EJS, Stylus 和 Markdown renderer 已默认安装，可以自由移除。

#### scaffolds
模版 文件夹。当新建文章时，Hexo 会根据 scaffold 来建立文件。

Hexo的模板是指在新建的文章文件中默认填充的内容。例如，如果修改scaffold/post.md中的Front-matter内容，那么每次新建一篇文章时都会包含这个修改。

#### source
资源文件夹是存放用户资源的地方。除 _posts 文件夹之外，开头命名为 _ (下划线)的文件 / 文件夹和隐藏的文件将会被忽略。Markdown 和 HTML 文件会被解析并放到 public 文件夹，而其他文件会被拷贝过去。

#### themes
主题 文件夹。Hexo 会根据主题来生成静态页面。



## 三、网站配置
可以在 _config.yml 中修改大部分的配置。

### 网站
title 	网站标题

subtitle 	网站副标题

description 	网站描述

keywords 	网站的关键词。支持多个关键词。

author 	您的名字

language 	网站使用的语言。对于简体中文用户来说，使用不同的主题可能需要设置成不同的值，请参考你的主题的文档自行设置，常见的有 zh-Hans和 zh-CN。

timezone 	网站时区。Hexo 默认使用您电脑的时区。请参考 时区列表 进行设置，如 America/New_York, Japan, 和 UTC 。一般的，对于中国大陆地区可以使用 Asia/Shanghai。




## 四、命令
### init
``` bash
$ hexo init [folder]
```
新建一个网站。如果没有设置 folder ，Hexo 默认在目前的文件夹建立网站。

本命令相当于执行了以下几步：
1. Git clone hexo-starter 和 hexo-theme-landscape 主题到当前目录或指定目录。
2. 使用 Yarn 1、pnpm 或 npm 包管理器下载依赖（如有已安装多个，则列在前面的优先）。npm 默认随 Node.js 安装。


### new
``` bash
$ hexo new [layout] <title>
```
新建一篇文章。如果没有设置 layout 的话，默认使用 _config.yml 中的 default_layout 参数代替。如果标题包含空格的话，请使用引号括起来。

参数：

-p, --path 	自定义新文章的路径

-r, --replace 	如果存在同名文章，将其替换

-s, --slug 	文章的 Slug，作为新文章的文件名和发布后的 URL

默认情况下，Hexo 会使用文章的标题来决定文章文件的路径。对于独立页面来说，Hexo 会创建一个以标题为名字的目录，并在目录中放置一个 index.md 文件。你可以使用 --path 参数来覆盖上述行为、自行决定文件的目录：
``` bash
$ hexo new page --path about/me "About me"
```
以上命令会创建一个 source/about/me.md 文件，同时 Front Matter 中的 title 为 "About me"

注意！title 是必须指定的！如果你这么做并不能达到你的目的：
``` bash
$ hexo new page --path about/me
```
此时 Hexo 会创建 source/_posts/about/me.md，同时 me.md 的 Front Matter 中的 title 为 "page"。这是因为在上述命令中，hexo-cli 将 page 视为指定文章的标题、并采用默认的 layout。


### generate
``` bash
$ hexo generate
```
生成静态文件

-d, --deploy 	文件生成后立即部署网站

-w, --watch 	监视文件变动

-b, --bail 	生成过程中如果发生任何未处理的异常则抛出异常

-f, --force 	强制重新生成文件。
Hexo 引入了差分机制，如果 public 目录存在，那么 hexo g 只会重新生成改动的文件。
使用该参数的效果接近 hexo clean && hexo generate

-c, --concurrency 	最大同时生成文件的数量，默认无限制

该命令可以简写为：
``` bash
$ hexo g
```



### publish
``` bash
$ hexo publish [layout] <filename>
```
发表草稿



### server
``` bash
$ hexo server
```
启动服务器

-p, --port 	重设端口

-s, --static 	只使用静态文件

-l, --log 	启动日记记录，使用覆盖记录格式



### deploy
``` bash
$ hexo deploy
```
部署网站。

-g, --generate 	部署之前预先生成静态文件

该命令可以简写为：
``` bash
$ hexo d
```



### render
``` bash
$ hexo render <file1> [file2] ...
```
渲染文件。




### clean
``` bash
$ hexo clean
```
清除缓存文件 (db.json) 和已生成的静态文件 (public)。

在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。



### list
``` bash
$ hexo list <type>
```
列出网站资料。



### version
``` bash
$ hexo version
```



## 五、写作
可以执行下列命令来创建一篇新文章或者新的页面。
``` bash
$ hexo new [layout] <title>
```

### 布局（Layout）
Hexo 有三种默认布局：post、page 和 draft。在创建这三种不同类型的文件时，它们将会被保存到不同的路径；而您自定义的其他布局和 post 相同，都将储存到 source/_posts 文件夹。

布局 	路径

post 	source/_posts

page 	source

draft 	source/_drafts

If you don’t want an article (post/page) to be processed with a theme, set layout: false in its front-matter. Refer to this section for more details.


### 文件名称
Hexo 默认以标题做为文件名称，但您可编辑 new_post_name 参数来改变默认的文件名称，举例来说，设为 :year-:month-:day-:title.md 可让您更方便的通过日期来管理文章。



### 草稿
刚刚提到了 Hexo 的一种特殊布局：draft，这种布局在建立时会被保存到 source/_drafts 文件夹，可通过 publish 命令将草稿移动到 source/_posts 文件夹，该命令的使用方式与 new 十分类似，也可在命令中指定 layout 来指定布局。
``` bash
$ hexo publish [layout] <title>
```


### 模版（Scaffold）
在新建文章时，Hexo 会根据 scaffolds 文件夹内相对应的文件来建立文件，例如：
``` bash
$ hexo new photo "My Gallery"
```
在执行这行指令时，Hexo 会尝试在 scaffolds 文件夹中寻找 photo.md，并根据其内容建立文章，以下是可以在模版中使用的变量：

layout 	布局

title 	标题

date 	文件建立日期




## 六、Front-matter
Front-matter 是文件最上方以 --- 分隔的区域，用于指定个别文件的变量，举例来说：
```
---
title: Hello World
date: 2013/7/13 20:46:25
---
```
以下是预先定义的参数，可在模板中使用这些参数值并加以利用。

layout 	布局 	config.default_layout

title 	标题 	文章的文件名

date 	建立日期 	文件建立日期

updated 	更新日期 	文件更新日期

comments 	开启文章的评论功能 	true

tags 	标签（不适用于分页） 

categories 	分类（不适用于分页） 	

permalink 	覆盖文章网址 	

excerpt 	Page excerpt in plain text. Use this plugin to format the text 	

disableNunjucks 	Disable rendering of Nunjucks tag and tag plugins when enabled 	

lang 	Set the language to override auto-detection 	Inherited from _config.yml


### 布局
The default layout is post, in accordance to the value of default_layout setting in _config.yml. When the layout is disabled (layout: false) in an article, it will not be processed with a theme. However, it will still be rendered by any available renderer: if an article is written in Markdown and a Markdown renderer (like the default hexo-renderer-marked) is installed, it will be rendered to HTML.

Tag plugins are always processed regardless of layout, unless disabled by the disableNunjucks setting or renderer.


### 分类和标签
只有文章支持分类和标签，您可以在 Front-matter 中设置。在其他系统中，分类和标签听起来很接近，但是在 Hexo 中两者有着明显的差别：分类具有顺序性和层次性，也就是说 Foo, Bar 不等于 Bar, Foo；而标签没有顺序和层次。



## 七、部署
使用 Travis CI 将 Hexo 博客部署到 GitHub Pages 上。Travis CI 对于开源 repository 是免费的，但是这意味着你的站点文件将会是公开的。
1. 新建一个 repository。如果你希望你的站点能通过域名 <你的 GitHub 用户名>.github.io 访问，你的 repository 应该直接命名为 <你的 GitHub 用户名>.github.io。
2. 将你的 Hexo 站点文件夹推送到 repository 中。默认情况下 public 目录将不会（也不应该）被推送到 repository 中，你应该检查 .gitignore 文件中是否包含 public 一行，如果没有请加上。
3. 将 Travis CI 添加到你的 GitHub 账户中。
4. 前往 GitHub 的 Applications settings，配置 Travis CI 权限，使其能够访问你的 repository。
5. 你应该会被重定向到 Travis CI 的页面。如果没有，请 手动前往。
6. 在浏览器内新建一个标签页，前往 GitHub 新建 Personal Access Token，只勾选 repo 的权限并生成一个新的 Token。Token 生成后请复制并保存好。
7. 回到 Travis CI，前往你的 repository 的设置页面，在 Environment Variables 下新建一个环境变量，Name 为 GH_TOKEN，Value 为刚才你在 GitHub 生成的 Token。确保 DISPLAY VALUE IN BUILD LOG 保持 不被勾选 避免你的 Token 泄漏。点击 Add 保存。
8. 在你的 Hexo 站点文件夹中新建一个 .travis.yml 文件：
9. 将 .travis.yml 推送到 repository 中。Travis CI 应该会自动开始运行，并将生成的文件推送到同一 repository 下的 gh-pages 分支下
10. 在 GitHub 中前往你的 repository 的设置页面，修改 GitHub Pages 的部署分支为 gh-pages。
11. 前往 https://<你的 GitHub 用户名>.github.io 查看你的站点是否可以访问。这可能需要一些时间。


### 一键部署
Hexo 提供了快速方便的一键部署功能，让您只需一条命令就能将网站部署到服务器上。
``` bash
$ hexo deploy
```
在开始之前，必须先在 _config.yml 中修改参数，一个正确的部署配置中至少要有 type 参数，例如：
```
deploy:
  type: git
```
#### Git
1 安装 hexo-deployer-git
``` bash
$ npm install hexo-deployer-git --save
```
2 修改配置
```
deploy:
  type: git
  repo: <repository url> #https://bitbucket.org/JohnSmith/johnsmith.bitbucket.io
  branch: [branch]
  message: [message]
```
3 生成站点文件并推送至远程库。执行 `hexo clean && hexo deploy`。

· You will be prompted with username and password of the target repository, unless you authenticate with a token or ssh key. 

· hexo-deployer-git does not store your username and password. Use [git-credential-cache](https://git-scm.com/docs/git-credential-cache) to store them temporarily.

4 登入 Github，请在库设置（Repository Settings）中将默认分支设置为_config.yml配置中的分支名称。稍等片刻，您的站点就会显示在您的Github Pages中。


### 这一切是如何发生的？
当执行 hexo deploy 时，Hexo 会将 public 目录中的文件和目录推送至 _config.yml 中指定的远端仓库和分支中，并且完全覆盖该分支下的已有内容。


Support for password authentication was removed on August 13, 2021. Please use a personal access token instead.



