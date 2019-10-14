# programming_blog
## 本地运行

* 拉取 docker 镜像

~~~
docker pull linling/jekyll_env:1.0.0
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
