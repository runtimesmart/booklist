
## Android源码通读笔记

### 1.android系统分Application、Framework、libraries、Linux内核四层

  * 其中Dalivk虚拟机在Libraries中，Linux内核主要是Camera驱动、光感Sensor驱动、重力加速器驱动、Audio、wifi、蓝牙、Display、Bindler等驱动
  * Libraries层主要是C/C++实现，包括init、Audio、AudioFlinger、Surface等系统
  
### 2.init进程
  * init进程是Linux系统也是Android系统的第一个进程，负责创建几个关键进程Zygote、SystemServer进程
  * 负责加载系统配置文件（init.rc文件）、创建属性服务（property service）属性服务将系统配置映射到共享内存空间，设置为只读属性。便于其他进程访问。
  
### 3.property_service  
  + 其中init进程会创建一个property_service作为独立的子进程,加载了default.prop文件，使用mmap映射到共享内存，并使用socket实现ipc通信
  + 目前最多支持加载247项属性--**待考证** 
  + init.rc文件具体可查看 [csdn]（https://blog.csdn.net/stoic163/article/details/78773141） bootanimation是在init.rc中定义的

### 4.zygote进程
  * init进程创建了zygote进程，zygote进程开始涉及java,它创建了system_server进程；zygote和system_server死亡会导致java崩溃。
  其中init进程检查发现zygote如果死亡，会将zygote及其所属的所有子进程全部杀死
  * zygote中有个AppRuntiem类，属于AndroidRuntime的子类，它创建了java Dalvik虚拟机。虚拟机中设置了heapsize=16MB,一般OEM商会改为32MB，stack为8MB
  
### 5.system进程
  * system_server进程是由zygote创建，注意的是它是由zygoteInit.java中throw exception创建，好处是不会占用之前的堆栈
  
### 6. 当创建了一个独立进程的Activity时，如何创建进程
  * 通过SystemServer的子进程activityManagerSerivce
  打开socket openSocketIfNeeded 写入请求到Zygote并通过流读取到返回的进程id.新创建的进程会在ZygoteConnection.java类中调用 handleChildProc->
  RuntimeInit.java中zygoteInit方法 重定向标准输入输出流，修改为System.setOut(AndroidPrintStream(Log.INFO,"System.Out"))..setErr等
  并且通过 ZygoteInitNative建立与Binder的绑定关系。
  
  Linux流分为文件流和socket流

### 7. WatchDog
*注意，android 7，8版本和之前实现完全不一样，旧版本主要是通过Handler发送emtpy消息，接收后来处理重置isCompleted和尝试获取锁来判断。新版本是通过
HandlerChecker 定时检查scheduleCheckLocked  通过回调monitor和synchronize（this）来重置isCompleted=true 来判断是否自杀。
* WatchDog是由SystemServer中启动，主要监听AMS WMS PMS。监听方法 通过在服务中调用addMonitor或者addThread传入对象，看门狗通过判断 服务中消息是否阻塞和是否死锁来决定是否通知System自杀,判断死锁的方法：首先在监听的服务中实现看门狗的Monitor接口，并添加回调addMonitor(this)，使用synchronized (Watchdog.this) {
                mCompleted = true;
                mCurrentMonitor = null;
            }
            尝试获取一次锁，如果死锁则无置为true
如果发现出问题会杀死systemServer,进而导致Zyote进程自杀，导致系统重启
    * 首先，sp是strong pointer。 wp是weak pointer，sp和wp是用于管理引用对象的内存回收的。引用的增减均为原子操作。
  
### 9.Binder机制
  * 这篇讲的较全面 [简书](https://www.jianshu.com/p/b4a8be5c6300 "简书")  
  * 分为两类，系统级Binder和应用层通信流程是不一样的。系统通信要经过serviceManager,应用层实现Stub,通过Proxy代理调用transact。
  * 1.系统级，用户在使用系统服务时，例如MediaPlayService时，调用对应的方法后MediaPlayService-->DefaultServiceManager(BPServiceManager).AddService-->mReomte(BpBinder).transact-->IPCThreadState.transact-->mIn(接收),mOut(发送-->发送之后-->waitForResponse
  
### 10.App启动和activity UI绘制流程简述


 * #### 1.首先从SystemServer中获取AMS代理对象，AMS通知Zygote进程fork一个新的App进程
 * #### 2.app进程创建ActivityThread，ActivityThread中执行handlerlauncherActivity,
 * #### 3.其中performLauncheActivity中使用Instrucmention使用反射创建一个Activity的类，其中会回调Activity的onCreate方法
 * #### 4.并且performLaunchActivity中会使用PolicyWindowManager创建一个PhoneWindow对象，其父类是Window，PhoneWindow主要
  作用是分发和处理上抛的触摸/按键事件，layout布局、通过ViewRoot获取Surface绘制view然后显示
 > ##### 4.1：Surface绘制会使用到skia的底层lib，涉及FrameBuffer,使用到 FrameBufferStack数据结构，属于生产消费模式，使用BackBuffer添加
   frontBuffer输出帧
 > ##### 4.2：surface有三种属性，分别是Normal模式，包括绘制帧(drawBuffer)和pushBuffer,UI绘制属于drawBuffer；blur模式和Dim模式，dim例如
   dialog弹框后背景变暗变模糊。
 >  ##### 4.3：SurfaceFlinger和AudioFlinger 分别使用到FrameBufferStack和RingBuffer；需要了解下。
 * #### 5.然后执行handleResumeActivity方法，这里面会获取window并setView()
           
  
