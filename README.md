1. **main**分支存储静态生成文件，网站直接访问的是该分支的内容
2. **hexo**分支存储源码，不同主机修改和同步的是该分支source/_posts路径下的博客。
3. 使用命令`hexo new [layout] <title>`来创建文章。layout 是文章的布局，默认为post
3. 使用命令`hexo g && hexo d` 可以将新写的文章生成静态文件推送到**main**分支。在主页中显示
   在此之前，有时可能需要执行`hexo clean`
4. 使用 `hexo s`可以在本地预览生成的博客
5. 分别使用下面三个命令，可以将新写的博客源文件推送到默认分支(hexo)，实现多个机器之间的同步
   - `git add .`
   - `git commit -m ChangeFiles（更新信息内容可改)`
   - `git push`
   - 此外，为了保证同步，推荐先`git pull`合并更新再进行博客的编写

6. 如果不想显示某一篇文章，就在该文章的开头加上`published: false`
