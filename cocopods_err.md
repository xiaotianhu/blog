Ttitle: cocopods重装解决framework not found Pods/no such module

刚开始学习Swift ios开发。
使用cocopods安装SnapKit, Xcode版本10.1 Swift4.2,按照Snapkit官网文档，修改Podfile之后Podinstall，中途一顿操作，结果不能用。  

发现我原来是忘记在需要的地方import SnapKit了。  

加上这行后，Xcode报 no such module。一顿折腾无果，找到一个解决方案：https://stackoverflow.com/questions/29865899/ld-framework-not-found-pods

重新初始化pods的好方法：

```
Xcode 9, 10, 11

install https://github.com/CocoaPods/cocoapods-deintegrate

pod deintegrate
then

pod install
```

完美解决，import SnapKit就可以了。
