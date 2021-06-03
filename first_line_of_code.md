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

### 菜单Menu

* 右上角三个点，点击后可以弹出一个下拉列表
* 在res目录下新建menu文件夹, 再新建menu.xml文件
* `<menu xmlns:android="http://schemas.android.com/apk/res/android">`
* `<item android:id="@+id/add_item android:title="Add"/>`
* `<item android:id="@+id/remove_item android:title="Remove"/>`
* `</menu>`
* 重写onCreateOptionsMenu
* `getMenuInflater().inflate(R.menu.menu,menu); return true;` 获取这个menu并显示
* 重写onOptionsItemSelected(MenuItem item)
* 通过item.getItemId()判断是哪一个item被选择了

### 销毁活动

* finish()

### Intent

* 显式

  * 作用：调用自己程序内的活动
  * Intent(当前活动对象，目标活动Class)
    *  `Intent it = new Intent(FirstActivity.this, SecondActivity.class)`
    * `startActivity(it);`
  * 总结：纯java代码实现跳转

* 隐式

  * 作用: 调用其他程序的活动

    * 实现打开一个浏览器

      * ```java 
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setData(Uri.parse("http://www.baidu.com"));
        startActivity(intent);
        ```

    * 实现打电话

      * ```java 
        Intent intent = new Intent(Intent.ACTION_DIAL);
        intent.setData(Uri.parse("tel:10086"));
        startActivity(intent);
        ```

  * ```xml 
    <activity android:name=".SecondActivity">
    	<intent-filter>
        	<action android:name="com.example.activitytest.ACTION_START"/>
            <category android:name="android.intent.category.DEFAULT"/>
        </intent-filter>
    </activity>
    ```

  * ```java 
    Intent intent = new Intent("com.example.activitytest.ACTION_START");
    ```

  * 通过 action的 android:name属性值找到匹配的活动页，进行跳转

  * 总结：XML和Java代码结合

* 传递数据给下一个活动

  * 发送发

    * ```java 
      intent.putExtra("extra_data",data);
      ```

  * 接收方

    * ```java
      Intent intent = getIntent();
      String data = intent.getStringExtra("extra_data");
      ```

* 传递数据给上一个活动

  * 主活动

    * ```java 
      startActivityForResult(intent,1); //第二个参数是唯一的代码
      
      //重写onActivityResult来接收来自次活动的数据
      protected void onActivityResult(int requestCode, int resultCode, Intent data){
          switch(requestCode){
              case 1:
                  if(resultCode == RESULT_OK){
                      String returnedData = data.getStringExtra("data_return");
                  }
                  break;
              default;
          }
      }
      ```

  * 次活动

    * ```java 
      Intent intent = new Intent();
      intent.putExtra("data_return","Hello FirstActivity");
      setResult(RESULT_OK,intent);
      finish();
      ```

### 活动的生命周期

* 符合栈的数据结构
* Back或者finish()用来去除栈顶活动
* 四种状态
  1. 运行状态
     1. 活动位于栈顶
  2. 暂停状态
     1. 活动不在栈顶，但是仍然可见
  3. 停止状态
     1. 活动不在栈顶，也不可见
  4. 销毁状态
     1. 活动从栈中移除

### 活动的生存期

* onCreate 活动第一次创建

* onStart 活动由不可见变成可见

* onResume 活动准备好和用户交互时调用

* onPause 启动或者恢复另一个活动

* onStop 活动变成不可见

* onDestroy 活动销毁之前

* onRestart 停止变成运行状态之前

* ![](imgs/activity_lifetime.png)

* 活动回收后数据保存问题：

  * onSaveInstanceState(Bundle outState) 方法中可以保存临时数据

  * ```java 
    @Override
    protected void onCreate(Bundle savedInstanceState){
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(savedInstanceState != null){
            String tempData = savedInstanceState.getString("data_key");
        }
    }
    ```

  * onCreate方法中Bundle参数可以判断是否存入临时数据

