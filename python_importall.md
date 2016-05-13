Date: 2015-05-13
Title: python中使用import加载目录中全部文件
Tags:  经验
Toc:no
Status: public
Position: 1

最近准备换用Flask,于是开始研究Python了.py的包导入机制,没有PHP那么直观,直接写文件名字就行.虽然学习起来有一些难度,不过能够有效的避免大型项目的命名空间问题,相比PHP 等到实在不行了引入一个namespace反而更乱了.

关于import 的基本用法就不说了,这个教程很多了.想要从其他文件夹里面import文件,需要在文件夹中建立一个__init__.py的文件用于初始化和声明,这个文件是空的也ok,不过在import的时候会自动执行这个文件,也可以做初始化来使用.

不过虽然建立了这个文件,在其他地方引用的时候还是只能每次通过from xx import xx加载一个文件,不是很方便.标准的用法是,修改目录下的__init__.py文件,用这个来实现加载全部文件的目的.

手动的方法:

```
import a
import b
```
或者:
```
__all__ = ["a", "b"]

```
这样会自动加载目录下的a.py和b.py文件了.

来个终极懒人的方法:

```
from os.path import dirname, basename, isfile
import glob
modules = glob.glob(dirname(__file__)+"/*.py")
__all__ = [ basename(f)[:-3] for f in modules if isfile(f)]

```

自动加载目录下的全部文件.再一次感叹py的简洁和强大的表达能力.
