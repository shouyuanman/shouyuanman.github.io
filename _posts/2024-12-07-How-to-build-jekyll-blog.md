---
title: 怎么在 GitHub 上用 jekyll 搭建自己的博客
date: 2024-12-07 20:40:10 +0800
categories: [环境部署手顺, jekyll]
tags: [jekyll, 博客建站]
music-id: 393685
---

## 从 jekyll 和 jekyll-now 开始

`jekyll` 是一个基于 `Ruby` 开发的开源静态网站生成器，支持 `Markdown` 和 `HTML` 两种文件类型，其中 `HTML` 使用了 `Liquid` 模板语言。它使用 `Ruby` 引擎将用 `Markdown` 编写的文章转换成静态的 `HTML` 文件，生成的网站可以方便地部署到各种网络服务器上。

[**jekyll-now**](https://github.com/barryclark/jekyll-now)是一个 `jekyll` 脚手架，可以基于它轻松地使用 `GitHub Pages` 服务创建 `jekyll` 博客，使用方式可以参考仓库`ReadMe`。

![Desktop View](/assets/img/20241207/jekyll_readme.png){: width="250" height="150" }
_jekyll readme_

## GitHub Pages
使用[**GitHub Pages**](https://pages.github.com/)这个免费的静态网站托管服务，可以直接从`GitHub`仓库托管网站。
```markdown
GitHub Pages 服务有如下特点，
- 支持 HTML、CSS 和 JavaScript 文件，并通过可选的构建过程运行文件，然后发布网站；
- GitHub Pages 适用于个人、组织和项目的网站托管，支持公开和私有仓库，但公开仓库才能访问‌；
- 提供 username.github.io 格式的免费域名，也支持自定义域名；
- 与 jekyll 集成，允许用户轻松创建博客和网站，同时自动处理页面生成和部署；
- 自动为站点提供HTTPS加密‌。
```

具体可以参考[**关于 GitHub Pages**](https://docs.github.com/zh/enterprise-server@3.8/pages/getting-started-with-github-pages/about-github-pages)帮助文档。

![Desktop View](/assets/img/20241207/github_pages_setting.png){: width="400" height="200" }
_github pages setting_

## jekyll 主题
`jekyll主题官网`(http://jekyllthemes.org/)上提供很多特色主题，可以选择一套自己喜欢的主题来定制网站，本站使用的主题是`jekyll-theme-chirpy`(http://jekyllthemes.org/themes/jekyll-theme-chirpy/)。

## 建站步骤
- 选定 `chirpy` 主题，在 `Github` 上 `fork` 主题仓库，仓库权限需要设置成 `public`；

![Desktop View](/assets/img/20241207/chirpy_fork.png){: width="300" height="100" }
_fork chirpy_

- 这里使用 `GitHub Pages` 服务提供的 `username.github.io` 免费域名，需要把 `fork` 的主题仓库名字改成 `username.github.io`；

![Desktop View](/assets/img/20241207/user_github_io.png){: width="200" height="100" }
_user github io_

- 浏览器输入域名简单看下效果，现在可以根据自己的需求定制样式了。

![Desktop View](/assets/img/20241207/blog_init_home_page.png){: width="200" height="100" }
_blog init home page_

## 搭建本地环境
1. **`clone` 仓库到本地，在自己的主机上安装正确的运行环境，方便后期本地调整博客实时看效果**

    - 安装一些必要的软件，比如 [`gcc`](https://gcc.gnu.org/install/)、[`make`](https://www.gnu.org/software/make/)、`VSCode`、`Git`，详细安装过程网上有很多资料，这里不再详述；

    - 安装 `Ruby` 和 `jekyll` 运行环境（这里仅介绍 Windows 上的安装步骤，其他系统平台安装细节具体参见 [**jekyll官网文档**](https://www.jekyll.com.cn/docs/)）；

        - 下载安装[**Ruby**](https://www.rubyinstaller.cn/)，Ruby 安装结束时会弹出一个窗口让你选择 3 个选项其中一个来安装，一般直接选 3 就行，这是安装具有本机扩展的 gem 必需的，具体介绍可以参考 `RubyInstaller` 文档。如果这一步没安装成功，可以使用 `ridk install` 命令重新安装；
            ```terminal
            $ ruby -v
            ```

        - 下载安装[**RubyGems**](https://rubygems.org/pages/download)；
            ```terminal
            // 下载 zip 文件到本地解压，cd到解压目录，执行下面命令进行安装
            $ ruby setup.rb
            $ gem -v
            ```

    - 安装 `jekyll`、`Bundler`、`github-pages`；
        ```terminal
        $ gem install jekyll
        $ gem install bundler
        $ gem install github-pages
        $ jekyll -v
        ```

    > _Ruby_ 中，_Gem_([**RubyGems**](https://rubygems.org/)) 和 [**Bundler**](https://github.com/capistrano/bundler)（[**官网**](https://bundler.io/)） 都是用于管理和处理项目依赖的工具。
    ```markdown
    Gem 和 Bundler 通常会结合使用；
    Gem 用于安装和管理单个 Gem 包（包管理器）；
    Bundler 用于管理整个项目的依赖关系（依赖管理工具），确保项目中的 gem 版本一致‌，避免版本冲突和依赖不一致的问题。
    ```
    {: .prompt-tip }

    - 到这里，运行环境配置好了，可以在本地运行了。

2. **初始化博客配置**
    > 初始化博客配置<br/>参考[chirpy starter - getting-started](https://chirpy.cotes.page/posts/getting-started/)
    {: .prompt-tip }

3. **博客内容编写好后，本机挂起博客网站随时查看调整内容**
    ```terminal
    $ bundle install
    $ bundle exec jekyll serve -P 5555 --watch
    ```

    > ‌bundle install 命令的正确使用姿势
    ```markdown
bundle install 命令在 Ruby on Rails 项目中用于安装和更新项目的依赖项，它会检查 Gemfile 文件中列出的所有依赖项，将它们安装到项目本地环境中。这个过程确保了项目中的所有依赖项都是最新的，并且可以避免因为依赖项版本不一致而导致的问题‌。
在使用 bundle install 命令之前，需要确保已经安装了 Ruby 和 RubyGems，另外需要在项目的根目录下创建一个名为 Gemfile 的文件，其中列出了项目所需的所有依赖项。最后要在包含 Gemfile 文件的项目根目录上运行。
    ```
    {: .prompt-tip }

    > jekyll gem 让 jekyll 可以通过命令行进行使用
```markdown
jekyll new PATH 
- 使用基于 gem 的默认主题在指定目录中创建一个全新的 jekyll 站点，必要的目录也会被自动创建
jekyll new PATH --blank
- 在指定的目录下创建一个全新的空的 jekyll 站点脚手架
jekyll build 或 jekyll b
- 执行一次构建，并将生成的站点输出到 `./_site` （默认） 目录
jekyll doctor
- 输出任何不推荐功能或配置方面的问题
jekyll help
- 显示帮助信息，也可以针对特定子命令显示帮助信息，例如jekyll help build
jekyll new-theme
- 创建一个新的 jekyll 主题脚手架
jekyll serve 或 jekyll
- 本地开发，源文件更改时构建站点并提供本地访问服务，--watch 表示这个本地网页是实时刷新的，当更改网页的内容时它能实时变化，不用重启和加载网页
jekyll build
- 为生产环境构建站点
jekyll clean
- 删除所有生成的文件：输出目录、元数据文件、Sass 和 jekyll 缓存
```
    {: .prompt-tip }

4. **确认博客内容符合预期后，把内容推到 `Github`仓库，`自动构建部署`成功后，在站点查看即可**

![Desktop View](/assets/img/20241207/blog_init_home_page.png){: width="800" height="400" }
_网站效果图_

## 配置评论系统
本站的评论功能通过集成`giscus`评论系统来实现。

>`jekyll`博客是静态的，并没有评论功能，一般能用`disqus`、`utterances`或`giscus`评论系统来实现。本站使用的`giscus`开源免费，它是基于[GitHub Discussions](https://docs.github.com/en/discussions)实现的。
{: .prompt-tip }

1. 在 [GitHub Marketplace](https://github.com/marketplace)中能搜索到[giscus应用](https://github.com/apps/giscus)，将其安装配置到`GitHub`，可以在[GitHub applicationis](https://github.com/settings/installations)里边查看；

    ![Desktop View](/assets/img/20241207/github_apps_page.png){: width="600" height="300" }
    _github applicationis page_

2. 在仓库`Setting`-`General`-`Features`中选中`Discussions`，开启`GitHub Discussions`功能；

    ![Desktop View](/assets/img/20241207/github_discussion_set.png){: width="600" height="300" }
    _github discussion checked_

    ![Desktop View](/assets/img/20241207/github_discussion_page.png){: width="600" height="300" }
    _github discussion page_

3. 打开[giscus配置页面](https://giscus.app/)，选择自定义配置，复制自动生成的`Enable giscus`配置；

    > giscus选项配置解析
```markdown
Page-Discussions Mapping
- 可以选择默认值 “Discussion title contains page pathname”；
- 含义：URL 为 https://xxx.github.io/posts/title 的文章将映射到标题为 /posts/title 的  Discussion，用 URL 的 pathname 部分作为 Discussion 标题。
Discussion Category
- 选择 Disussions 下边的类别名称，比如`General`，也可以自己定义。
```
    {: .prompt-tip }

    ![Desktop View](/assets/img/20241207/giscus_enable_content.png){: width="600" height="300" }
    _giscus enable content_

4. 把上面自动生成的配置放在`_config.yml`中，重新构建发布后，就能看到评论系统了。

```yaml
comments:
    # Global switch for the post-comment system. Keeping it empty means disabled.
    provider: giscus # [disqus | utterances | giscus]
    # The provider options are as follows:
    disqus:
        shortname: # fill with the Disqus shortname. › https://help.disqus.com/en/articles/1717111-what-s-a-shortname
    # utterances settings › https://utteranc.es/
    utterances:
        repo: # <gh-username>/<repo>
        issue_term: # < url | pathname | title | ...>
    # Giscus options › https://giscus.app
    giscus:
        repo: xxx/xxx.github.io # <gh-username>/<repo>
        repo_id: R_kgDxxx
        category: General
        category_id: DIC_kwDxxx
        mapping: pathname # optional, default to 'pathname'
        strict: # optional, default to '0'
        input_position: # optional, default to 'bottom'
        lang: # optional, default to the value of `site.lang`
        reactions_enabled: # optional, default to the value of `1`
```
{: file='_config.yml' .nolineno }

![Desktop View](/assets/img/20241207/blog_comment.png){: width="600" height="300" }
_blog comment_

## 写博客

参考[Writing a New Post](https://chirpy.cotes.page/posts/write-a-new-post/)，写第一篇博客吧。

## 给网站添加一些小插件

先来看看功能效果图，

![Desktop View](/assets/img/20241207/busuanzi_count_site.png){: width="400" height="200" }
_效果图 1_

![Desktop View](/assets/img/20241207/busuanzi_count_post.png){: width="400" height="200" }
_效果图 2_

### 统计并显示网站访问量
`jekyll chirpy`主题中支持`goatcounter`计数器，貌似需要注册账号，本站没使用这种方式，而是选择了[不蒜子](https://busuanzi.ibruce.info/)计数，参考[不蒜子使用手册](https://ibruce.info/2015/04/04/busuanzi/)。

1. **分别创建统计网站pv/uv、网页pv计数器**

    ```html
    <script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
    </script>

    <!-- pv的方式，单个用户点击1篇文章，本篇文章记录1次阅读量 -->
    本文总阅读量<em><span id="busuanzi_value_page_pv"></span>次</em>，
    ```
    {: file='page_pv.html' .nolineno }

    ```html
    <script async src="https://busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
    </script>

    <!-- pv的方式，单个用户连续点击n篇文章，记录n次访问量 -->
    <span id="busuanzi_container_site_pv">
        本站总访问量<em><span id="busuanzi_value_site_pv"></span>次</em>，
    </span>

    <!-- uv的方式，单个用户连续点击n篇文章，只记录1次访客数 -->
    <span id="busuanzi_container_site_uv">
    本站访客数<em><span id="busuanzi_value_site_uv"></span>人次</em>
    </span>
    ```
    {: file='site_pv_uv.html' .nolineno }

2. **分别把计数器添加到post、footer容器中**

    ```html
    <!-- ...... -->
    <!-- busuanzi counter -->
    {include page_pv.html content=content prompt=true lang=lang}
    <!-- ...... -->
    ```
    {: file='post_case.html' .nolineno }

    ```html
    <!-- ...... -->
    <p>
    <!-- busuanzi counter -->
        {include site_pv_uv.html content=content prompt=true lang=lang}
    </p>
    <!-- ...... -->
    ```
    {: file='footer_case.html' .nolineno }

### 使用网易云音乐插件播放音乐
1. **在includes目录创建cloud-music.html**

    ```html
    <!-- cloud music -->
    <!-- auto=1 可以控制自动播放与否，当值为 1 即打开网页就自动播放，值为 0 时需要访客手动点击播放 -->
    <iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86
            src="https://music.163.com/outchain/player?type=2&id={{ page.music-id }}&auto=1&height=66">
    </iframe>
    ```
    {: file='cloud-music.html' .nolineno }

2. **把cloud-music.html嵌入post容器**

    ```html
    <!-- 添加网易云音乐插件 -->
    {if page.music-id}
    {include cloud-music.html}
    {endif}
    ```
    {: file='post_case.html' .nolineno }

3. **在md文件开头的配置项里添加 music-id: xxxxx**
- `xxxxx`为网易云外链播放器的曲目`id`。




## End 🌈🌈🌈
