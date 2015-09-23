Date: 2015-09-23
Title: 使用GITHUB+JustWriting搭建BLOG
Intro:   PHP blog with no database
Tags:  经验
Toc:yes
Status: public
Position: 1

最近觉得需要写一个BLOG了.至于为什么有这种感觉,后面再说了.

>**Why not Wordpress**


写BLOG之前都是用Wordpress,又大又慢比较蛋疼,而且树大招风,漏洞比较多.经常被人攻击挂马,烦.整天研究插件主题乱七八糟的东西,耗时耗精力,不专注.因为vps不固定,备份也是个问题,文件需要打包,数据库也要单独备份,麻烦.这次在公司使用的开源WIKI软件 dokuWiki,不用数据库直接用文件来保存数据,这种做法深得我心,简单方便.换个环境,基本上配置好NGINX扔那解析过去就能跑,而且从源头上杜绝了SQL Injection的问题,安全.于是目标就是寻找一个不用数据库的博客系统.

>**PHP BLOG Engine With no Database**

之前见过的一个方案是拿github搭博客,总觉得这样比较蛋疼,而且VPS也浪费了,可能也不利于搜索引擎过来抓,所以放弃了.

google一下,blog with no database,第一个结果出来的是dropplets,看着官网还不错.兴冲冲装上,根本不知道怎么用啊,看了看代码,写的一坨屎,觉得不怎么安全.放弃了

继续找,无意中发现了JustWriting(https://github.com/hjue/JustWriting),star很多 这就是口碑的保证啊.看文档,直接把md文件扔上去就能显示,符合需求.默认主题不难看,而且国人开发,github开源,就这个了.

作者推荐的是通过Dropbox来同步md文件,因为GFW的问题dp在国内也不好用,好麻烦.todo里面写着未来预计支持百度网盘,然而这么久过去估计也没信儿了.想了想,直接用github来搞,VPS上clone下来仓库,写个脚本每分钟拉一次,在本地写好了push上去 就能显示了,so easy.搞起来

git先建个仓库,教程可以看这个(http://www.cnblogs.com/smilejinge/p/3589479.html),弄好之后把服务器上代码里的posts目录干掉,clone自己的仓库然后mv成posts,随便找个目录写个pull.sh脚本保存一下:

```
#!/bin/bash
cd /home/wwwroot/blog/posts
git pull origin master
```
crontab -e,用vim添加:
```
*/1 * * * *  source /home/wwwroot/fetch.sh
```
每分钟pull一次git,妥妥够了.

然后在自己电脑上clone一下,写好了commit&push就搞定啦~

ps:另外发现一个在线编辑Markdown文本的地方:https://stackedit.io/, 支持直接发布git和ssh server,但是觉得图片问题不好解决,得找好图床然后做外链,不方便.等有空给JustWriteing写一个上传图片的工具 应该就ok了






