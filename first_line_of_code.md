# 第一行代码笔记

### 项目目录

* proguard-rules.pro
  * 指定项目代码混淆规则，防止其他人破解
* AppCompatActivity
  * 向下兼容到Android2.1系统
* build.gradle
  * jcenter:	代码托管仓库
  * dependencies:	
    * 本地依赖: compile fileTree(dir: 'lib', include: ['*.jar'])
    * 库依赖: compile project(':helper')
    * 远程依赖: compile 'com.android.support:appcompat-v7 : 24.2.1'

### 日志工具Log

* Log.v    verbose 
* Log.d   debug
* Log.i    info
* Log.w  warn
* Log.e   error
* 以上过滤程度从低到高: verbose可以看到所有日志，error只能看到error日志
* 在Android Studio中，还可以Edit Filter Configuration去自定义日志过滤器