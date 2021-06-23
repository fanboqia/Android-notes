# 安卓进阶解密笔记

#### 安卓操作系统结构

* Application (Java)
* Framework (Java API framework)
* Native 
  * C/C++
  * Runtime library
    * Core library
    * ART
* HAL (硬件抽象层) 软件调用硬件接口
* Linux Kernel

#### Android系统启动

* 电源按下，ROM程序进入RAM
* Bootloader导引程序拉起OS
* Linux 内核启动寻找init.rc，执行init进程
* Init进程初始化，fork Zygote子进程
* ![](imgs\launch_flow.png)

#### Android Init Language

* 解析init.rc, init.zygote64.rc
* Action, Command, Service, Option, and Import
  * Service
    * ServiceParser 解析器
      * `std::make_unique<Service>(name, str_args)` <font color=red>学习make_unique</font> 
      * ServiceManager单例加入Service链表中
    * 启动Zygote
      * `ServiceManager::GetInstance().ForEachServiceInClass(args[1], [](Service * s){ s->StartIfNotDisabled();});` <font color=red>学习lambda表达式</font> 
      * 先判断是否已经启动
        * 如果启动了那么fork一个子Zygote进程
        * <font color=red>学习execve</font> 

![](imgs/init_zygote.png)

#### 属性服务

* 记录之前的注册表中的记录，初始化时使用
* 网络编程
  * <font color=red>epoll</font>
  * non-blocking socket
* 检查属性
  * ctl. 控制属性: 执行命令
  * 普通属性
    * ro. 只读
    * persist. 

#### Zygote

* Zygote创造DVM,ART,application process, and System Server
* 通过 args 的 --zygote, --start-system-server来区分当前进程是在zygote，还是在system-server中，因为zygote进程是通过fork自己创建子进程的，需要flag来区分
* runtime.start() C++方法来调用Java方法
  * 启动Java虚拟机
  * 为Java虚拟机注册JNI方法
  * `env->NewStringUTF(className)` C++ String to Java String
  * `toSlashClassName(className)` //将className . 替换成 /
  * `env->FindClass(slashClassName)`
  * `jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;])V")` 获取main方法
  * `env->CallStaticVoidMethod(startClass, startMeth, strArray);` 通过JNI调用main方法
* ZygoteInit.java
  * 创建一个Server端的Socket
  * 预加载类和资源
  * 启动SystemServer
  * 等待AMS请求

#### SystemServer

* SystemServer创建AMS, WMS, 和 PMS
* 启动Binder线程池与其他线程通信
* 加载动态库libandroid_servers.so
* 启动引导服务
* 启动核心服务
* 启动其他服务
* ![](imgs/system_server.png)

#### PackageManagerService

* 由mSystemServiceManager创建
* 注册到ServiceManager
  * 管理各种Service
  * C/S架构中Binder通信机制

#### Launcher

* 是Android系统的桌面
  * 用于启动应用程序
  * 管理应用图标和显示
* 请求PackageManagerService返回系统中已安装的application信息
* 调用了AMS的startHomeActivityLocked方法
* 三种工厂模式
  * 非工厂模式
  * 低级工厂模式
  * 高级工厂模式
* 有HomeIntent来启动桌面活动
* 如何显示各应用图标
  * 每个工作区用来描述一个抽象桌面
    * 由n个屏幕组成，每个屏幕分成n个单元格，每个单元格用来显示一个应用程序的快捷图标

#### 应用程序进程启动过程

* 应用程序需要先有对应的进程
* AMS作为Client端，发送请求到Zygote的Server端Socket，来创建应用程序进程
* 连接Zygote分为主模式(zygote)和辅模式(zygote_secondary)
* ABI (Application Binary Interface)
  * 应用程序二进制接口ABI（Application Binary Interface）定义了二进制文件（尤其是.so文件）如何运行在相应的系统平台上，从使用的指令集，内存对齐到可用的系统函数库。
* ActivityThread
* Binder线程池
  * `ZygoteInit.nativeZygoteInit()`
  * `ProcessState->startThreadPool()`
  * `spawnPooledThread()`
  * IPCThreadState的joinThreadPool将当前线程注册到Binder驱动程序中,新创建的线程就加入了Binder线程池中，新创建的应用就支持Binder进程间通信

#### 消息循环创建

* ActivityThread类用于管理当前应用进程的主线程
* `Looper.prepareMainLooper()`
* `sMainThreadHandler = thread.getHandler()`
* `Looper.loop()`

#### 四大组件的工作过程

#### 根Activity的启动过程

* Root Activity是第一个被启动的活动

* 调用startActivity

* 调用Instrumentation的execStartActivity

  * Instrumentation用来监控应用程序和系统的交互

* ActivityManager.getService()

* 调用IActivityManagerSingleton的get方法

  * ```java
    private static final Singleton<IActivityManager> IActivityManagerSingleton = new Singleton<IActivityManager>(){
        @Override
        protected IActivityManager create(){
            final IBinder  b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
            final IActivityManager am = IActivityManager.Stub.asInterface(b);
            return am;
        }
    }
    ```

  * 以上是AIDL生成的

  * IActivityManager.Stub类实现进程间通信

* AMS到ApplicationThread

  * ![](imgs\ams.png)

* ![](imgs\activity_launch.png)

#### Service的启动过程

* ContextImpl到ActivityManageService
  * Activity.attach将Activity和上下文appContext关联起来
* ActivityThread启动Service
  * ServiceLookupResult
  * ServiceRecord (可以从PackageManagerService获取)
  * 找到ServiceRecord对应的ProcessRecord
    * 如果进程不存在
      * 调用AMS的startProcessLocked方法创建对应的应用进程
  * ApplicationThread是ActivityThread的内部类
  * Service被封装成ActivityClientRecord
  * sendMessage方法向H类发送为CREATE_SERVICE的消息
  * mH指向H类，是ActivityThread的内部类并继承自Handler，是应用进程中主线程的消息管理类
  * 获取要启动Service的应用程序的LoadedAPK
  * Service.attach来初始化
  * 调用service的onCreate方法

#### Service的绑定过程

* ContextImpl到AMS的绑定过程
  * IServiceConnection实现Binder机制，这样Service绑定就支持跨进程
    * 具体实现为ServiceDispatcher.InnerConnection，其中ServiceDispatcher是LoadedApk的内部类
  * bindService
* Service的绑定过程

#### 广播的注册、发送和接受过程

* 广播的注册
  * 静态注册 (由PackageManagerService来注册完成)
  * 动态注册 (registerReceiver方法)
    * ContextWrapper中实现
    * ContextImpl中registerReceiverInternal
    * 获取IIntentReceiver对象
    * 注册广播是一个跨进程过程
    * 粘性广播 (sendStickyBroadcast)
      * Intent会一直保留到广播事件结束，而这种广播也没有所谓的10秒限制，10秒限制是指普通的广播如果onReceive方法执行时间太长，超过10秒的时候系统会将这个广播置为可以干掉的candidate，一旦系统资源不够的时候，就会干掉这个广播而让它不执行
* 广播的发送
  * ContextImpl到AMS
    * 调用AMS的broadcastIntent方法
      1. 验证广播是否合法
      2. 将动态注册的广播接收者和静态注册的广播接受者按照优先级高低不同存储在不同的列表中
      3. 调用broadcastQueue
  * AMS到BroadReceiver
    * BroadcastHandler对象发送BROADCAST_INTENT_MSG类型消息
    * 在BroadcastHandler的handleMessage方法中处理
    * 进入processNextBroadcast方法
    * 遍历无序广播发送给Receivers
    * 判断广播接受者所在进程是否存在，即ApplicationThread
    * 通过IIntentReceiver实现跨进程通信(AIDL)，具体实现为LoadedApk.ReceiverDispatcher.InnerReceiver继承自IIntentReceiver.Stub是Binder通信的服务端，而IIntentReceiver则是Binder通信的客户端
    * LoadedApk.ReceiverDispatcher执行具体逻辑
    * mActivityThread.post(args.getRunnable())
    * 将Args对象的getRunnable方法通过H类发送到线程的消息队列中

#### ContentProvider的启动过程

* query到AMS
  * getContentResolver()
  * ContextImpl.getContentResolver()返回ApplicationContentResolver
  * acquireUnstableProvider(Uri uri)
  * mMainThread.acquireProvider()即ActivityThread获取Provider
  * 有一个mProviderMap用来缓存ContentProvider，不需要每次都调用AMS的getContentProvider方法
  * 获取ContentProvider的应用进程信息（ProcessRecord）
  * 判断进程是否已经启动，如果没有就创建
  * 如果没有启动
    * ActivityThread的main里的Looper.loop()会开始消息循环
    * 调用AMS的attachApplication方法
* AMS启动ContentProvider
  * AMS的attachApplication
  * thread的bindApplication向H发送BIND_APPLICATION类型信息
  * ActivityThread的handleBindApplication
  * 创建ContextImpl，反射创建Instrucmentation，创建application，调用onCreate方法，在启动之前启动ContentProvider
  * AMS的publishConentProvider方法将ContentProvider存储在AMS的mProviderMap中
  * 反射创建localProvider,调用ContentProvider的onCreate抽象方法

#### 上下文对象Context

![](imgs\context.png)

* ContextWrapper使用了装饰模式
* ContextThemeWrapper中包含不同的主题，那么Activity就继承他。反之，Service不需要主题
* 组合而非继承的方式

#### Application Context的创建过程

* ActivityThread类的内部类ApplicationThread的scheduleLaunchActivity方法启动Activity
* ApplicationThread的scheduleLaunchActivity方法向H类发送消息，目的是启动Activity在主线程中
* H类的handleMessage通过getPackageInfoNoCheck获得LoadedApk对象
* ActivityThread的performLaunchActivity
* LoadedApk.makeApplication
* ContextImpl的createAppContext方法创建ContextImpl
* mActivityThread.mInstrumentation.newApplication传入contextImpl
* ContextImpl.setOuterContext(app)
* app.attach(context)
* 将ContextImpl赋值给ContextWrapper的mBase

#### Application Context的获取过程

* ContextWrapper的getApplicationContext()
* 如果LoadedApk的mPackageInfo不为null，则调用LoadedApk的getApplication方法
* 返回mApplication

#### Activity的Context创建过程

* ActivityThread的createBaseContextForActivity(ActivityClientRecord)返回ContextImpl
* 传入ContextImpl到activity通过attach方法
* ContextImpl.createActivityContext返回ContextImpl
* attach方法中调用attachBaseContext(ContextThemeWrapper)
* 调用父类(ContextWrapper)的attachBaseContext

#### Service的Context创建过程

* ActivityThread启动Service
  * ActivityThread的内部类ApplicationThread的scheduleCreateService
  * 其中sendMessage方法向H类发送CREATE_SERVICE消息
  * H类handleMessage调用ActivityThread的handleCreateService
  * ContextImpl.createAppContext返回ContextImpl
  * service.attach(context)
  * attachBaseContext(context)

#### 理解AMS

* Android7.0 AMS
  * ActivityManager通过ActivityManagerNative的getDefault方法获取ActivityManagerProxy，通过AMP和AMS通信
  * Instrumentation的execStartActivity调用ActivityManagerNative的getDefault获取AMP
    * getDefault是一个singleton类
    * 获取IBinder类型的AMS引用
    * 将其封装成AMP类型对象 (asInterface(IBinder)) -> new ActivityManagerProxy(IBinder)
  * 通过IBinder类型对象mRemote向服务端AMS发送一个START_ACTIVITY_TRANSACTION类型的进程间通信请求，服务端会从Binder线程池中读取客户端发来的数据，最终会调用AMN的onTransact方法
  * onTransact调用AMS的startActivity方法
* AMP是Client端，AMN是Server端
* AMS是AMN的子类，AMP是AMS的Client端代理，AMN又实现了Binder类，这样AMP和AMS就可以实现Binder间通信了

* ![](imgs\android7_ams.png)

* ![](imgs\amp_ams.png)

#### Android8.0的AMS

* 和7.0的区别在于采用了AIDL (IActivityManager是AIDL工具在编译时自动生成)
* 服务端只需要让AMS继承IActivityManager.Stub并实现方法
* ![](imgs\android8_pms.png)

#### AMS的启动过程

* SystemServer中startBootstrapServices方法启动引导服务
* mSystemServiceManager.startServer(ActivityManagerService.Lifecycle.class)
* service.onStart就是启动了AMS

#### AMS与应用进程

* AMS检查应用程序所在进程是否存在，如果不存在就请求Zygote进程创建所需的应用进程

#### AMS重要的数据结构

* ActivityRecord
  * ![](imgs\activity_record.png)

* TaskRecord
  * ![](imgs\task_record.png)

* ActivityStack

  * ActivityStackSupervisor管理ActivityStack

  * ActivityState

    * ```java
      enum ActivityState{
          INITIALIZATING,
          RESUMED,
          PAUSING,
          PAUSED,
          STOPPING,
          STOPPED,
          FINISHING,
          DESTROYING,
          DESTROYED
      }
      ```

    * 当ActivityState为RESUMED或者PAUSING时才会调用WMS的方法来切换动画

#### Activity任务栈管理

![](imgs\activity_inst.png)

* LaunchFlags类似launch mode，只不过在源码中
* taskAffinity设置在AndroidManifest.xml，用来指定Activity希望归属的栈

#### 理解WindowManager

* Window, WindowManager, 和 WMS

* Window是一个抽象类（具体实现为PhoneWindow）, WindowManage是一个接口继承自ViewManager，WindowManager的实现类为WindowManagerImpl

* ![](imgs\wms.png)

  * WindowManager除了拥有ViewManager增删改的方法，还添加了

    * ```java
      public Display getDefaultDisplay(); //将Window添加到哪个屏幕上
      public void removeViewImmediate(); //完成传入View的相关销毁工作
      ```

  * PhoneWindow在Activity的attach中创建

  * WindowMangerImpl在addView方法中使用了[桥接模式](https://www.runoob.com/w3cnote/bridge-pattern2.html)，将功能[委托](https://www.runoob.com/w3cnote/delegate-mode.html)给了WindowManagerGlobal

  * ![](imgs\wm.png)

* Window的类型

  * Application Window （应用程序窗口）
    * Type从1-99
  * Sub Window  (子窗口)
    * Type从1000-1999
  * System Window (系统窗口)
    * Type从2000-2999
  * 窗口显示次序
    * 从屏幕里到屏幕外作为z轴
    * Type越大越靠近用户（排前面）

* Window的标志

  * ![](imgs\window_flag.png)

  * ```java
    Window mWindow = getWindow();
    mWindow.addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
    ```

  * ```java
    Window mWindow = getWindow();
    mWindow.setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);
    ```

* 系统窗口的添加过程
  * StatusBar是系统的状态栏（显示时间，电量，信号）
  * 构造StatusBar视图并且添加到StatusBarManagerWindowManager
  * mWindowManager.addView(mStatusBarView) 添加到WMS中，调用的其实是ViewManager的addView方法
  * mGlobal.addView
  * ViewRootImpl
    * View树的根并管理View树
    * 触发View树的测量
    * 管理Surface
    * 负责与WMS进行进程间通信
  * IWindowSession是一个Binder类型，是Client的代理; Server端实现为Session
  * ![](imgs\viewRootImpl.png)
  * WMS为添加的窗口分配Surface，并确定窗口的显示次序，Surface就是画布，而SurfaceFlinger是一个混合器
* Window的更新过程
  * ViewManager的updateViewLayout，在WindowManagerImpl中实现
  * 调用WindowManagerGlobal的updateViewLayout方法
  * ViewRootImpl的scheduleTraversals
  * performTraversals()
    * relayout (重新构造结构)
    * measure (重新测量)
    * draw (重新绘制)

#### 理解WindowManagerService

* 职责
  * 窗口管理
  * 窗口动画 (WindowAnimator)
  * 输入系统的中转站（触摸事件）（InputManagerService）
  * Surface管理 （绘制）
  * ![](imgs\wms_job.png)

* WMS的创建过程
  * SystemServer的main方法
  * WMS属于其他服务的一种（引导服务i.e.AMS，核心服务i.e.BatteryService,，其他服务i.e.WMS）
  * startOtherService中
    * watchdog用来监视系统的一些关机服务
    * WMS是在SystemServer的线程中的
    * 调用WMS的main方法
  * DisplayThread.getHandler().runWithScissors()是一个单例的前台线程，这个线程用来处理需要低延时显示的相关操作
    * 判断了当前线程是否是Handler所指向的线程
      * 如果是
        * 执行Runnable的run方法
      * 如果不是
        * 执行BlockingRunnable的postAndWait方法
        * 让system_server线程等待android.display线程
  * 创建了一个WindowManagerService的单例
  * 构造方法中
    * 持有IMS的引用
    * AMS的引用
    * 创建WindowAnimator
    * WindowManagerPolicy定义一个窗口策略所要遵守的通用规范
    * 将WMS加入到Watchdog中（Watchdog每分钟都会对被监控的系统服务进行检查，如果服务出现死锁，则会杀死Watchdog所在进程）
  * ![](imgs\thread1.png)

* WMS的重要成员
  * mPolicy: WindowManagerPolicy
    * 允许定制窗口层级和特殊窗口类型以及关键调度和布局
  * mSession: ArraySet
    * 进程间通信
    * 其他应用进程通信和WMS必须经过Session，每个进程对应一个session
  * mWindowMap: WindowHashMap
    * 继承自HashMap
    * 限制了HashMap的key为IBinder类型
  * mFinishedStarting: ArrayList
    * WindowToken的子类
      * 窗口令牌（应用程序申请窗口的通行证）
  * mResizingWindows: ArrayList
    * WindowState
    * 用来存储正在调整大小的窗口
  * mAnimator: WindowAnimator
    * 管理窗口动画和特效动画
  * mH: H
    * 系统的Handler类，用于将任务加入到主线程的消息队列中
  * mInputManager: InputManagerService
    * 输入系统的管理者

* Window的添加过程
  * WMS的addWindow方法
    * WMP的checkAddPermission方法来检查权限
    * 通过displayId来获得窗口要添加到哪个DisplayContent上（DisplayContent用来描述一块屏幕）
    * type代表一个窗口类型位于1000-1999（是子窗口）
    * windowForClientLocked方法会根据attrs.token作为key从mWindowMap中获得该子窗口的父窗口
  * 创建WindowState
  * IWindow会将WMS中窗口管理的操作回调给ViewRootImpl
  * 判断添加窗口的客户端是否已经死亡

* Window的删除过程
  * WindowManagerImpl的removeView
  * ViewRootImpl的die方法
  * immediate为立即执行
  * WindowManagerGlobal的doRemoveView
    * 从ViewRootImpl列表，布局参数列表和View列表中删除View元素
  * ViewRootImpl的doDie中dispatchDetachedFromWindow
  * mWindowSession.remove(mWindow);
  * WMS的removeWindow
  * removeIfPossible（不会直接执行删除操作，进行多个条件过滤）
  * removeImmediately()
  * 清除session对应的surfaceSession
  * WMS的postWindowRemoveCleanupLocked进行对View的清理

#### JNI原理

* Java Native Interface（Java与其他语言通信的桥梁）

  * 调用Java语言不支持的依赖操作系统平台特性的一些功能
  * 整合以前非Java语言开发的系统
  * C/C++提升运行效率

* MediaRecorder分析

  * Framework层

    * ```java
      public class MediaRecorder{
          static{
              System.loadLibrary("media_jni");
              native_init();
          }
          
          private static native final void native_init();
          
          public native void start() throws IllegalStateException;
      }
      ```

  * JNI层

    * ```c++
      static void android_media_MediaRecorder_native_init(JNIEnv *env){
          jclass clazz;
          clazz = env->FindClass("android/media/MediaRecorder");
          if(clazz == NULL){
              return;
          }
          
          fields.post_event = env->GetStaticMethodID(clazz, "postEventFromNative", "(Ljava/lang/Object;IIILjava/lang/Object;)V");
          
          if(fields.post_event == NULL){
              return;
          }
      }
      
      static void android_media_MediaRecorder_start(JNIEnv *env, jobject thiz){
          ALOGV("start");
          sp<MediaRecorder> mr = getMediaRecorder(env, thiz);
          process_media_recorder_call(env, mr->start(), "java/lang/RuntimeException", "start failed.");
      }
      ```

  * Native方法注册

    * 静态注册

      * NDK开发

      * ```shell
        javac com/example/MediaRecorder.java
        javah com.example.MediaRecorder
        ```

      * 生成com_example_MediaRecorder.h文件

      * ```java
        public static native final void native_init();
        public native void start() throws IllegalStateException;
        ```

      * ```c
        JNIEXPORT void JNICALL Java_com_example_MediaRecorder_native_1init(JNIEnv *, jclass);
        //以Java开头，说明从Java平台调用
        //com.example.MediaRecorder变成了com_example_MediaRecorder
        //native_init变成了native_1init（这里因为原本的native_init中有"_"，所以用"_1"）
        //JNIEnv* 指针可以在Native世界中访问Java代码
        //jclass 对应java.lang.Class类
        
        JNIEXPORT void JNICALL Java_com_example_MediaRecorder_start(JNIEnv *, jobject);
        //这里的jobject指的是Java类的对象
        ```

    * 动态注册

      * Framework开发

      * ```c
        typedef struct{
            const char* name;//Java方法的名字
            const char* signature;//Java方法的签名信息
            void* fnPtr;//JNI中对应的方法指针
        } JNINativeMethod;
        ```

      * ```c++
        static const JNINativeMethod gMethods[] = {
            {"start", "()V", (void *) android_media_MediaRecorder_start},
            {"native_init", "()V", (void *) android_media_MediaRecorder_native_init}
        }
        //start是Java层方法名
        //()V是签名 ()代表没有参数，V代表返回参数为void
        //android_media_MediaRecorder_start 是对应的JNI函数
        ```

      * ```c++
        (*env)->RegisterNatives(e, c.get(), gMethods, numMethods)  //通过JNIEnv的RegisterNatives注册
        ```

      * JNI_OnLoad会在System.loadLibrary函数之后调用

    * 数据类型转换

      * 基本数据类型转换
        * ![](imgs\convert.png)

      * 引用数据类型转换
        * ![](imgs\reference.png)

      * 例子

        * ```java
          private native void_setOutputFile(FileDescriptor fd, long offset, long length) throws IllegalStateException, IOException
          ```

        * ```c++
          static void android_media_MediaRecorder_SetOutputFileFD(JNIEnv *env, jobject thiz, jobject fileDescriptor, jlong offset, jlong length)
          //JNIEnv是Java世界指针
          //jobject是Java类指针
          //jobject是FileDescriptor类指针
          //jlong是 long
          ```

    * 方法签名

      * ```java
        private native final void native_setup(Object mediarecorder_this, String clientName, String opPackageName) throws IllegalStateException
        ```

      * ```c++
        (Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;)V
        ```

      * 通过命令可自动生成签名

      * ```
        javac MediaRecorder.java
        javap -s -p MediaRecorder.class
        ```

    * 解析JNIEnv

      * 不能跨线程传递

      * 内部包含了JNINativeInterface

      * ```c++
        struct _JNIEnv{
            const struct JNINativeInterface* functions;
            
            #if defined(__cplusplus)
            	jclass FindClass(const char *name){
                    return functions->FindClass(this, name);
                }
            	
             	jmethodID GetMethodID(jclass clazz, const char* name, const char* sig){
                    return functions->GetMethodID(this, clazz, name, sig);
                }
            
            	jfieldID GetFieldID(jclass clazz, const char* name, const char* sig){
                    return functions->GetFieldID(this, clazz, name, sig);
                }
            #else
            
            #endif
        }
        ```

      * ```c++
        struct JNINativeInterface{
            
            jclass (*FindClass)(JNIEnv *, const char*);
            
            jmethodID (*GetMethodID)(jclass, const char*, const char*);
            
            jfieldID (*GetFieldID)(jclass, const char*, const char*);
        }
        ```

      * jmethodID指的Java方法

      * jfieldID指的是Java类成员变量

      * 用法

        * 获取

          * ![](imgs\usage.png)

        * 调用

          * ```c++
            JNIEnv* env = AndroidRuntime::getJNIEnv();
            env->CallStaticVoidMethod(mClass,fields.post_event,mObject,msg,ext1,ext2,NULL);
            ```

          * ```c++
            jobject surface = env->GetObjectField(thiz, fields.surface);
            ```

    * 引用类型

      * 本地引用（Local References）

        * 当Native函数返回时，这个本地引用就会被自动释放

        * 只在创建它的线程中有效，不能跨线程使用

        * 局部引用是JVM负责的引用类型，受JVM管理

        * ```c++
          jclass clazz = env->FindClass("android/media/MediaRecorder");
          //可以使用DeleteLocalRef手动释放
          ```

      * 全局引用（Global Reference）

        * 在native函数返回时不会被自动释放，因此全局引用需要手动进行释放，并且不会被GC回收

        * 可以跨进程使用

        * 不受JVM管理

        * ```c++
          jclass clazz = env->GetObjectClass(thiz);
          mClass = (jclass) env->NewGlobalRef(clazz);
          //将本地引用转换为全局引用
          ```

        * ```c++
          //析构函数
          JNIMediaRecorderListener::~JNIMediaRecorderListener(){
              //删除全局引用
              JNIEnv *env = AndroidRuntime::getJNIEnv();
              env->DeleteGlobalRef(mObject);
              env->DeleteGlobalRef(mClass);
          }
          ```

      * 弱全局引用（Weak Global References）

        * 特殊的全局引用

        * 可以被GC回收

        * ```c++
          jclass clazz = env->GetObjectClass(thiz);
          mClass = (jclass) env->NewWeakGlobalRef(clazz);
          //将本地引用转换为弱全局引用
          ```

        * ```c++
          //析构函数
          JNIMediaRecorderListener::~JNIMediaRecorderListener(){
              JNIEnv *env = AndroidRuntime::getJNIEnv();
              //因为可能被GC回收，要先判断是否为NULL
              if(!env->IsSameObject(mObject,NULL)){
                  env->DeleteWeakGlobalRef(mObject);
              }
          }
          ```

#### Java虚拟机

* 虚拟机家族
  * HotSpot
    * Longview
  * J9
    * IBM
  * Zing
    * 低延迟
    * 启动后快速预热
    * 可管理性
* ![](imgs\compile.png)
* 虚拟机架构
  * ![](imgs\arc.png)
* Class文件格式
  * ![](imgs\format.png)
* 类的生命周期
  * 加载：查找并加载Class文件
  * 链接：包括验证、准备和解析
  * 验证：确保被导入类型的正确性
  * 准备：为类的静态字段分配字段，并用默认值初始化这些字段
  * 解析：虚拟机将常量池内的符号引用替换为直接引用
* 类加载子系统
  * 系统加载器
    * Bootstrap ClassLoader（引导类记载器）
      * C/C++实现
      * 加载指定JDK核心库
        * java.lang
        * java.util
        * $JAVA_HOME/jre/lib
        * -Xbootclasspath
    * Extensions ClassLoader（拓展类加载器）
      * 加载Java的拓展类
        * $JAVA_HOME/jre/lib/ext
        * java.ext.dir
    * Application ClassLoader（应用程序类加载器）
      * 可以通过ClassLoader的getSystemClassLoader方法获取
      * 当前应用程序Classpath目录
      * java.class.path
  * 自定义加载器
    * 通过继承java.lang.ClassLoader类
* 运行时数据区域
  1. 程序计数器
     * 记录着每一条指令的地址
     * 线程私有的
  2. Java虚拟机栈
     * 线程私有
     * 有多个帧栈
       * 局部变量表
       * 动态链接
       * 方法出口
  3. 本地方法栈
     * Native方法的栈
  4. Java堆
     * 线程共享
     * 新生代
     * 老年代
     * 被垃圾回收器管理
  5. 方法区
     * 线程共享
  6. 运行时常量池
     * 方法区的一部分
* 对象的创建
  1. 判断对象对应的类是否加载、链接和初始化
  2. 为对象分配内存
     1. 指针碰撞
     2. 空闲列表
  3. 处理并发安全问题
     1. CAS算法
     2. 失败重试
     3. Thread Local Allocation Buffer（TLAB）
  4. 初始化分配到的内存空间
     1. 初始化为0
  5. 设置对象的对象头
     1. 对象所属类
     2. Hashcode
     3. GC分代年龄
  6. 执行init方法
* 对象的堆内存布局
  * 对象头
    * Mark World
      * HashCode
      * 锁标志位
      * GC分代年龄
    * 元数据指针
      * 方法区中目标类的元数据
  * 实例数据
    * 各种类型的字段信息
  * 对齐填充
    * 占位符
* oop-klass模型
  * oop（Ordinary Object Pointer）表示对象的实例信息
  * klass（描述元数据）
  * ![](imgs\oop.png)
* 垃圾标记算法
  * Java中的引用
    * 强引用
      * 创建一个对象，那个对象就是强引用，垃圾回收器不会回收它
    * 软引用
      * 内存不够时回收（SoftReference）
    * 弱引用
      * 一旦发现，不管当前内存是否够用都会回收
    * 虚引用
      * 任何时候都可能被GC回收
      * 被回收会受到通知
      * PhantomReference
  * 引用计数算法
    * 被引用一次+1
    * 引用次数为0，删除
    * 不能解决对象互相依赖的问题
  * 根搜索算法
    * ![](imgs\root.png)
    * GCRoots
      * Java栈中引用的对象
      * 本地方法栈中JNI引用的对象
      * 方法区中运行时常量池引用的对象
    * 可达性
* Java对象在虚拟机中的生命周期
  1. 创建阶段（Created）
  2. 应用阶段（In Use）
  3. 不可见阶段（Invisible）
  4. 不可达阶段（Unreachable）
  5. 收集阶段（Collected）
  6. 终结阶段（Finalized）
  7. 对象空间重新分配（Deallocated）
* 垃圾收集算法
  * Mark-Sweep
    * 标记阶段：标记出可回收的对象
    * 清除阶段：回收被标记的对象所占用的空间
    * 缺点：内存碎片化
  * 复制算法
    * 把内存空间分成两个相等的区域
    * 垃圾回收的时候，把存活的对象复制另外一个区域中，最后再回收当前区域
    * 缺点：内存变成原来的一半
    * 被广泛应用于新生代中
  * 标记整理算法
    * Mark-Sweep之后，再将存活的对象推至内存的一端，使得它们紧凑的排列在一起
* 分代收集算法
  * 新生代
    * Eden
    * From Survivor
    * To Survivor
    * HotSpot-> Eden : From+To Survivor = 8 : 1
  * 老年代
  * 分代收集
    * Minor Collection：新生代垃圾收集
    * Full Collection：老年代垃圾收集
      * 会伴随至少一次的Minor Collection
    * Minor Collection之后，Eden空间的存活对象会被复制到To Survivor空间，From Survivor对象会被复制到To Survivor空间
    * 两种情况Eden和To空间的存活对象不会复制到To Survivor空间，而是晋升到老年代
      * 存活的对象的分代年龄超过-XX:MaxTenuringThreshold（多少次Minor GC后才晋升到老年代）
      * To Survivor空间达到阈值
    * 当Eden和From Survivor空间中剩下的都是可回收的对象
      * 这个时候GC执行Minor Collection，Eden和From 会被清空
      * 随后，再将To Survivor和From Suvivor进行交换

#### Dalvik和ART

* Dalvik Virtual Machine (DVM)

  * Google为Android平台开发的虚拟机

  * 和JVM的区别

    * 架构不同
      * JVM基于栈，所以需要的指令更多，导致速度变慢
      * DVM基于寄存器，减少了出入栈指令
    * 字节码不同
      * JVM
        * java -> class -> jar
      * DVM
        * java -> class -> dex
      * ![](imgs\compare.png)
      * dex整合了class文件，去除了冗余信息，减少了I/O操作，加快了类的查找速度
    * DVM允许在有限的内存中运行多个进程
    * DVM由Zygote创建和初始化
      * Zygote fork自身来创建DVM实例
    * DVM有共享机制
      * 不同的应用在运行时可以共享相同的类
    * DVM早期没有使用JIT编译器
      * DVM通过解释器将dex编译成机器码
      * 从Android2.2开始，使用了JIT

  * DVM架构

    * 源码位于dalvik/目录下

  * DVM的运行时堆

    * Mark-Sweep
    * 两个space和多个辅助数据结构组成
      * 两个Space: Zygote Space 和 Allocation Space
      * Zygote Space不会触发GC，在Zygote进程和应用程序进程之间会共享Zygote Space
      * 在Zygote进程fork第一个子进程之前，会把Zygote Space分为两个部分
        * 原来已经被使用的那部分堆叫做Zygote Space
        * 未使用的那部分堆叫Allocation Space，以后的对象都会在Allocation Space上进行分配和释放
        * Allocation Space不是进程间共享的
      * 辅助数据结构：
        * Card Table
          * DVM Concurrent GC
        * Heap Bitmap
          * 两个Heap Bitmap
            * 一个用来记录上次GC存活的对象
            * 另外一个记录这次GC存活的对象
        * Mark Stack
          * DVM的运行时堆使用标记-清除
          * 用来遍历存活对象

  * DVM的GC日志

    * 打印到logcat中

    * ```
      <GC_Reason> <Amount_freed>, <Heap_stats>, <External_memory_stats>, <Pause_time>
      ```

* ART虚拟机

  * Android4.4发布的Android Runtime虚拟机
  * 替换DVM
  * Android5.0默认了ART，DVM退出历史舞台
  * ART和DVM的区别
    * ART会进行AOT（Ahead of time compilation）预编译，不需要JIT那样每次都编译
    * 运行效率更高，耗电变少
    * AOT会使安装时间变长
    * DVM是32位CPU
    * ART支持64位并兼容32位CPU
    * ART改进了GC，将GC暂停从2次变成1次
    * ART运行时空间划分和DVM不同

* ART的运行时堆

  * 默认采用Concurrent Mark Sweep GC回收算法
  * 堆的划分
    * Zygote Space（进程共享）
    * Allocation Space
    * Image Space（预加载类）（进程共享）
    * Large Object Space（分配一些大对象）
  * 垃圾收集器名称：
    * Concurrent Mark Sweep
    * Concurrent Partial Mark Sweep
    * Concurrent Sticky Mark Sweep
    * Marksweep + Semispace

* DVM和ART的诞生（Zygote中存储他们的实例，通过fork自身，不用每次启动应用程序进程都创建他们）

  * init启动Zygote时调用app_main.cpp的main函数
  * AndroidRuntime::start
  * jni_invocation.Init(NULL)
  * GetLibrary获取libart.so

#### 理解ClassLoader

* Java中的ClassLoader
  * Bootstrap ClassLoader
  * Extensions ClassLoader
  * Application ClassLoader
  * Custom ClassLoader
  * ![](imgs\classloader.png)
  * ClassLoader是一个抽象类
  * SecureClassLoader添加了安全权限
  * URLClassLoader实现了通过URL路径从jar文件和文件夹中加载类和资源
  * ExtClassLoader和AppClassLoader是Launcher的内部类
  * Launcher是Java虚拟机的入口
* 双亲委派原则
  * ![](imgs\sqwp.png)
  * 好处
    * 避免重复加载
    * 更加安全，不可以自定义一个String类来替代系统的String类
* 自定义ClassLoader
  * 实现加载网络上或者D盘某一文件中的jar包和Class文件
    1. 继承抽象类ClassLoader
    2. 复写findClass方法，并在findClass方法中调用defineClass方法

* Android中的ClassLoader
  * 系统类加载器
    * BootClassLoader
      * ClassLoader的内部类
      * 继承自ClassLoader
      * 单例
      * 系统启动时使用，来预加载常用类
    * PathClassLoader
      * 继承自BaseDexClassLoader
      * 加载已经安装的apk
    * DexClassLoader
      * 加载dex文件和压缩文件
      * 继承自BaseDexClassLoader
    * ![](imgs\android_classloader.png)
  * ClassLoader加载过程
    * ClassLoader
    * BaseDexClassLoader
    * DexPathList
    * Element
    * DexFile（defineClassNative）
  * BootClassLoader的创建
    * ZygoteInit.java中main的preload方法
    * 加载了/system/etc/preload-classes目录下的文件
    * 用classForName这个Native方法加载
  * PathClassLoader的创建
    * Zygote进程启动SystemServer进程时调用ZygoteInit的startSystemServer方法
    * 在handleSystemServerProcess方法中调用createPathClassLoader方法
    * PathClassLoaderFactory的createClassLoaders（工厂模式）
  * 自定义加载器

#### 热修复原理

* 代码修复
  * 类加载方案
    * 65536限制
      * 应用中引用的方法超过了最大数65536个
    * LinearAlloc限制
      * 固定的缓存区，当方法数超出了缓存区大小会报错
    * Dex分包
      * 打包时，将应用分成多个dex
        * 直接引用类放到主dex中
        * 次引用类放到次dex中
      * 缓解了65536和LinearAlloc限制问题
    * 类加载方案原理
      * 将新的Class打包成jar，然后放在Element数组的第一个，那么就会替换旧的Dex文件
      * ![](imgs\patch.png)
      * 类无法被卸载，所以需要重启App
      * 不能及时生效
    * 底层替换方案
      * 直接在Native层修改原有类
      * invoke底层实现
      * 修改ArtMethod的结构或者字段
* 资源修复
  * Instant Run
  * ![](imgs\instant.png)
  * 具体实现在MonkeyPatcher类中的monkeyPatchExistingResources方法中
    1. 创建新的AssetManager，通过反射调用addAssetPath方法加载外部的资源，这样新创建的AssetManager就含有了外部资源
    2. 将AssetManager类型的mAssets字段的引用全部替换为新创建的AssetManager
* 动态链接库修复
  * 重新加载so库
    * System的load方法
      * 传入参数是so的完整路径
    * System的LoadLibrary方法
      * 用于加载App安装后自动从apk包中复制到/data/data/packagename/lib下的so
    * 都是进入doLoad方法执行
    * 将so补丁插入到NativeLibraryElement数组前部
    * NativeLoad方法分析
      * Runtime_nativeLoad调用JVM_NativeLoad方法
    * LoadNativeLibrary函数总结
      * 判断so是否被加载过，避免重复加载
      * 打开so获得so句柄，如果so句柄获取失败，就返回false。创建新的SharedLibrary，如果传入path对应的library为空指针，就将创建的SharedLibrary赋值给library，并将library存储到libraries_中
      * 查找JNI_OnLoad的函数指针，根据不同情况设置was_successful的值，最终返回该was_successful
  * 调用System的load方法来接管so的加载入口
* ![](imgs\fix.png)

#### Hook技术

* 逆向工程
  * 静态分析
    * 源码分析
  * 动态分析
    * 调试分析
* ![](imgs\hook.png)

* 技术分类

  * API语言
    * Hook Java
    * Hook Native
  * 进程
    * 应用程序进程Hook
    * 全局Hook
  * 实现方式
    * 通过反射和代理实现，只能Hook当前的应用程序进程
    * 通过Hook框架实现，比如Xposed

* 代理模式（委托模式）（结构性设计模式）

  * 定义：为其他对象提供一种代理以控制对这个对象的访问成为代理模式

  * ![](imgs\proxy.png)

  * 例子：委托朋友买东西（静态代理）

    * 抽象主题类：

      * ```java
        public interface IShop{
            void buy();
        }
        ```

    * 真实主题类：

      * ```java
        public class LiuWangShu implements IShop{
            @Override
            public void buy(){
                System.out.println("购买");
            }
        }
        ```

    * 代理类：

      * ```java
        public class Purchasing implements IShop{
            private IShop mShop;
            public Purchasing(IShop shop){
                mShop = shop;
            }
            @Override
            public void buy(){
                mShop.buy();
            }
        }
        ```

    * 客户端类：

      * ```java
        public class Client{
            public static void main(String[] args){
                IShop liuwangshu = new LiuWangShu();
                IShop purchasing = new Purchasing(liuwangshu);
                purchasing.buy();
            }
        }
        ```

  * 动态代理（代码运行时通过反射来动态的生成代理类的对象）（运行时决定代理谁）

    * ```java
      public class DynamicPurchasing implements InvocationHandler{
          private Object obj;
          public DynamicPurchasing(Object obj){
              this.obj = obj;
          }
          @Override
          public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
              Object result = method.invoke(obj,args);
              if(method.getName().equals("buy")){
                  System.out.println("Liuwangshu在买买买");
              }
              return result;
          }
      }
      ```

    * ```java
      public class Client{
          public static void main(String[] args){
              //创建Liuwangshu
              IShop liuwangshu = new Liuwangshu();
              //创建动态代理
              DynamicPurchasing mDynamicPurchasing = new DynamicPurchasing(liuwangshu);
              ClassLoader loader = liuwangshu.getClass().getClassLoader();
              //动态创建代理类
              IShop purchasing = (IShop) Proxy.newProxyInstance(loader,new Class[]{IShop.class},mDynamicPurchasing);
              purchasing.buy();
          }
      }
      ```

    * 优点：

      * 不需要为每一个类去编写一个代理类
      * 抽象出一段通用代码给所有的被代理类

  * Hook startActivity方法

    * mInstrumentation.exeStartActivity方法来启动Activity

    * 用InstrumentationProxy替代原始的Instrumentation

    * ```java
      public class InstrumentationProxy extends Instrumentation{
          private static final TAG = "InstrumentationProxy";
          public InstrumentationProxy(Instrumentation instrumentation){
              mInstrumentation = instrumentation;
          }
          public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options){
              Log.d(TAG,"Hook 成功"+"---who:"+who);
              try{
                  //通过反射找到Instrumentation的execStartActivity方法
                  Method execStartActivity = Instrumentation.class.getDeclaredMethod("execStartActivity",Context.class,IBinder.class,IBinder.class,Activity.class,Intent.class,int.class,Bundle.class);
                  return (ActivityResult) execStartActivity.invoke(mInstrumentation, who, contextThread, token, target, intent, requestCode, options);
              }catch(Exception e){
                  throw new RuntimeException(e);
              }
          }
      }
      ```

    * ```java
      //MainActivity.java
      public class MainActivity extends Activity{
          @Override
          protected void onCreate(Bundle savedInstance){
              super.onCreate(savedInstance);
              setContentView(R.layout.activity_main);
              //替换增强版instrumentation类
              replaceActivityInstrumentation(this);
              Intent intent = new Intent(Intent.ACTION_VIEW);
              intent.setData(Uri.parse("http://liuwangshu.cn"));
              startActivity(intent);
          }
          
          public void replaceActivityInstrumentation(Activity activity){
              try{
                  //得到Activity的mInstrumentation字段
                  Field field = Activity.class.getDeclaredField("mInstrumentation");
                  field.setAccessible(true);
                  Instrumentation instrumentation = (Instrumentation) field.get(activity);
                  Instrumentation instrumentationProxy = new InstrumentationProxy(instrumentation);
                  field.set(activity,instrumentationProxy);
              }catch(Exception e){
                  e.printStackTrace();
              }
      	}
      }
      ```

  * Hook Context的startActivity方法

    * ```java
      //MainActivity.java
      public class MainActivity extends Activity{
          @Override
          protected void onCreate(Bundle savedInstance){
              super.onCreate(savedInstance);
              setContentView(R.layout.activity_main);
              //替换增强版instrumentation类
              replaceActivityInstrumentation();
              Intent intent = new Intent(Intent.ACTION_VIEW);
              intent.setData(Uri.parse("http://liuwangshu.cn"));
              //调用Context启动
              getApplicationContext().startActivity(intent);
          }
          
          public void replaceActivityInstrumentation(){
              try{
                  //获取ActivityThread类
                  Class<?> activityThreadClazz = Class.forName("android.app.ActivityThread");
                  Field activityThreadField = activityThreadClazz.getDeclaredField("sCurrentActivityThread");
                  activityThreadField.setAccessible(true);
                  Object currentActivityThread = activityThreadField.get(null);
                  //得到Activity的mInstrumentation字段
                  Field mInstrumentationField = activityThreadClazz.class.getDeclaredField("mInstrumentation");
                  mInstrumentationField.setAccessible(true);
                  Instrumentation instrumentation = (Instrumentation) mInstrumentationField.get(currentActivityThread);
                  Instrumentation mInstrumentationProxy = new InstrumentationProxy(mInstrumentation);
                  mInstrumentationField.set(currentActivityThread,mInstrumentationProxy);
              }catch(Exception e){
                  e.printStackTrace();
              }
      	}
      }
      ```

#### 插件化原理

* 动态加载技术
  * 原理
    * 动态加载一些程序中本来不存在的可执行文件并运行
  * 衍生技术
    * 热修复技术
      * 修复Bug
    * 插件化技术
      * 实现功能模块的解耦
  
* 产生
  1. 业务复杂，模块耦合
  2. 应用间的接入
  3. 65536限制，内存占用大
  4. ![](imgs\plug.png)

* 主流框架
  * VirtualApk比较全面
  * 如果加载的插件不需要和宿主有任何耦合，也无需和宿主进行通信，比如加载第三方App，那么推荐使用RePlugin

* Activity 插件化

  * 反射实现

  * 接口实现

  * Hook技术实现

    * Hook IActivityManager方案实现

      * 注册Activity进行占坑

        * ```xml
          <!--AndroidManifest.xml-->
          <application android:name="com.xxx.xxx.MyApplication">
          	<activity android:name=".StubActivity"/>
          </application>
          ```

      * 使用占坑Activity通过AMS校验

        * 用StubActivity替换TargetActivity
        * Android7.0和Android8.0的区别
          * 8.0使用IActivityManager，直接采用AIDL进行通信
          * 7.0的Activity的启动会调用ActivityManagerNative的getDefault方法

      * 实现：

        * ```java
          //Singleton.java
          //在Android框架中定义
          public abstract class Singleton<T>{
              private T mInstance;
              protected abstract T create();
              public final T get(){
                  synchronized(this){
                      if(mInstance == null){
                          mInstance = create();
                      }
                      return mInstance;
                  }
              }
          }
          ```

        * ```java
          //反射工具类FieldUtil
          public class FieldUtil{
              public static Object getField(Class clazz, Object target, String name) throws Exception{
                  Field field = clazz.getDeclaredField(name);
                  field.setAccessible(true);
                  return field.get(target);
              }
              
              public static Field getField(Class clazz, String name) throws Exception{
                  Field field = clazz.getDeclaredField(name);
                  field.setAccessible(true);
                  return field;
              }
              
              public static void setField(Class clazz, Object target, String name, Object value) throws Exception{
                  Field field = clazz.getDeclaredField(name);
                  field.setAccessible(true);
                  field.set(target, value);
              }
          }
          ```

        * ```java
          public class IActivityManagerProxy implements InvocationHandler{
              private Object mActivityManager;
              private static final String TAG = "IActivityManagerProxy";
              public IActivityManagerProxy(Object activityManager){
                  this.mActivityManager = activityManager;
              }
              @Override
              public Object invoke(Object o, Method method, Object[] args) throws Throwable{
                  if("startActivity".equals(method.getName())){
                      //替换args中Intent的内容，将它指向StubActivity
                      Intent intent = null;
                      int index = 0;
                      for(int i = 0; i < args.length; i++){
                          if(args[i] instanceof Intent){
                              index = i;
                              break;
                          }
                      }
                      intent = (Intent) args[index];
                      Intent subIntent = new Intent();
                      String packageName = "com.example.liuwangshu.pluginactivity";
                      //存储原先的Activity, 可以还原
                      subIntent.setClassName(packageName,packageName+".StubActivity");
                      subIntent.putExtra(HookHelper.TARGET_INTENT, intent);
                      //替换
                      args[index] = subIntent;
                  }
                  return method.invoke(mActivityManager, args);
              }
          }
          ```

        * ```java
          public class HookHelper{
              public static final String TARGET_INTENT = "target_intent";
              public static void hookAMS() throws Exception{
                  Object defaultSingleton = null;
                  //Android8.0逻辑
                  if(Build.VERSION.SDK_INT >= 26){
                      Class<?> activityManageClazz = Class.forName("android.app.ActivityManager");
                      //获取activityManager中的IActivityManagerSingleton字段
                      defaultSingleton = FieldUtil.getField(activityManageClazz, null, "IActivityManagerSingleton");
                  }else{
                      //Android7.0逻辑
                      Class<?> activityManagerNativeClazz = Class.forName("android.app.ActivityManagerNative");
                      //获取ActivityManagerNative中的gDefault字段
                      defaultSingleton = FieldUtil.getField(activityManagerNativeClazz, null, "gDefault");
                  }
                  Class<?> singletonClazz = Class.forName("android.util.Singleton");
                  Field mInstanceField = FieldUtil.getField(singletonClazz,"mInstance");
                  //获取iActivityManager
                  Object iActivityManager = mInstanceField.get(defaultSingleton);
                  Class<?> iActivityManagerClazz =  Class.forName("android.app.IActivityManager");
                  Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),new Class<?>[] { iActivityManagerClazz}, new IActivityManagerProxy(iActivityManager));
                  mInstanceField.set(defaultSingleton, proxy);
              }
          }
          ```

        * ```java
          public  class MyApplication extends Application{
              @Override
              public void attachBaseContext(Context base){
                  super.attachBaseContext(base);
                  try{
                      HookHelper.hookAMS();
                  }catch(Exception e){
                   	e.printStackTrace();   
                  }
              }
          }
          ```

        * ```java
          public class MainActivity extends Activity{
              private Button bt_hook;
              @Override
              protected void onCreate(Bundle savedInstance){
                  super.onCreate(savedInstance);
                  setContentView(R.layout.activity_main);
                  bt_hook = (Button) this.findViewById(R.id.bt_hook);
                  bt_hook.setOnClickListener(()->{
                      Intent intent = new Intent(MainActivity.class, TargetActivity.class);
                      startActivity(intent);
                  });
              }
          }
          ```
          

      * 还原插件

        * 我们需要启动的还是插件TargetActivity，所以还要用target去替换Stub

        * ```java
          public class HCallback implements Handler.Callback{
              public static final int LAUNCH_ACTIVITY = 100;
              Handler mHandler;
              public HCallback(Handler handler){
                  mHandler = handler;
              }
              @Override
              public boolean handleMessage(Message msg){
                  if(msg.what == LAUNCH_ACTIVITY){
                      Object r = msg.obj;
                      try{
                          //得到消息中的Intent（启动StubActivity的Intent）
                          Intent intent = (Intent) FieldUtil.getField(r.getClass(), r, "intent");
                          //得到此前保存起来的Intent（启动TargetActivity的Intent）
                          Intent target = intent.getParcelableExtra(HookHelper.TARGET_INTENT);
                          //将启动SubActivity的Intent替换为启动TargetActivity的Intent
                          intent.setComponent(target.getComponent());
                      }catch(Exception e){
                          e.printStackTrace();
                      }
                  }
                  mHandler.handleMessage(msg);
                  return true;
              }
          }
          ```

        * ```java
          public class HookHelper{
              public static final String TARGET_INTENT = "target_intent";
              public static void hookHandler() throws Exception{
                  Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
                  //获取ActivityThread自己的静态变量
                  Object currentActivityThread = FieldUtil.getField(activityThreadClass, null, "sCurrentActivityThread");
                  //获取ActivityThread的变量mH
                  Field mHField = FieldUtil.getField(activityThreadClass,"mH");
                  Handler mH = (Handler) mHField.get(currentActivityThread);
                  FieldUtil.setField(Handler.class,mH,"mCallback",new HCallback(mH));
              }
          }
          ```

        * 这样启动的插件还是Target，只不过中间的逻辑经过了我们的拦截

      * 插件Activity的生命周期

    * Hook Instrumentation方案实现

      * ```java
        public class InstrumentationProxy extends Instrumentation{
            private Instrumentation mInstrumentation;
            private PackageManager mPackageManager;
            public InstrumentationProxy(Instrumentation instrumentation, PackageManager packageManager){
                mInstrumentation = instrumentation;
                mPackageManager = packageManager;
            }
            public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options){
                //判断Activity是否在AndroidManifest.xml中注册了
                List<ResolveInfo> infos = mPackageManager.queryIntentActivities(intent, PackageManager.MATCH_ALL);
                if(infos == null || infos.size() == 0){
                    //将要启动的Activity的ClassName保存起来用于后面还原TargetActivity
                    intent.putExtra(HookHelper.TARGET_INTENT, intent.getComponent().getClassName());
                    //替换要启动的Activity为StubActivity
                    intent.setClassName(who, "com.example.liuwangshu.pluginactivity.StubActivity");
                }
                try{
                    //反射调用execStartActivity
                    Method execMethod = Instrumentation.class.getDeclaredMethod("execStartActivity", Context.class, IBinder.class, IBinder.class, Activity.class, Intent.class, int.class, Bundle.class);
                    return (ActivityResult) execMethod.invoke(mInstrumentation, who, contextThread, token, target, intent, requestCode, options);
                }catch(Exception e){
                    e.printStackTrace();
                }
                return null;
            }
            
            //还原TargetActivity
            public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
                String intentName = intent.getStringExtra(HookHelper.TARGET_INTENT_NAME);
                if(!TextUtils.isEmpty(intentName)){
                    return super.newActivity(cl, intentName, intent);
                }
                return super.newActivity(cl, className, intent);
            }
        }
        ```

      * ```java
        public class HookHelper{
            public static void hookInstrumentation(Context context) throws Exception{
                Class<?> contextImplClass = Class.forName("android.app.ContextImpl");
                Field mMainThreadField = FieldUtil.getField(contextImplClass, "mMainThread");
                Object activityThread = mMainThreadField.get(context);
                Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
                Field mInstrumentationField = FieldUtil.getField(activityThreadClass, "mInstrumentation");
                FieldUtil.setField(activityThreadClass, activityThread, "mInstrumentation", new InstrumentationProxy((Instrumentation) mInstrumentationField.get(activityThread), context.getPackageManager()));
            }
        }
        ```

* Service插件化

  * 和Activity插件化方面的不同

    * Activity是基于栈管理的，一个栈中的Activity的数量不多，因此插件化框架处理的Activity数量有限
    * Standard模式下启动Activity可以创建多个实例，但是Service就是一个
    * Service的生命周期不受用户影响，而Activity会

  * 代理分发实现

    * 核心：保证优先级（先启动代理Service，再在它的onStartCommand等方法里进行转发，最后执行TargetService的onCreate方法）

      1. 启动代理Service

         1. 注册代理ProxyService在AndroidManifest.xml

         ```xml
         <application>
         	<service android:name=".ProxyService"/>
         </application>
         ```

         2. 在MainActivity当其中启动这个Service

            ```java
            Intent intent = new Intent(MainActivity.this,TargetService.class);
            startService(intent);

         3. 编写TargetService

            ```java
            public class TargetService extends Service{
                private static final String TAG = "TargetService";
                @Nullable
                @Override
                public IBinder onBind(Intent intent){
                    return null;
                }
            	@Override
                public void onCreate(){
                    super.onCreate();
                    Log.d(TAG, "onCreate");
                }
                @Override
                public int onStartCommand(Intent intent, int flags, int startId){
                    Log.d(TAG,"onStartCommand");
                    return super.onStartCommand(intent, flag, startId);
                }
            }
            ```

         4. Hook IActivityManager

            ```java
            public class IActivityManagerProxy implements InvocationHandler{
                private Object mActivityManager;
                private static final String TAG = "IActivityManagerProxy";
                public IActivityManagerProxy(Object activityManager){
                    this.mActivityManager = activityManager;
                }
                @Override
                public Object invoke(Object o, Method method, Object[] args) throws Throwable{
                    if("startService".equals(method.getName())){
                        //替换args中Intent的内容，将它指向StubActivity
                        Intent intent = null;
                        int index = 0;
                        for(int i = 0; i < args.length; i++){
                            if(args[i] instanceof Intent){
                                index = i;
                                break;
                            }
                        }
                        intent = (Intent) args[index];
                        Intent subIntent = new Intent();
                        String packageName = "com.example.liuwangshu.pluginservice";
                        //存储原先的Service, 可以分发
                        subIntent.setClassName(packageName,packageName+".ProxyService");
                        subIntent.putExtra(HookHelper.TARGET_SERVICE, intent.getComponent().getClassName());
                        //替换
                        args[index] = subIntent;
                        Log.d(TAG,"Hook成功");
                    }
                    return method.invoke(mActivityManager, args);
                }
            }
            ```

         5. HookHelper

            ```java
            public class HookHelper{
                public static final String TARGET_INTENT = "target_intent";
                public static void hookAMS() throws Exception{
                    Object defaultSingleton = null;
                    //Android8.0逻辑
                    if(Build.VERSION.SDK_INT >= 26){
                        Class<?> activityManageClazz = Class.forName("android.app.ActivityManager");
                        //获取activityManager中的IActivityManagerSingleton字段
                        defaultSingleton = FieldUtil.getField(activityManageClazz, null, "IActivityManagerSingleton");
                    }else{
                        //Android7.0逻辑
                        Class<?> activityManagerNativeClazz = Class.forName("android.app.ActivityManagerNative");
                        //获取ActivityManagerNative中的gDefault字段
                        defaultSingleton = FieldUtil.getField(activityManagerNativeClazz, null, "gDefault");
                    }
                    Class<?> singletonClazz = Class.forName("android.util.Singleton");
                    Field mInstanceField = FieldUtil.getField(singletonClazz,"mInstance");
                    //获取iActivityManager
                    Object iActivityManager = mInstanceField.get(defaultSingleton);
                    Class<?> iActivityManagerClazz =  Class.forName("android.app.IActivityManager");
                    Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),new Class<?>[] { iActivityManagerClazz}, new IActivityManagerProxy(iActivityManager));
                    mInstanceField.set(defaultSingleton, proxy);
                }
            }
            ```

         6. MyApplication

            ```java
            public  class MyApplication extends Application{
                @Override
                public void attachBaseContext(Context base){
                    super.attachBaseContext(base);
                    try{
                        HookHelper.hookAMS();
                    }catch(Exception e){
                     	e.printStackTrace();   
                    }
                }
            }
            ```

      2. 代理分发

         ```java
         //返回START_STICKY为了重新创建并执行onStartCommand方法
         public class ProxyService extends Service{
             public static final String TARGET_SERVICE = "target_service";
             
             @Override
             public int onStartCommand(Intent intent, int flags, int startId){
                 if(null == intent || !intent.hasExtra(TARGET_SERVICE)){
                     return START_STICKY;
                 }
                 String serviceName = intent.getStringExtra(TARGET_SERVICE);
                 if(null == serviceName){
                     return START_STICKY;
                 }
                 Service targetService = null;
                 try{
                     Class activityThreadClazz = Class.forName("android.app.ActivityThread");
                     Method getActivityThreadMethod = activityThreadClazz.getDeclaredMethod("getApplicationThread");
                     getActivityThreadMethod.setAccessible(true);
                     Object activityThread = FieldUtil.getField(activityThreadClazz, null, "sCurrentActivityThread");
                     Object applicationThread = getActivityThreadMethod.invoke(activityThread);
                     Class iInterfaceClazz = Class.forName("android.os.IInterface");
                     Method asBinderMethod = iInterfaceClazz.getDeclaredMethod("asBinder");
                     asBinderMethod.setAccessible(true);
                     Object token = asBinderMethod.invoke(applicationThread);
                     Class serviceClazz = Class.forName("android.app.Service");
                     Method attachMethod = serviceClazz.getDeclared("attach",Context.class, activityThreadClazz, String.class, IBinder.class, Application.class, Object.class);
                     attachMethod.setAccessible(true);
                     Object defaultSingleton = null;
                     //Android8.0逻辑
                     if(Build.VERSION.SDK_INT >= 26){
                         Class<?> activityManageClazz = Class.forName("android.app.ActivityManager");
                         //获取activityManager中的IActivityManagerSingleton字段
                         defaultSingleton = FieldUtil.getField(activityManageClazz, null, "IActivityManagerSingleton");
                     }else{
                         //Android7.0逻辑
                         Class<?> activityManagerNativeClazz = Class.forName("android.app.ActivityManagerNative");
                         //获取ActivityManagerNative中的gDefault字段
                         defaultSingleton = FieldUtil.getField(activityManagerNativeClazz, null, "gDefault");
                     }
                     Class<?> singletonClazz = Class.forName("android.util.Singleton");
                     Field mInstanceField = FieldUtil.getField(singletonClazz,"mInstance");
                     //获取iActivityManager
                     Object iActivityManager = mInstanceField.get(defaultSingleton);
                     targetService = (Service) Class.forName(serviceName).newInstance();
                     attachMethod.invoke(targetService, this, activityThread, intent.getComponent().getClassName(), token, getApplication(), iActivityManager);
                     targetService.onCreate();
                 }catch(Exception e){
                     e.printStackTrace();
                     return START_STICKY;
                 }
                 targetService.onStartCommand(intent, flags, startId);
                 return START_STICKY;
             }
         }
         ```

* ContentProvider插件化

  * 分析滴滴的VirtualApk如何编写

    * ContentProvider启动分析

      * ContentProvider的query方法调用ActivityThread的acquireProvider方法
      * 先获取IContentProvider，如果通过mProviderMap查到了，直接返回
      * 如果没有，通过AMS的getContentProvider方法获取ContentProviderHolder（包含了IContentProvider类型数据）
      * IContentProvider是一个Binder对象用于进程间通信
      * AMS启动ContentProvider通过ActivityThread向H类发送消息，将代码逻辑运行在主线程中
      * 最终调用ContentProvider的onCreate方法

    * 实现

      * 需要注册一个真正的ContentProvider作为代理ContentProvider

      * VirtualAPK的初始化

        * MainActivity中 this.loadPlugin(this);
        * PluginManager.getInstance(base);（使用了双重校验锁）
        * PluginManager.loadPlugin(apk)
        * Hook IActivityManager
        * LoadedPlugin的代码比较多，主要创建一些类型的对象，存储4大组件相关的数据结构

      * 启动代理ContentProvider

        * ```java
          Cursor bookCursor = getContentResolver().query(bookUri, new String[]{"_id", "name"}, null, null, null);
          ```

        * ```java
          @Override
          public ContentResolver getContentResolver(){
              return new PluginContentResolver(getHostContext());
          }
          ```

        * PluginContentResolver复写了acquireUnstableProvider方法

        * ```java
          protected IContentProvider acquireUnstableProvider(Context context, String auth){
              try{
                  //查找是否有匹配的ContentProvider
                  if(mPluginManager.resolveContentProvider(auth, 0) != null){
                      return mPluginManager.getIContentProvider();
                  }
                  //没有就调用系统的ContentProvider
                  return (IContentProvider) sAcquireUnstableProvider.invoke(mBase, context, auth);
              }catch(Exception e){
                  e.printStackTrace();
              }
          }
          ```

        * ```java
          public synchronized IContentProvider getIContentProvider(){
              if(mIContentProvider == null){
                  hookIContentProviderAsNeeded();
              }
              return mIContentProvider;
          }
          ```

        * hookIContentProviderAsNeeded

          * 获取插件ContentResolver的Uri
          * 得到IContentProvider
          * 反射得到ActivityThread实例
          * 反射获取mProviderMap
          * 反射获得IContentProvider proxy = IContentProviderProxy.newInstance(mContext, rawProvider);
          * mIContentProvider = proxy; （用代理IContentProviderProxy替换IContentProvider）
          * WrapperUri实现了替换Uri的操作

      * 代理分发

        * 注册代理ContentProvider在AndroidManifest.xml

          ```xml
          <provider
                    android:name="com.didi.virtualapk.delegate.RemoteContentProvider"
                    android:authorities="${applicationId}.VirtualAPK.Provider"
                    android:process=":daemon">
          </provider>
          ```

        * 调用ContentProvider的query方法，实际会调用RemoteContentProvider的query方法

        * RemoteContentProvider的getContentProvider

        * 先从缓存中通过auth读取对应的插件ContentProvider

        * PluginManager.loadPlugin加载插件APK

        * 从已加载的APK中得到匹配的auth的ProviderInfo

        * ProviderInfo创建ContentProvider

        * ContentProvider的attachInfo方法，最终会调用onCreate方法

        * 将插件加入缓存，防止重复创建插件

* BroadcastReceiver插件化

  * 将静态注册的BroadcastReceiver全部转换为动态注册来处理

  * VirtualAPK实现

    * ```java
      LoadedPlugin(PluginManager pluginManager, Context context, File apk) throws PackageParserException{
          //存储插件在静态注册的BroadcastReceiver信息
          Map<ComponentName, ActivityInfo> receivers = new HashMap<ComponentName, ActivityInfo>();
          for(PackageParser.Activity receiver : this.mPackage.receivers){
              //存储
              receivers.put(receiver.getComponentName(), receiver.info);
              try{
                  //根据类名，加载类
                  BroadcastReceiver br = BroadcastReceiver.class.cast(getClassLoader().loadClass(receiver.getComponentName().getClassName()).newInstance());
                  for(PackageParser.ActivityIntentInfo aii : receiver.intents){
                      //调用宿主Context完成插件BroadcastReceiver的注册
                      this.mHostContext.registerReceiver(br,aii);
                  }
              }catch(Exception e){
                  e.printStackTrace();
              }
          }
          this.mReceiverInfos = Collections.unmodifiableMap(receivers);
          this.mPackageInfo.receivers = receivers.values().toArray(new ActivityInfo[receivers.size()]);
      }
      ```

* 资源的插件化

  * 系统资源加载

    * 启动Activity时会调用performLaunchActivity方法，其内部会调用LoadedApk的makeApplication方法

    * ```java
      //创建应用的Context
      ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
      //创建application
      app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
      ```

    * ```java
      //将资源赋值给ContextImpl
      context.setResources(packageInfo.getResources());
      ```

    * ```java
      public Resources getResources(){
          mResources = ResourcesManager.getInstance().getResources(...);
          return mResources;
      }
      ```

    * ```java
      private @Nullable Resources getOrCreateResources(...){
          //实现Resources
          ResourcesImpl resourcesImpl = createResourcesImpl(key);
          
          //创建resources
          resources = getOrCreateResourcesForActivityLocked(...)
      }
      ```

    * ```java
      private @Nullable ResourceImpl createResourcesImpl(ResourcesKey key){
          //创建资源管理器
          final AssetManager assets = createAssetManager(key);
          //新建ResourcesImpl对象
          final ResourcesImpl impl = new ResourcesImpl(assets, dm, config, daj);
      }
      ```

  * VirtualApk实现

    * 两种方案

      * 合并资源
        * 将插件的资源全部添加到宿主的Resources中
        * 可以访问宿主的资源
      * 构建插件资源
        * 每个插件都能构造出独立的Resources
        * 不可以访问宿主的资源

    * 实现：

      * ```java
        //LoadedPlugin
        @WorkerThread
        private static Resources createResources(Context context, File apk){
            //合并资源
            if(Constants.COMBINE_RESOURCES){
                Resources resources = ResourcesManager.createResources(context, apk.getAbsolutePath());
                //用新的resources替换
                ResourceManager.hookResources(context, resources);
                return resources;
            }else{
                //构建插件资源
                Resources hostResources = context.getResources();
                AssetManager assetManager = createAssetManager(context, apk);
                return new Resources(assetManager, hostResources.getDisplayMetrics(), hostResources.getConfiguration());
            }
        }
        ```

      * ```java
        private static AssetManager createAssetManager(Context context, File apk){
            try{
                AssetManager am = AssetManager.class.newInstance();
                //通过addAssetPath来加载插件
                ReflectUtil.invoke(AssetManager.class, am, "addAssetPath", apk.getAbsolutePath());
                return am;
            }catch(Exception e){
                e.printStackTrace();
                return null;
            }
        }
        ```

* So的插件化

  * 将so补丁插入到NativeLibraryElement数组的前部
  * 调用System的load方法来接管so的加载入口
  * VirtualApk实现
    * 加载DexClassLoader
    * 将宿主和插件的DexElement合并得到allDexElements，并通过反射用allDexElements替换dexElements
    * 避免重复插入so
    * 获取宿主的PathList
    * Android5.1以上情况
      * 得到宿主存储so文件的List集合nativeLibraryDirectories
      * 插件存储so文件添加在nativeLibraryDirectories
      * 获取宿主的nativeLibraryElement和插件的nativeLibraryElement
      * 得到插件的newNativeLibraryPathElements
      * 创建allNativeLibraryPathElements，将baseNativeLibraryPathElements复制到allNativeLibraryPathElements
      * 遍历newNativeLibraryPathElements，将so添加到allNativeLibraryPathElements
      * 最后通过反射allNativeLibraryPathElements替换nativeLibraryPathElements

#### 绘制优化

* 绘制原理
  * framework层
    * measure
    * layout
    * draw
  * native层
    * surfaceFlinger服务完成
  * CPU进行Measure, layout, record, execute的数据计算工作
  * GPU负责栅格化，渲染
  * CPU和GPU通过图形驱动层进行连接，图形驱动层维护了一个队列，CPU将display list添加到队列，GPU取出数据进行绘制
  * 60FPS指的是一秒钟传输图片的量
  * 需要每16.6667ms刷新一次
  * Android每16ms发出VSYNC信号，触发UI进行渲染
  * 卡顿的原因
    * 布局layout太复杂
    * 同一时间动画执行次数过多
    * view过度绘制，导致某些像素同一帧时间被绘制多次
    * UI线程中做了耗时操作
    * GC回收的停顿时间
* Profile GPU Rendering
  * Android 4.1提供的开发辅助功能（在开发者选项中打开）
  * <img src="imgs\profile.png" width = "250" height = "400" alt="" align=center />
  * **橙色**代表处理的时间，是CPU告诉GPU渲染一帧的地方，这是一个阻塞调用，因为CPU会一直等待GPU发出接到命令的回复，如果橙色柱状图很高，则表明GPU很繁忙。
  * **红色**代表执行的时间，这部分是Android进行2D渲染 Display List的时间。如果红色柱状图很高，可能是由重新提交了视图而导致的。还有复杂的自定义View也会导致红的柱状图变高。
  * **蓝色**代表测量绘制的时间，也就是需要多长时间去创建和更新DisplayList。如果蓝色柱状图很高，可能是需要重新绘制，或者View的onDraw方法处理事情太多。

![VKJ8mj.png](https://s2.ax1x.com/2019/05/30/VKJ8mj.png)

- Swap Buffers：表示处理的时间，和上面讲到的橙色一样。
- Command Issue：表示执行的时间，和上面讲到的红色一样。
- Sync & Upload：表示的是准备当前界面上有待绘制的图片所耗费的时间，为了减少该段区域的执行时间，我们可以减少屏幕上的图片数量或者是缩小图片的大小。
- Draw：表示测量和绘制视图列表所需要的时间，和上面讲到的蓝色一样。
- Measure/Layout：表示布局的onMeasure与onLayout所花费的时间，一旦时间过长，就需要仔细检查自己的布局是不是存在严重的性能问题。
- Animation：表示计算执行动画所需要花费的时间，包含的动画有ObjectAnimator，ViewPropertyAnimator，Transition等。一旦这里的执行时间过长，就需要检查是不是使用了非官方的动画工具或者是检查动画执行的过程中是不是触发了读写操作等等。
- Input Handling：表示系统处理输入事件所耗费的时间，粗略等于对事件处理方法所执行的时间。一旦执行时间过长，意味着在处理用户的输入事件的地方执行了复杂的操作。
- Misc Time/Vsync Delay：表示在主线程执行了太多的任务，导致UI渲染跟不上VSYNC的信号而出现掉帧的情况。

* Systrace

  * Android 4.1中新增的性能数据采样和分析工具

  * 跟踪系统IO，内核，工作队列，CPU负载

  * DDMS中使用Systrace

  * 用命令行使用Systrace

    * ```shell
      python systrace.py --time=10 -o newtrace.html sched gfx view wm
      ```

  * 代码中使用Systrace

    * ```java
      ...
       private final Runnable mUpdateChildViewsRunnable = new Runnable() {
              public void run() {
                  if (!mFirstLayoutComplete) {
                      return;
                  }
                  if (mDataSetHasChangedAfterLayout) {
                      //开始
                      TraceCompat.beginSection(TRACE_ON_DATA_SET_CHANGE_LAYOUT_TAG);
                      dispatchLayout();
                      //结束
                      TraceCompat.endSection();
                  } else if (mAdapterHelper.hasPendingUpdates()) {
                      TraceCompat.beginSection(TRACE_HANDLE_ADAPTER_UPDATES_TAG);
                      eatRequestLayout();
                      mAdapterHelper.preProcess();
                      if (!mLayoutRequestEaten) {
                          rebindUpdatedViewHolders();
                      }
                      resumeRequestLayout(true);
                      TraceCompat.endSection();
                  }
              }
          };
          ...
      ```

  * 用Chrome分析Systrace

    * Alert区域
      * ![VKJUhV.png](https://s2.ax1x.com/2019/05/30/VKJUhV.png)
    * CPU区域
      * ![VKJdpT.png](https://s2.ax1x.com/2019/05/30/VKJdpT.png)
      * ![VKJNt0.png](https://s2.ax1x.com/2019/05/30/VKJNt0.png)
    * 应用区域
      * ![VKJrnJ.png](https://s2.ax1x.com/2019/05/30/VKJrnJ.png)
        * 绿色F表示流畅，黄色F和红色F都表示渲染时间超过16ms
      * ![VKJw1U.png](https://s2.ax1x.com/2019/05/30/VKJw1U.png)
    * Alerts总体分析
      * ![VKJsB9.png](https://s2.ax1x.com/2019/05/30/VKJsB9.png)

* Traceview

  * Android SDK中自带的工具

    * 单次执行耗时的方法
    * 执行次数多的方法

  * 使用

    * DDMS中使用

    * 代码中加入

      * ```java
        Debug.startMethodTracing();
        ...
        Debug.stopMethodTracing();
        ```

      * ```xml
        <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
        ```

  * 分析

    * 为了分析Traceview，我们来举一个简单的例子来生成trace文件，这里采用第二种方式：代码中加入调试语句。代码如下所示。

    * ```java
      public class CoordinatorLayoutActivity extends AppCompatActivity {
          private ViewPager mViewPager;
          private TabLayout mTabLayout;
          @Override
          protected void onCreate(Bundle savedInstanceState) {
              super.onCreate(savedInstanceState);
              setContentView(R.layout.activity_tab_layout);
              //开始
              Debug.startMethodTracing("test");
              initView();
         ...
          }
          private void initView() {
              try {
                  Thread.sleep(1000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          }
          @Override
          protected void onStop() {
              super.onStop();
              //结束
              Debug.stopMethodTracing();
          }
      }
      ```

    * ![VKJBX4.png](https://s2.ax1x.com/2019/05/30/VKJBX4.png)

    * 我们进入traceview所在的目录(直接将traceview.bat拖入到cmd中)，并执行上图的traceview语句后会弹出Traceview视图，它分为两部分，分别是时间片面板和分析面板，我们先来看时间片面板，如下图所示。

    * ![VKJ2h6.png](https://s2.ax1x.com/2019/05/30/VKJ2h6.png)

    * 其中x轴代表时间的消耗，单位为ms，y轴代表各个线程。一般会查看色块的长度，明显比较长的方法重点去关注，具体的分析还得看分析面板，如下图所示。

    * ![VKJcA1.png](https://s2.ax1x.com/2019/05/30/VKJcA1.png)

    * 每一列数据的代表的含义如下表所示。

      * | 列名                        | 含义                                               |
        | --------------------------- | -------------------------------------------------- |
        | Name                        | 该线程运行过程中调用的函数名                       |
        | Incl Cpu Time%              | 某个方法包括其内部调用的方法所占用CPU时间百分比    |
        | Excl Cpu Time%              | 某个方法不包括其内部调用的方法所占用CPU时间百分比  |
        | Incl Real Time%             | 某个方法包括其内部调用的方法所占用真实时间百分比   |
        | Excl Real Time%             | 某个方法不包括其内部调用的方法所占用真实时间百分比 |
        | Calls + Recur Calls / Total | 某个方法次数+递归调用次数                          |
        | Cpu Time / Call             | 该方法平均占用CPU时间                              |
        | Cpu Time / Call             | 该方法平均占用真实时间                             |
        | Incl Cpu Time               | 某个方法包括其内部调用的方法所占用CPU时间          |
        | Excl Cpu Time               | 某个方法不包括其内部调用的方法所占用CPU时间        |
        | Incl Real Time              | 某个方法包括其内部调用的方法所占用真实时间         |
        | Excl Real Time              | 某个方法不包括其内部调用的方法所占用真实时间       |

    * 因为我们用sleep方法来进行耗时操作，所以这里我们可以单击Incl Real Time来进行降序排列。其中有很多系统调用的方法，我们来进行一一过滤。最终我们发现了CoordinatorLayoutActivity的initView方法Incl Real Time的时间为1000.493ms，这显然有问题，如下图所示

    * ![VKJgtx.png](https://s2.ax1x.com/2019/05/30/VKJgtx.png)

    * 从图中我们可以看出是调用sleep方法导致的耗时

#### 布局优化

* 工具

  * Hierarchy Viewer

    * Android SDK自带的可视化的调试工具，用来检查布局嵌套和绘制的时间
    * ![VKJO9f.png](https://s2.ax1x.com/2019/05/30/VKJO9f.png)
    * Windows：当前设备所有界面列表。
    * Tree View：将当前Activity的所有View的层次按照高层到低层从左到右显示出来。
    * Tree Overview：全局概览，以缩略的形式显示。
    * Layout View：整体布局图，以手机屏幕上真实的位置呈现出来。单击某一个控件，会在Tree Overview窗口中显示出对应的控件。
    * ![VKJvjg.png](https://s2.ax1x.com/2019/05/30/VKJvjg.png)点击这个按钮可以查看这个布局的view的measure, layout,draw的耗时
    * ![VKJjgS.png](https://s2.ax1x.com/2019/05/30/VKJjgS.png)
    * 被选中的LinearLayout给出了自身Measure、Layout和Draw的耗时，并且它所包含的View中都有了三个指示灯，分别代表当前View在Measure、Layout和Draw的耗时，绿色代表比其他50%View的同阶段（比如Measure阶段）速度要快，黄色则代表比其他50%View同阶段速度要慢，红色则代表比其他View同阶段都要慢，则需要注意了。如果想要看View的具体耗时，则点击该View就可以了。

  * Android Lint

    * 代码扫描工具
    * 我们可以通过Android Studio的Analyze->Inspect Code来配置检查的范围，如下图所示。
    * ![VKJq4P.png](https://s2.ax1x.com/2019/05/30/VKJq4P.png)
    * 图中列出了项目中出现的问题种类，以及每个问题种类的个数，问题种类包括我们前面提到的Correctness 、Internationalization 、Performance等。我们点击展开最后的XML一项，点击一个问题，就会出现如下图的提示。
    * ![VKYSBj.png](https://s2.ax1x.com/2019/05/30/VKYSBj.png)

  * 布局优化方法

    * #### **合理运用布局**

      * 一般情况下，但是如果布局层数较多时，推荐用RelativeLayout来实现。如果布局嵌套较多，推荐使用LinearLayout来实现。

    * #### **使用Include标签来进行布局复用**

      * ```xml
        <include layout="@layout/titlebar" />
        ```

    * #### **用Merge标签去除多余层级**

      * ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <merge xmlns:android="http://schemas.android.com/apk/res/android"
            android:layout_width="match_parent"
            android:layout_height="40dp"
            android:background="@android:color/darker_gray">
                                
            <ImageView
                android:layout_width="30dp"
                android:layout_height="30dp"
                android:src="@drawable/ico_left"
                android:padding="3dp"
                android:layout_gravity="center"/>
        
            <TextView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_gravity="center"
                android:gravity="center"
                android:text="绘制优化" />
        </merge>
        ```

      * Merge标签可以去除一层布局

      * 但是这里用merge标签来替代LinearLayout会导致LinearLayout失效，布局就会错乱。merge标签最好是来替代FrameLayout，或者是布局方向一致的LinearLayout，比如当前父布局LinearLayout的布局方向是垂直的，包含的子布局LinearLayout的布局方向也是垂直的则可以用merge标签

    * #### **使用ViewStub来提高加载速度**

      * ViewStub是轻量级的View，不可见并且不占布局位置。当ViewStub调用inflate方法或者设置可见时，系统会加载ViewStub指定的布局，然后将这个布局添加到ViewStub中，在对ViewStub调用inflate方法或者设置可见之前，它是不占布局空间和系统资源的，它主要的目的就是为目标视图占用一个位置。因此，使用ViewStub可以提高界面初始化的性能，从而提高界面的加载速度。

      * ```xml
        <?xml version="1.0" encoding="utf-8"?>
        <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical">
          <ViewStub
              android:id="@+id/viewsub"
              android:layout_width="match_parent"
              android:layout_height="40dp"
              android:layout="@layout/titlebar"/>
           ...
        </LinearLayout>
        ```

      * ```java
        public class MainActivity extends AppCompatActivity {
            private ViewStub viewsub;
            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(R.layout.activity_main);
                viewsub= (ViewStub) findViewById(R.id.viewsub);
        //        viewsub.inflate();//1
                viewsub.setVisibility(View.VISIBLE);//2
            }
        }
        ```

    * ### **避免GPU过度绘制**

      * 在屏幕上某个像素在同一帧的时间内被绘制多次，从而浪费了GPU和CPU的资源

        * 在XML布局中，控件有重叠且都有设置背景。

        * View的OnDraw中同一区域绘制多次。

        * ![VKYeu4.png](https://s2.ax1x.com/2019/05/30/VKYeu4.png)

        * ![VKYVvF.png](https://s2.ax1x.com/2019/05/30/VKYVvF.png)

        * 各个颜色的定义为：

          - 原色： 没有过度绘制 – 每个像素在屏幕上绘制了一次。
          - 蓝色： 一次过度绘制 – 每个像素点在屏幕上绘制了两次。
          - 绿色： 两次过度绘制 – 每个像素点在屏幕上绘制了三次。
          - 粉色： 三次过度绘制 – 每个像素点在屏幕上绘制了四次。
          - 红色： 四次或四次以上过度绘制 – 每个像素点在屏幕上绘制了五次或者五次以上。

          最理想的是蓝色，一个像素只绘制一次，合格的页面绘制是白色、蓝色为主，绿色以上区域不能超过整个的三分之一，颜色越浅越好。

        * 避免过度绘制主要有以下几个方案：

          1. 移除不需要的background。
          2. 在自定义View的OnDraw方法中，用canvas.clipRect来指定绘制的区域，防止重叠的组件发生过度绘制。

#### 内存优化

* 内存泄漏

  * 可达性分析中，在Reference Chain上的Object一直不能回收导致

  * 产生原因

    * 又开发人员自己编码造成
    * 第三方框架造成
    * 由Android系统或者第三方ROM造成的泄漏

  * 场景

    1. 非静态内部类的静态实例（内部类容易出现内存泄漏，而静态内部类独立于外部类）

       ```java
       public class SecondActivity extends AppCompatActivity {
           private static Object inner;
           private Button button;
       
           @Override
           protected void onCreate(Bundle savedInstanceState) {
               super.onCreate(savedInstanceState);
               setContentView(R.layout.activity_main);
               button = (Button) findViewById(R.id.bt_next);
               button.setOnClickListener(new View.OnClickListener() {
                   @Override
                   public void onClick(View v) {
                       createInnerClass();
                       finish();
                   }
               });
           }
       
           void createInnerClass() {
               class InnerClass {
               }
               inner = new InnerClass();//1
           }
       }
       ```

       当点击Button时，会在注释1处创建了非静态内部类InnerClass的静态实例inner，该实例的生命周期会和应用程序一样长，并且会一直持有SecondActivity 的引用，导致SecondActivity无法被回收。

    2. ​	**匿名内部类的静态实例**

       ```java
       public class AsyncTaskActivity extends AppCompatActivity {
           private Button button;
           @Override
           protected void onCreate(Bundle savedInstanceState) {
               super.onCreate(savedInstanceState);
               setContentView(R.layout.activity_async_task);
               button = (Button) findViewById(R.id.bt_next);
               button.setOnClickListener(new View.OnClickListener() {
                   @Override
                   public void onClick(View v) {
                       startAsyncTask();
                       finish();
                   }
               });
           }
           void startAsyncTask() {
               new AsyncTask<Void, Void, Void>() {//1
                   @Override
                   protected Void doInBackground(Void... params) {
                       while (true) ;
                   }
               }.execute();
           }
       }
       ```

       在注释1处实例化了一个AsyncTask，当AsyncTask的异步任务在后台执行耗时任务期间，AsyncTaskActivity 被销毁了，被AsyncTask持有的AsyncTaskActivity实例不会被垃圾收集器回收，直到异步任务结束。

       解决办法就是自定义一个静态的AsyncTask，如下所示。

       ```java
       public class AsyncTaskActivity extends AppCompatActivity {
           private Button button;
           @Override
           protected void onCreate(Bundle savedInstanceState) {
               super.onCreate(savedInstanceState);
               setContentView(R.layout.activity_async_task);
               button = (Button) findViewById(R.id.bt_next);
               button.setOnClickListener(new View.OnClickListener() {
                   @Override
                   public void onClick(View v) {
                       startAsyncTask();
                       finish();
                   }
               });
           }
           void startAsyncTask() {
               new MyAsyncTask().execute();
           }
           private static class MyAsyncTask extends AsyncTask<Void, Void, Void> {
               @Override
               protected Void doInBackground(Void... params) {
                   while (true) ;
               }
           }
       }
       ```

    3. **Handler内存泄漏**

       Handler的Message被存储在MessageQueue中，有些Message并不能马上被处理，它们在MessageQueue中存在的时间会很长，这就会导致Handler无法被回收。如果Handler 是非静态的，则Handler也会导致引用它的Activity或者Service不能被回收。

       详情见 [https://medium.com/bumble-tech/android-handler-memory-leaks-7291c5be6101] 解释

    4. **未正确使用Context**

       对于不是必须使用Activity Context的情况（Dialog的Context就必须是Activity Context），我们可以考虑使用Application Context来代替Activity的Context，这样可以避免Activity泄露，比如如下的单例模式：

       ```java
       public class AppSettings { 
        private Context mAppContext;
        private static AppSettings mAppSettings = new AppSettings();
        public static AppSettings getInstance() {
         return mAppSettings;
        }
         
        public final void setup(Context context) {
         mAppContext = context;
        }
       }
       ```

       mAppSettings作为静态对象，其生命周期会长于Activity。当进行屏幕旋转时，默认情况下，系统会销毁当前Activity，因为当前Activity调用了setup方法，并传入了Activity Context，使得Activity被一个单例持有，导致垃圾收集器无法回收，进而产生了内存泄露。
       解决方法就是使用Application的Context：

       ```java
       public final void setup(Context context) {
        mAppContext = context.getApplicationContext(); 
       }
       ```

    5. #### **静态View**

       使用静态View可以避免每次启动Activity都去读取并渲染View，但是静态View会持有Activity的引用，导致Activity无法被回收，解决的办法就是在onDestory方法中将静态View置为null。

       ```java
       public class SecondActivity extends AppCompatActivity {
           //静态view
           private static Button button;
           @Override
           protected void onCreate(Bundle savedInstanceState) {
               super.onCreate(savedInstanceState);
               setContentView(R.layout.activity_main);
               button = (Button) findViewById(R.id.bt_next);
               button.setOnClickListener(new View.OnClickListener() {
                   @Override
                   public void onClick(View v) {
                       finish();
                   }
               });
           }
           
           //解决办法
           @Override
           protected void onDestroy(){
               button = null;
           }
        }   
       ```

    6. #### **WebView**

       不同的Android版本的WebView会有差异，加上不同厂商的定制ROM的WebView的差异，这就导致WebView存在着很大的兼容性问题。WebView都会存在内存泄漏的问题，在应用中只要使用一次WebView，内存就不会被释放掉。通常的解决办法就是为WebView单开一个进程，使用AIDL与应用的主进程进行通信。WebView进程可以根据业务需求，在合适的时机进行销毁。

    7. #### **资源对象未关闭**

       资源对象比如Cursor、File等，往往都用了缓冲，不使用的时候应该关闭它们。把他们的引用置为null，而不关闭它们，往往会造成内存泄漏。因此，在资源对象不使用时，一定要确保它已经关闭，通常在finally语句中关闭，防止出现异常时，资源未被释放的问题。

    8. #### **集合中对象没清理**

       通常把一些对象的引用加入到了集合中，当不需要该对象时，如果没有把它的引用从集合中清理掉，这样这个集合就会越来越大。如果这个集合是static的话，那情况就会更加严重。

    9. #### **Bitmap对象**

       临时创建的某个相对比较大的bitmap对象，在经过变换得到新的bitmap对象之后，应该尽快回收原始的bitmap，这样能够更快释放原始bitmap所占用的空间。
       避免静态变量持有比较大的bitmap对象或者其他大的数据对象，如果已经持有，要尽快置空该静态变量。

    10. #### **监听器未关闭**

        很多系统服务（比如TelephonyMannager、SensorManager）需要register和unregister监听器，我们需要确保在合适的时候及时unregister那些监听器。自己手动add的Listener，要记得在合适的时候及时remove这个Listener。

* #### **Memory Monitor**

  * 监视程序的性能和内存使用情况

  * ![VQz8SO.png](https://s2.ax1x.com/2019/05/31/VQz8SO.png)

    图中的标注的功能如下：

    * Initiate GC(标识1)：用来手动触发GC。
    * Dump Java heap(标识2)：保存内存快照。
    * Start/Stop Allocation Tracking(标识3)：打开Allocation Tracker工具（后面会介绍）。
    * Free(标识4)：当前应用未分配的内存大小。
    * Allocated(标识5)：当前应用分配的内存大小。

    图中y轴显示当前应用的分配的内存和未分配的内存大小；x轴表示经过的时间。

* ####  内存抖动

  * ![VQzU0A.jpg](https://s2.ax1x.com/2019/05/31/VQzU0A.jpg)
  * 内存抖动一般指在很短的时间内发生了多次内存分配和释放，严重的内存抖动还会导致应用程序卡顿。

* ### **Allocation Tracker**

  * Allocation Tracker用来跟踪内存分配，它允许你在执行某些操作的同时监视在何处分配对象
  * ![VQzGlD.png](https://s2.ax1x.com/2019/05/31/VQzGlD.png)

* ### **Heap Dump**

  * ![VQzNmd.png](https://s2.ax1x.com/2019/05/31/VQzNmd.png)
  * 行信息中比较重要的是free，它与总览视图中的free的含义不同，它代表内存碎片。当新创建一个对象时，如果碎片内存能容下该对象，则复用碎片内存，否则就会从free空间（总览视图中的free）重新划分内存给这个新对象。free是判断内存碎片化程度的一个重要的指标。
    此外，1-byte array这一行的信息也很重要，因为图片是以byte[]的形式存储在内存中的，如果1-byte array一行的数据过大，则需要检查图片的内存管理了。

* **内存分析工具MAT**

  * [刘望舒网站](http://liuwangshu.cn/application/performance/ram-5-mat.html)

  * 堆存储文件可以使用DDMS或者Memory Monitor来生成，输出的文件格式为hprof，而MAT就是分析它的

  * 引用树

    * ![VliYX8.png](https://s2.ax1x.com/2019/05/31/VliYX8.png)

  * 支配树

    * ![Vlia7Q.png](https://s2.ax1x.com/2019/05/31/Vlia7Q.png)
    * C直接支配D、E，因此C是D、E的父节点，这一点根据上面的阐述很容易得出结论。C直接支配H，这可能会有些疑问，能到达H的主要有两条路径，而这两条路径FD和GE都不是必须要经过的节点，只有C满足了这一点，因此C直接支配H，C就是H的父节点。通过支配树，我们就可以很容易的分析一个对象的Retained Set，比如E被回收，则会释放E、G的内存，而不会释放H的内存，因为F可能还引用着H，只有C被回收，H的内存才会被释放。

  * #### **OQL**

    * ```
      SELECT * FROM [ INSTANCEOF ]	<class_name> [ WHERE <filter-expression>]
      ```

    * ![Vlis10.png](https://s2.ax1x.com/2019/05/31/Vlis10.png)

* LeakCanary

  * Square公司基于MAT开源了[LeakCanary](https://github.com/square/leakcanary)

  * ### **使用LeakCanary**

    * ```xml
      dependencies {
        debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5.2'
        releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.2'
      }
      ```

    * ```java
      public class LeakApplication extends Application {
          @Override public void onCreate() {
          super.onCreate();
          //当前进程是给LeakCanary进行堆分析的
          if (LeakCanary.isInAnalyzerProcess(this)) {//1
            // This process is dedicated to LeakCanary for heap analysis.
            // You should not init your app in this process.
            return;
          }
          LeakCanary.install(this);
        }
      }
      ```

  * ### **LeakCanary应用举例**

    * ```java
      public class LeakApplication extends Application {
          private RefWatcher refWatcher;
          @Override
          public void onCreate() {
              super.onCreate();
              refWatcher= setupLeakCanary();
          }
          private RefWatcher setupLeakCanary() {
              if (LeakCanary.isInAnalyzerProcess(this)) {
                  return RefWatcher.DISABLED;
              }
              return LeakCanary.install(this);
          }
      
          public static RefWatcher getRefWatcher(Context context) {
              LeakApplication leakApplication = (LeakApplication) context.getApplicationContext();
              return leakApplication.refWatcher;
          }
      }
      ```

    * ```java
      public class MainActivity extends AppCompatActivity {
          @Override
          protected void onCreate(Bundle savedInstanceState) {
              super.onCreate(savedInstanceState);
              setContentView(R.layout.activity_main);
              LeakThread leakThread = new LeakThread();
              leakThread.start();
          }
          class LeakThread extends Thread {
              @Override
              public void run() {
                  try {
                      Thread.sleep(6 * 60 * 1000);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
          }
          @Override
          protected void onDestroy() {
              super.onDestroy();
              RefWatcher refWatcher = LeakApplication.getRefWatcher(this);//1
              refWatcher.watch(this);
          }
      }
      ```

    * ![VlFo5j.png](https://s2.ax1x.com/2019/05/31/VlFo5j.png)

    * ![VlFIaQ.png](https://s2.ax1x.com/2019/05/31/VlFIaQ.png)

    * 点击加号就可以查看具体类所在的包名称。整个详情就是一个引用链：MainActiviy的内部类LeakThread引用了LeakThread的`this$0`，`this$0`的含义就是内部类自动保留的一个指向所在外部类的引用，而这个外部类就是详情最后一行所给出的MainActiviy的实例，这将会导致MainActivity无法被GC，从而产生内存泄漏。

      除此之外，我们还可以将 heap dump（hprof文件）和info信息分享出去，如下图所示。
