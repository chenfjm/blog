
##安装

- 安装[RubyInstaller](http://rubyinstaller.org/downloads/)   
- 安装[DevKit](http://rubyinstaller.org/downloads/)   
- 安装Jekyll：
    -	$ gem install jekyll   
    -	$ gem install rdiscount

##创建版本库

登陆你的[Github账户](http://github.com)，创建一个新的版本库，命名为`USERNAME`.github.com
   
安装Jekyll-Bootstrap:   

	$ git clone https://github.com/plusjade/jekyll-bootstrap.git
	$ git remote set-url origin 
 	$ git push origin master   

如果你安装了Jekyll，你可以在本地预览你的Blog:     

	$ git clone https://github.com/plusjade/jekyll-bootstrap.git
	$ cd jekyll-bootstrap
	$ jekyll --server    

在浏览器预览http://localhost:4000.

##创建第一篇博文
>   rake post title="posttitle"

默认情况下rake命令会在你的_posts目录下创建一个名为[年-月-日-posttitle.md]的文件，名称中的空格会转换成“-”，时间为当前系统时间。

##创建第一个页面
根目录下创建页面  
>	$ rake page name="about.md"

自定义目录下创建页面    
>	$ rake page name="pages/about.md"

创建类似./pages/about/index.html目录结构的页面   
>	$ rake page name="pages/about"

##发布

完成一篇博文或者做一些修改之后可以用简单的git命令提交到远程的Github版本库。同时Github可以将md文件解析成html文件，通过    USERNAME.github.com就可以访问刚才提交的博文。

    $ git add .
    $ git commit -m "description"
    $ git push origin master

当然你还可以运用一些预置的主题，做一些自定义的配置，以及自己定义主题增加模板配置文件、增加Blog挂件、加入Google Analytics、Disqus等等   


