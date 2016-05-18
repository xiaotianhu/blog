Date: 2016-03-12
Title: mac安装mysql-python(with XAMPP)
Tags:  mac mysql-python XAMPP flask
Toc:no
Status: public
Position: 1

继续研究flask框架,SQL部分准备用SQLAlchemy.调用的时候报错:

```
ImportError: No module named MySQLdb
```

搜了一下,需要安装mysql-python :http://stackoverflow.com/questions/25459386/mac-os-x-environmenterror-mysql-config-not-found
```
sudo pip install mysql-python
```
安装的时候,报错 找不到mysql_config.想到之前装过XAMPP,里面是包含mysql的.
```
$ locate mysql_config
/Applications/XAMPP/xamppfiles/bin/mysql_config

```
果然有.加入到PATH里就能找到了
```
export PATH=$PATH:/Applications/XAMPP/xamppfiles/bin/
```

装好之后运行,还是报错:
```
Reason: unsafe use of relative rpath libmysqlclient.18.dylib in /Library/Python/2.7/site-packages/_mysql.so with restricted binary
```

搜了一下,跟新macos EICapitan的安全机制有关系:http://stackoverflow.com/questions/31343299/mysql-improperly-configured-reason-unsafe-use-of-relative-path

按着答案,执行
```
sudo install_name_tool -change libmysqlclient.18.dylib /Applications/XAMPP/xamppfiles/lib/libmysqlclient.18.dylib /Library/Python/2.7/site-packages/_mysql.so
```
终于执行成功了.

总结:stackoverflow真是个好东西.
