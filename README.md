# programming_blog
## 本地运行

* 拉取 docker 镜像

~~~
docker pull linling/jekyll_env:2.0.0
~~~

* 启动镜像

~~~
docker run --name="jekyll_env" -it  -p  0.0.0.0:3999:4000/tcp --privileged  -v /Users/linling/Documents/study/jekyll/programming_blog:/programming_blog   linling/jekyll_env:1.0.0
~~~

* 启动 jekyll

~~~
cd /programming_blog
bundle exec jekyll serve --host 0.0.0.0
~~~

宿主机上访问 `http://127.0.0.1:3999/programming_blog/`

<br>

## 常见问题
### 文章提交 Github 后，不生效  

* 问题现象

本地调试正常，但是提交到 Github 后却不生效，也没有收到 Github 发送的 Build Failure邮件。  
此时查看，commit 记录 或者 environment 发现新提交的代码没有被 build 成功：  

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8no3v9inpj30rw0jl76j.jpg)

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8no53faslj30t10gugnz.jpg)

如果发生如上情况，很有可能是因为 Github 使用了较新版本的 jekyll，而你本地调试的 jekyll 版本低于 Github 使用的版本，所以即使你使用以前的本地 jekyll 编译没有问题，远端的 jekyll 编译时也可能出错。

* 解决方案

更新本地 jekyll

~~~
# gem install jekyll

// 在本地编译你的repository
# bundle exec jekyll build --safe

// 在本地运行服务 
# bundle exec jekyll serve --host 0.0.0.0
~~~
