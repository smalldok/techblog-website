# 技术博客生成工具

* 构建工具 [Hexo](https://hexo.io/zh-cn/docs/)
* 主题 [NexT.Mist](https://github.com/iissnan/hexo-theme-next)

`我生成的技术博客访问地址：` [https://smalldok.github.io/index.html](https://smalldok.github.io/index.html)


# 使用

一般的写文章操作步骤：

* 手写markdown，如 xxx.md
* 打开xxx.md进行编辑，在文章最上面加入

    ```
    ---
    title: 这是我的文章标题
    comments: true
    date: 2018-07-9 10:12:00
    tags:
      - docker  
    categories: 容器
    ---
    ```
* 在hexo安装目录下，执行命令
    ```
    $ hexo g #生成页面
    $ hexo d #部署到github
    ```


# 其他

```
$ hexo new post <title> #写文章
$ hexo server #启动本地服务, 访问http://localhost:4000/
$ hexo clean #清除本地缓存
$ hexo g #生成页面
$ hexo d #部署到github
$ hexo new page "about"  #新建关于页面
$ hexo new page "tags"  #新建标签页面
$ hexo new page "categories" #新建类别页面
$ hexo new page "categories" #新建归档页面

```

# 参考
* https://hexo.io/zh-cn/docs/  
* http://www.jianshu.com/p/58e789456440

# 常见问题
## hexo g无法生成页面,报错
一般是编写了md文件，里面的markdowm格式不规范，建议检查修改，使用atom编辑md文件会更容易看到问题；