<h1>第5章  深入理解 PowerManagerService</h1>
<h2>本章主要内容：</h2>
<p>·  深入分析PowerManagerService</p>
<p>·  深入分析BatteryService和BatteryStatsService</p>
<h2>本章所涉及的源代码文件名及位置：</h2>
<p>·  PowerManagerService.java</p>
<p>frameworks/base/services/java/com/android/server/PowerManagerService.java</p>
<p>·  com_android_server_PowerManagerService.cpp</p>
<p>frameworks/base/services/jni/com_android_server_PowerManagerService.cpp</p>
<p>·  PowerManager.java</p>
<p>frameworks/base/core/java/android/os/PowerManager.java</p>
<p>·  WorkSoure.java</p>
<p>frameworks/base/core/java/android/os/WorkSoure.java</p>
<p>·  Power.java</p>
<p>frameworks/base/core/java/android/os/Power.java</p>
<p>·  android_os_Power.cpp</p>
<p>frameworks/base/core/jni/android_os_Power.cpp</p>
<p>·  com_android_server_InputManager.cpp</p>
<p>frameworks/base/services/jni/com_android_server_InputManager.cpp</p>
<p>·  LightService.java</p>
<p>frameworks/base/services/java/com/android/server/LightService.java</p>
<p>·  com_android_server_LightService.cpp</p>
<p>frameworks/base/services/jni/com_android_server_LightService.cpp</p>
<p>·  BatteryService.java</p>
<p>frameworks/base/services/java/com/android/server/BatteryService.java</p>
<p>·  com_android_server_BatteryService.cpp</p>
<p>frameworks/base/services/jni/com_android_server_BatteryService.cpp</p>
<p>·  ActivityManagerService.java</p>
<p>frameworks/base/services/java/com/android/server/am/ActivityManagerService.java</p>
<p>·  BatteryStatsService.java</p>
<p>frameworks/base/services/java/com/android/server/am/BatteryStatsService.java</p>
<p>·  BatteryStatsImpl.java</p>
<p>frameworks/base/core/java/com/android/internal/os/BatteryStatsImpl.java</p>
<p>·  LocalPowerManager.java</p>
<p>frameworks/base/core/java/android/os/LocalPowerManager.java</p>
<h2>5.1  概述</h2>
<p>PowerManagerService负责Andorid系统中电源管理方面的工作。作为系统核心服务之一，PowerManagerService与其他服务及HAL层等都有交互关系，所以PowerManagerService相对PackageManager来说，其社会关系更复杂，分析难度也会更大一些。</p>
<p>先来看直接与PowerManagerService有关的类家族成员，如图5-1所示</p>

![image](images/chapter5/image001.png)

<p>图5-1  PowerManagerService及相关类家族</p>
<p>由图5-1可知：</p>
<p>·  PowerManagerService从IPowerManager.Stub类派生，并实现了Watchdog.Monitor及LocalPowerManager接口。PowerManagerService内部定义了较多的成员变量，在后续分析中，我们会对其中比较重要的成员逐一进行介绍。</p>
<p>·  根据第4章介绍的知识，IPowerManager.Stub及内部类Proxy均由aidl工具处理PowerManager.aidl后得到。</p>
<p>·  客户端使用PowerManager类，其内部通过代表BinderProxy端的mService成员变量与PowerManagerService进行跨Binder通信。</p>
<p>现在开始PowerManagerService（以后简写为PMS）的分析之旅，先从它的调用流程入手。</p>
<div>
    <p><strong>提示</strong>PMS和BatteryService、BatteryStatsService均有交互关系，这些内容放在后面分析。</p>
</div>
<h2>5.2  初识PowerManagerService</h2>
<p>PMS由SystemServer在ServerThread线程中创建。这里从中提取了4个关键调用点，如下所示： </p>
<p>[--&gt;SystemServer.java]</p>
<div>
    <p>    ......//ServerThread的run函数</p>
    <p>    power =new PowerManagerService();//①创建PMS对象</p>
    <p>   ServiceManager.addService(Context.POWER_SERVICE, power);//注册到SM中</p>
    <p>   ......</p>
    <p>   //②调用PMS的init函数</p>
    <p>   power.init(context,lights, ActivityManagerService.self(), battery);</p>
    <p>   ......//其他服务</p>
    <p>   power.systemReady();//③调用PMS的systemReady</p>
    <p>   ......//系统启动完毕，会收到<a>ACTION_BOOT_COMPLETED</a>广播</p>
    <p>   //④PMS处理ACTION_BOOT_COMPLETED广播</p>
</div>
<p>先从第一个关键点即PMS的构造函数开始分析。</p>
<h3>5.2.1  PMS构造函数分析</h3>
<p>PMS构造函数的代码如下：</p>
<p>[--&gt;PowerManagerService.java::构造函数]</p>
<div>
    <p>PowerManagerService() {</p>
    <p>    longtoken = Binder.clearCallingIdentity();</p>
    <p>    MY_UID =Process.myUid();//取本进程（即SystemServer）的uid及pid</p>
    <p>    MY_PID =Process.myPid();</p>
    <p>   Binder.restoreCallingIdentity(token);</p>
    <p>    //设置超时时间为1周。Power类封装了同Linux内核交互的接口。本章最后再来分析它</p>
    <p>   Power.setLastUserActivityTimeout(7*24*3600*1000);</p>
    <p>    //初始化两个状态变量，它们非常有意义。其具体作用后续再分析</p>
    <p>    mUserState= mPowerState = 0;</p>
    <p>    //将自己添加到看门狗的监控管理队列中</p>
    <p>   Watchdog.getInstance().addMonitor(this);</p>
    <p> }</p>
</div>
<p>PMS的构造函数比较简单。值得注意的是mUserState和mPowerState两个成员，至于它们的具体作用，后续分析时自会知晓。</p>
<p>下面分析第二个关键点。</p>
<h3>5.2.2  init分析</h3>
<p>第二个关键点是init函数，该函数将初始化PMS内部的一些重要成员变量，由于此函数代码较长，此处将分段讨论。</p>
<p>从流程角度看，init大体可分为三段。</p>
<h4>1.  init分析之一</h4>
<p>[--&gt;PowerManagerService.java::init函数]</p>
<div>
    <p>void init(Context context, LightsService lights,IActivityManager activity,</p>
    <p>                            BatteryService battery) {</p>
    <p>   //①保存几个成员变量</p>
    <p>  mLightsService = lights;//保存LightService</p>
    <p>   mContext= context;</p>
    <p>  mActivityService = activity;//保存ActivityManagerService</p>
    <p>   //保存BatteryStatsService</p>
    <p>  mBatteryStats = BatteryStatsService.getService();//</p>
    <p>  mBatteryService = battery;//保存BatteryService</p>
    <p>   //从LightService中获取代表不同硬件Light的Light对象</p>
    <p>   mLcdLight= lights.getLight(LightsService.LIGHT_ID_BACKLIGHT);</p>
    <p>  mButtonLight = lights.getLight(LightsService.LIGHT_ID_BUTTONS);</p>
    <p>  mKeyboardLight = lights.getLight(LightsService.LIGHT_ID_KEYBOARD);</p>
    <p>  mAttentionLight = lights.getLight(LightsService.LIGHT_ID_ATTENTION);</p>
    <p>   //②调用nativeInit函数</p>
    <p>  nativeInit();</p>
    <p>  synchronized (mLocks) {</p>
    <p>     updateNativePowerStateLocked();//③更新Native层的电源状态</p>
    <p>  }</p>
</div>
<p>第一阶段工作可分为三步：</p>
<p>·  对一些成员变量进行赋值。</p>
<p>·  调用nativeInit函数初始化Native层相关资源。</p>
<p>·  调用updateNativePowerStateLocked更新Native层的电源状态。这个函数的调用次数较为频繁，以后续分析时讨论。</p>
<p>先来看第一阶段出现的各类成员变量，如表5-1所示。</p>
<p>表5-1  成员变量说明</p>
<table>
    <tbody>
        <tr>
            <td>
                <p>成员变量名</p>
            </td>
            <td>
                <p>数据类型</p>
            </td>
            <td>
                <p>作用</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mLightsService</p>
            </td>
            <td>
                <p>LightsService</p>
            </td>
            <td>
                <p>和LightsService交互用</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mActivityService</p>
            </td>
            <td>
                <p>IActivityManager</p>
            </td>
            <td>
                <p>和ActivityManagerService交互</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mBatteryStats</p>
            </td>
            <td>
                <p>IBatteryStats</p>
            </td>
            <td>
                <p>和BatteryStatsService交互，用于系统耗电量统计方面的工作</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mBatteryService</p>
            </td>
            <td>
                <p>BatteryService</p>
            </td>
            <td>
                <p>用于获取电源状态，例如是否为低电状态、查询电池电量等</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mLcdLight、mButtonLight、</p>
                <p>mKeyboardLight、mAttentionLight</p>
            </td>
            <td>
                <p>LightsService.Light</p>
            </td>
            <td>
                <p>由PMS控制，在不同状态下点亮或熄灭它们</p>
            </td>
        </tr>
    </tbody>
</table>
<p>下面来看nativeInit函数，其JNI层实现代码如下：</p>
<p>[--&gt;com_android_server_PowerManagerService.cpp]</p>
<div>
    <p>static void android_server_PowerManagerService_nativeInit(JNIEnv*env,</p>
    <p>                             jobject obj) {</p>
    <p>    //非常简单，就是创建一个全局引用对象gPowerManagerServiceObj</p>
    <p>   gPowerManagerServiceObj = env-&gt;NewGlobalRef(obj);</p>
    <p>}</p>
</div>
<p>init第一阶段工作比较简单，下面进入第二阶段的分析。</p>
<h4>2.  init分析之二</h4>
<p>init第二阶段工作将创建两个HandlerThread对象，即创建两个带消息循环的工作线程。PMS本身由ServerThread线程创建，并且将自己的工作委托给这两个线程，它们分别是：</p>
<p>·  mScreenOffThread：按Power键关闭屏幕时，屏幕不是突然变黑的，而是一个渐暗的过程。mScreenOffThread线程就用于控制关屏过程中的亮度调节。</p>
<p>·  mHandlerThread：该线程是PMS的主要工作线程。</p>
<p>先来看这两个线程的创建。</p>
<h5>（1） mScreenOffThread和mHandlerThread分析</h5>
<p>[--&gt;PowerManagerService.java::init函数]</p>
<div>
    <p>......</p>
    <p> mScreenOffThread= new HandlerThread("PowerManagerService.mScreenOffThread") {</p>
    <p>   protected void onLooperPrepared() {</p>
    <p>  mScreenOffHandler = new Handler();//向这个handler发送的消息，将由此线程处理</p>
    <p>   synchronized (mScreenOffThread) {</p>
    <p>      mInitComplete = true;</p>
    <p>      mScreenOffThread.notifyAll();</p>
    <p>      }</p>
    <p>    }</p>
    <p>   };</p>
    <p> mScreenOffThread.start();//创建对应的工作线程</p>
    <p> synchronized (mScreenOffThread) {</p>
    <p>    while(!mInitComplete) {</p>
    <p>       try {//等待mScreenOffThread线程创建完成</p>
    <p>             mScreenOffThread.wait();</p>
    <p>        } ......</p>
    <p>       }</p>
    <p>    }</p>
</div>
<p>注意，在Android代码中经常出现“线程A创建线程B，然后线程A等待线程B创建完成”的情况，读者了解它们的作用即可。接着看以下代码。</p>
<p>[--&gt;PowerManagerService.java::init函数]</p>
<div>
    <p>   mInitComplete= false;</p>
    <p>   //创建 mHandlerThread</p>
    <p>  mHandlerThread = new HandlerThread("PowerManagerService") {</p>
    <p>   protectedvoid onLooperPrepared() {</p>
    <p>      super.onLooperPrepared();</p>
    <p>      initInThread();//①初始化另外一些成员变量</p>
    <p>     }</p>
    <p>   };</p>
    <p> mHandlerThread.start();</p>
    <p>        ......//等待mHandlerThread创建完成</p>
</div>
<p>由于mHandlerThread承担了PMS的主要工作任务，因此需要先做一些初始化工作，相关的代码在initInThread中，拟放在单独一节中进行讨论。</p>
<h5>（2） initInThread分析</h5>
<p>initInThread本身比较简单，涉及三个方面的工作，总结如下：</p>
<p>·  PMS需要了解外面的世界，所以它会注册一些广播接收对象，接收诸如启动完毕、电池状态变化等广播。</p>
<p>·  PMS所从事的电源管理工作需要遵守一定的规则，而这些规则在代码中就是一些配置参数，这些配置参数的值可以是固定写死的（编译完后就无法改动），也可以是经由Settings数据库动态设定的。</p>
<p>·  PMS需要对外发出一些通知，例如屏幕关闭/屏幕开启。</p>
<p>了解initInThread的概貌后，再来看如下代码。</p>
<p>[--&gt;PowerManagerService.java::initInThread]</p>
<div>
    <p>void initInThread() {</p>
    <p>   mHandler= new Handler();</p>
    <p>   //PMS内部也需要使用WakeLock，此处定义了几种不同的UnsynchronizedWakeLock。它们的</p>
    <p>   //作用见后文分析</p>
    <p>   mBroadcastWakeLock = newUnsynchronizedWakeLock(</p>
    <p>              PowerManager.PARTIAL_WAKE_LOCK, "sleep_broadcast", true);</p>
    <p>   //创建广播通知的Intent，用于通知SCREEN_ON和SCREEN_OFF消息</p>
    <p>  mScreenOnIntent = new Intent(Intent.ACTION_SCREEN_ON);</p>
    <p>  mScreenOnIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);</p>
    <p>  mScreenOffIntent = new Intent(Intent.ACTION_SCREEN_OFF);</p>
    <p>  mScreenOffIntent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);</p>
    <p> </p>
    <p>   //取配置参数，这些参数是编译时确定的，运行过程中无法修改</p>
    <p>   Resourcesresources = mContext.getResources();</p>
    <p>  mAnimateScreenLights = resources.getBoolean(</p>
    <p>               com.android.internal.R.bool.config_animateScreenLights);</p>
    <p>        ......//见下文的配置参数汇总</p>
    <p>        //通过数据库设置的配置参数</p>
    <p>   ContentResolver resolver =mContext.getContentResolver();</p>
    <p>   Cursor settingsCursor =resolver.query(Settings.System.CONTENT_URI, null,</p>
    <p>               ......//设置查询条件和查询项的名字，见后文的配置参数汇总</p>
    <p>               null);</p>
    <p>   //ContentQueryMap是一个常用类，简化了数据库查询工作。读者可参考SDK中该类的说明文档</p>
    <p>   mSettings= new ContentQueryMap(settingsCursor, Settings.System.NAME,</p>
    <p>                                   true, mHandler);</p>
    <p>   //监视上边创建的ContentQueryMap中内容的变化</p>
    <p>  SettingsObserver settingsObserver = new SettingsObserver();</p>
    <p>   mSettings.addObserver(settingsObserver);</p>
    <p>   settingsObserver.update(mSettings, null);</p>
    <p>   //注册接收通知的BroadcastReceiver</p>
    <p>  IntentFilter filter = new IntentFilter();</p>
    <p>  filter.addAction(Intent.ACTION_BATTERY_CHANGED);</p>
    <p>  mContext.registerReceiver(new BatteryReceiver(), filter);</p>
    <p>   filter =new IntentFilter();</p>
    <p>  filter.addAction(Intent.ACTION_BOOT_COMPLETED);</p>
    <p>  mContext.registerReceiver(new BootCompletedReceiver(), filter);</p>
    <p>   filter =new IntentFilter();</p>
    <p>  filter.addAction(Intent.ACTION_DOCK_EVENT);</p>
    <p>  mContext.registerReceiver(new DockReceiver(), filter);</p>
    <p>    //监视Settings数据中secure表的变化</p>
    <p>  mContext.getContentResolver().registerContentObserver(</p>
    <p>            Settings.Secure.CONTENT_URI, true,</p>
    <p>           new ContentObserver(new Handler()) {</p>
    <p>               public void onChange(boolean selfChange) {</p>
    <p>                   updateSettingsValues();</p>
    <p>               }</p>
    <p>           });</p>
    <p>   updateSettingsValues();</p>
    <p>    ......//通知其他线程</p>
    <p> }</p>
</div>
<p>在上述代码中，很大一部分用于获取配置参数。同时，对于数据库中的配置值，还需要建立监测机制，细节部分请读者自己阅读相关代码，这里总结一下常用的配置参数，如表5-2所示。</p>
<p>表5-2  PMS使用的配置参数</p>
<table>
    <tbody>
        <tr>
            <td>
                <p>参数名:类型</p>
            </td>
            <td>
                <p>来源</p>
            </td>
            <td>
                <p>备注</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mAnimateScreenLights:bool</p>
            </td>
            <td>
                <p>config.xml<a>[①]</a></p>
            </td>
            <td>
                <p>关屏时屏幕光是否渐暗，默认为true</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mUnplugTurnsOnScreen:bool</p>
            </td>
            <td>
                <p>config.xml</p>
            </td>
            <td>
                <p>拔掉USB线，是否点亮屏幕</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mScreenBrightnessDim:int</p>
            </td>
            <td>
                <p>config.xml</p>
            </td>
            <td>
                <p>PMS可设置的屏幕亮度的最小值，默认20（单位lx）</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mUseSoftwareAutoBrightness:bool</p>
            </td>
            <td>
                <p>config.xml</p>
            </td>
            <td>
                <p>是否启用Setting中的亮度自动调节，如果硬件不支持该功能，则可由软件控制。默认为false</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mAutoBrightnessLevels:int[]</p>
                <p>mLcdBacklightValues:int[]</p>
                <p>......</p>
            </td>
            <td>
                <p>config.xml，具体值由硬件厂商定义</p>
            </td>
            <td>
                <p>当使用软件自动亮度调节时，需配置不同亮度时对应的参数</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>STAY_ON_WHILE_PLUGGED_IN:int</p>
            </td>
            <td>
                <p>Settings.db</p>
            </td>
            <td>
                <p>插入USB时是否保持唤醒状态</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>SCREEN_OFF_TIMEOUT:int</p>
            </td>
            <td>
                <p>Settings.db</p>
            </td>
            <td>
                <p>屏幕超时时间</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>DIM_SCREEN:int</p>
            </td>
            <td>
                <p>Settings.db</p>
            </td>
            <td>
                <p>是否变暗（dim）屏幕</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>SCREEN_BRIGHTNESS_MODE:int</p>
            </td>
            <td>
                <p>Settings.db</p>
            </td>
            <td>
                <p>屏幕亮度模式（自动还是手动调节）</p>
            </td>
        </tr>
    </tbody>
</table>
<p>除了获取配置参数外，initInThread还创建了好几个UnsynchronizedWakeLock对象，它的作用是：在Android系统中，为了抢占电力资源，客户端要使用WakeLock对象。PMS自己也不例外，所以为了保证在工作中不至于突然掉电（当其他客户端都不使用WakeLock的时候，这种情况理论上是有可能发生的），PMS需要定义供自己使用的WakeLock。由于线程同步方面的原因，PMS封装了一个UnsynchronizedWakeLock结构，它的调用已经处于锁保护下，所以在内部无需再做同步处理。UnsynchronizedWakeLock比较简单，因此不再赘述。</p>
<p>下面来看init第三阶段的工作。</p>
<h4>3.  init分析之三</h4>
<p>[--&gt;PowerManagerService.java::init函数]</p>
<div>
    <p>  nativeInit();//不知道此处为何还要调用一次nativeInit，笔者怀疑此处为bug</p>
    <p>  synchronized (mLocks) {</p>
    <p>     updateNativePowerStateLocked();//更新native层power状态，以后分析</p>
    <p>     forceUserActivityLocked();//强制触发一次用户事件</p>
    <p>    mInitialized = true;</p>
    <p> }//init函数完毕</p>
</div>
<p>forceUserActivityLocked表示强制触发一次用户事件。这个解释是否会让读者丈二和尚摸不着头？先来看它的代码：</p>
<p>[--&gt;PowerManagerService.java:: forceUserActivityLocked]</p>
<div>
    <p>private void forceUserActivityLocked() {</p>
    <p>   if(isScreenTurningOffLocked()) {</p>
    <p>    mScreenBrightness.animating = false;</p>
    <p>  }</p>
    <p>   boolean savedActivityAllowed =mUserActivityAllowed;</p>
    <p>  mUserActivityAllowed = true;</p>
    <p>  //下面这个函数以后会分析， SDK中有对应的API</p>
    <p>  userActivity(SystemClock.uptimeMillis(), false);</p>
    <p>   mUserActivityAllowed= savedActivityAllowed;</p>
    <p> }</p>
</div>
<p>forceUserActivityLocked内部就是为调用userActivity扫清一切障碍。对于SDK中PowerManager.userActivity的说明文档“User activity happened.Turnsthe device from whatever state it's in to full on, and resets the auto-offtimer.”简单翻译过来是：调用此函数后，手机将被唤醒。屏幕超时时间将重新计算。</p>
<p>userActivity是PMS中很重要的一个函数，本章后面将对其进行详细分析。</p>
<h4>4.  init函数总结</h4>
<p>PMS的init函数比较简单，但是其众多的成员变量让人感到有点头晕。读者自行阅读代码时，不妨参考表5-1和表5-2。</p>
<h3>5.2.3  systemReady分析</h3>
<p>下面来分析PMS第三阶段的工作。此时系统中大部分服务都已创建好，即将进入就绪阶段。就绪阶段的工作在systemReady中完成，代码如下：</p>
<p>[--&gt;PowerManagerService.java::systemReady]</p>
<div>
    <p>void systemReady() {</p>
    <p>  /*</p>
    <p>  创建一个SensorManager，用于和系统中的传感器系统交互，由于该部分涉及较多的native层</p>
    <p>  代码，因此将相关内容放到本书后续章节进行讨论</p>
    <p>  */</p>
    <p> mSensorManager = new SensorManager(mHandlerThread.getLooper());</p>
    <p> mProximitySensor = mSensorManager.getDefaultSensor(Sensor.TYPE_PROXIMITY);</p>
    <p>  if(mUseSoftwareAutoBrightness) {</p>
    <p>      mLightSensor =mSensorManager.getDefaultSensor(Sensor.TYPE_LIGHT);</p>
    <p>  }</p>
    <p>  if(mUseSoftwareAutoBrightness) {</p>
    <p>     setPowerState(SCREEN_BRIGHT);</p>
    <p>    } else {//不考虑软件自动亮度调节，所以执行下面这个分支</p>
    <p>   setPowerState(ALL_BRIGHT);//设置手机电源状态为ALL_BRIGHT，即屏幕、按键灯都打开</p>
    <p> }</p>
    <p> synchronized (mLocks) {</p>
    <p> mDoneBooting = true;</p>
    <p>  //根据情况启用LightSensor</p>
    <p> enableLightSensorLocked(mUseSoftwareAutoBrightness&amp;&amp;mAutoBrightessEnabled);</p>
    <p>  longidentity = Binder.clearCallingIdentity();</p>
    <p>  try {//通知BatteryStatsService，它将统计相关的电量使用情况，后续再分析它</p>
    <p>    mBatteryStats.noteScreenBrightness(getPreferredBrightness());</p>
    <p>    mBatteryStats.noteScreenOn();</p>
    <p>  }......</p>
    <p>}</p>
</div>
<p>systemReady主要工作为：</p>
<p>·  PMS创建SensorManager，通过它可与对应的传感器交互。关于Android传感器系统，将放到本书后续章节讨论。PMS仅仅启用或禁止特定的传感器，而来自传感器的数据将通过回调的方式通知PMS，PMS根据接收到的传感器事件做相应处理。</p>
<p>·  通过setPowerState函数设置电源状态为ALL_BRIGHT（不考虑UseSoftwareAutoBrightness的情况）。此时屏幕及键盘灯都会点亮。关于setPowrState函数，后文再做详细分析。</p>
<p>·  调用BatteryStatsService提供的函数，以通知屏幕打开事件，在BatteryStatsService内部将处理该事件。稍后，本章将详细讨论BatteryStatsService的功能。</p>
<p>当系统中的服务都在systemReady中进行处理后，系统会广播一次ACTION_BOOT_COMPLETED消息，而PMS也将处理该广播，下面来分析。</p>
<h3>5.2.4  BootComplete处理</h3>
<p>[--&gt;PowerManagerService.java::BootCompletedReceiver]</p>
<div>
    <p>private final class BootCompletedReceiver extendsBroadcastReceiver {</p>
    <p>  publicvoid onReceive(Context context, Intent intent) {</p>
    <p>  bootCompleted();//调用PMS的bootCompleted函数</p>
    <p>  }</p>
    <p>} </p>
</div>
<p>[--&gt;PowerManagerService.java::bootCompleted函数]</p>
<div>
    <p>void bootCompleted() {</p>
    <p>  </p>
    <p> synchronized (mLocks) {</p>
    <p>  mBootCompleted = true;</p>
    <p>   //再次碰见userActivity，根据前面的描述，此时将重新计算屏幕超时时间</p>
    <p>  userActivity(SystemClock.uptimeMillis(), false, BUTTON_EVENT, true);</p>
    <p>  updateWakeLockLocked();//此处先分析这个函数</p>
    <p>  mLocks.notifyAll();</p>
    <p>   }</p>
    <p> }</p>
</div>
<p>在以上代码中，再一次遇见了userActivity，暂且对其置之不理。先分析updateWakeLockLocked函数，其代码如下：</p>
<div>
    <p>private void updateWakeLockLocked() {</p>
    <p>  /*</p>
    <p>    mStayOnConditions用于控制当插上USB时，手机是否保持唤醒状态。</p>
    <p>    mBatteryService的isPowered用于判断当前是否处于USB充电状态。</p>
    <p>    如果满足下面的if条件满，则PMS需要使用wakeLock来确保系统不会掉电</p>
    <p>  */</p>
    <p>  if(mStayOnConditions != 0 &amp;&amp;mBatteryService.isPowered(mStayOnConditions)) {</p>
    <p>     mStayOnWhilePluggedInScreenDimLock.acquire();</p>
    <p>     mStayOnWhilePluggedInPartialLock.acquire();</p>
    <p>  } else {</p>
    <p>      //如果不满足if条件，则释放对应的wakeLock，这样系统就可以进入休眠状态</p>
    <p>     mStayOnWhilePluggedInScreenDimLock.release();</p>
    <p>     mStayOnWhilePluggedInPartialLock.release();</p>
    <p>  }</p>
    <p>}</p>
</div>
<p>mStayOnWhilePluggedInScreenDimLock和mStayOnWhilePluggedInPartialLock都为UnsynchronizedWakeLock类型，它们封装了WakeLock，可帮助PMS在使用它们时免遭线程同步之苦。</p>
<h3>5.2.5  初识PowerManagerService总结</h3>
<p>这一节向读者展示了PMS的大体面貌，包括：</p>
<p>·  主要的成员变量及它们的作用和来历。如有需要，可查阅表5-1和5-2。</p>
<p>·  见识了PMS中几个主要的函数，其中有一些将留到后文进行深入分析，现在只需要了解其大概作用即可。</p>
<h2>5.3  PMS WakeLock分析</h2>
<p>WakeLock是Android提供给应用程序获取电力资源的唯一方法。只要还有地方在使用WakeLock，系统就不会进入休眠状态。</p>
<p>WakeLock的一般使用方法如下：</p>
<div>
    <p>PowerManager pm = (PowerManager)getSystemService(Context.POWER_SERVICE);</p>
    <p> //①创建一个WakeLock，注意它的参数</p>
    <p> PowerManager.WakeLock wl =pm.newWakeLock(PowerManager.SCREEN_DIM_WAKE_LOCK,</p>
    <p>                                              "MyTag");</p>
    <p> wl.acquire();//②获取该锁</p>
    <p>   ......//工作</p>
    <p> wl.release();//③释放该锁</p>
</div>
<p>以上代码中共列出三个关键点，本章将分析前两个（在此基础上，读者可自行分析release函数）。</p>
<p>这3个函数都由PMS的Binder客户端的PowerManager使用，所以将本次分析划分为客户端和服务端两大部分。</p>
<p> </p>
<h3>5.3.1  WakeLock客户端分析</h3>
<h4>1.  newWakeLock分析</h4>
<p>通过PowerManager（以后简称PM）的newWakeLock将创建一个WakeLock，代码如下：</p>
<div>
    <p>public WakeLock newWakeLock(int flags, String tag)</p>
    <p>{</p>
    <p>  //tag不能为null，否则抛异常</p>
    <p>  return new WakeLock(flags, tag);//WakeLock为PM的内部类，第一个参数flags很关键</p>
    <p> }</p>
</div>
<p>WakeLock的第一个参数flags很关键，它用于控制CPU/Screen/Keyboard的休眠状态。flags的可选值如表5-3所示。</p>
<p>表5-3  WakeLock 的flags参数说明</p>
<table>
    <tbody>
        <tr>
            <td>
                <p>flags值</p>
            </td>
            <td>
                <p>CPU</p>
            </td>
            <td>
                <p>Screen</p>
            </td>
            <td>
                <p>Keyboard</p>
            </td>
            <td>
                <p>备注</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>PARTIAL_WAKE_LOCK</p>
            </td>
            <td>
                <p>On</p>
            </td>
            <td>
                <p>Off</p>
            </td>
            <td>
                <p>Off</p>
            </td>
            <td>
                <p>不受电源键影响</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>SCREEN_DIM_WAKE_LOCK</p>
            </td>
            <td>
                <p>On</p>
            </td>
            <td>
                <p>Dim</p>
            </td>
            <td>
                <p>Off</p>
            </td>
            <td>
                <p>按下电源键后，系统还是会进入休眠状态</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>SCREEN_BRIGHT_WAKE_LOCK</p>
            </td>
            <td>
                <p>On</p>
            </td>
            <td>
                <p>Bright</p>
            </td>
            <td>
                <p>Off</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>FULL_WAKE_LOCK</p>
            </td>
            <td>
                <p>On</p>
            </td>
            <td>
                <p>Bright</p>
            </td>
            <td>
                <p>On</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>ACQUIRE_CAUSES_WAKEUP</p>
            </td>
            <td>
                <p>说明：在正常情况下，获取WakeLock并不会唤醒机器（例如acquire之前机器处于关屏状态，则无法唤醒）。加上该标志后，acquire WakeLock同时也能唤醒机器（即点亮屏幕等）。该标志常用于提示框、来电提醒等应用场景</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>ON_AFTER_RELEASE</p>
            </td>
            <td>
                <p>说明：和用户体验有关，当WakeLock释放后，如没有该标志，系统会立即黑屏。有了该标志，系统会延时一段时间再黑屏</p>
            </td>
        </tr>
    </tbody>
</table>
<p>由表5-3可知：</p>
<p>·  WakeLock只控制CPU、屏幕和键盘三大部分。</p>
<p>·  表中最后两项是附加标志，和前面的其他WAKE_LOCK标志组合使用。注意， PARTIAL_WAKE_LOCK比较特殊，附加标志不能影响它。</p>
<p>·  PARTIAL_WAKE_LOCK不受电源键控制，即按电源键不能使PARTIAL_WAKE_LOCK系统进入休眠状态（屏幕可以关闭，但CPU不会休眠）。</p>
<p>了解了上述知识后，再来看如下代码：</p>
<p>[--&gt;PowerManager.java::WakeLock]</p>
<div>
    <p>WakeLock(int flags, String tag)</p>
    <p>{</p>
    <p>     //检查flags参数是否非法</p>
    <p>    mFlags =flags;</p>
    <p>    mTag =tag;</p>
    <p>    //创建一个Binder对象，除了做Token外，PMS需要监视客户端的生死状况，否则有可能导致</p>
    <p>    //WakeLock不能被释放</p>
    <p>     mToken= new Binder();</p>
    <p>}</p>
</div>
<p>客户端创建WakeLock后，需要调用acquire以确保电力资源供应正常。下面对acquire代码进行分析。</p>
<h4>2.  acquire分析</h4>
<p>[--&gt;PowerManager.java::WakeLock.acquire函数]</p>
<div>
    <p>public void acquire()</p>
    <p> {</p>
    <p> synchronized (mToken) {</p>
    <p>  acquireLocked();//调用acquireLocked函数</p>
    <p>  }</p>
    <p> }</p>
    <p>//acquireLoced函数</p>
    <p>private void acquireLocked() {</p>
    <p>  if(!mRefCounted || mCount++ == 0) {</p>
    <p>     mHandler.removeCallbacks(mReleaser);//引用计数控制</p>
    <p>  try {</p>
    <p>      //调用PMS的acquirewakeLock，注意这里传递的参数，其中mWorkSource为空</p>
    <p>     mService.acquireWakeLock(mFlags, mToken, mTag, mWorkSource);</p>
    <p>  }......</p>
    <p>    mHeld =true;</p>
    <p>   }</p>
    <p>}</p>
</div>
<p>上边代码中调用PMS的acquireWakeLock函数与PMS交互，该函数最后一个参数为WorkSource类。这个类从Android 2.2开始就存在，但一直没有明确的作用，下面是关于它的一段说明。</p>
<div>
    <p>/**    见WorkSoure.java</p>
    <p> * Describesthe source of some work that may be done by someone else.</p>
    <p> * Currentlythe public representation of what a work source is is not</p>
    <p> * defined;this is an opaque container.</p>
    <p> */</p>
</div>
<p>由以上注释可知，WorkSource本意用来描述某些任务的Source。传递此Source给其他人，这些人就可以执行该Source对应的工作。目前使用WorkSource的地方仅是ContentService中的SynManager。读者暂时可不理会WorkSource。</p>
<p>客户端的功能比较简单，和PMS仅通过acquireWakeLock函数交互。下面来分析服务端的工作。</p>
<h3>5.3.2  PMSacquireWakeLock分析</h3>
<p>[--&gt;PowerManagerService.java::acquireWakeLock]</p>
<div>
    <p>public void acquireWakeLock(int flags, IBinderlock, String tag, WorkSource ws) {</p>
    <p>        intuid = Binder.getCallingUid();</p>
    <p>        intpid = Binder.getCallingPid();</p>
    <p>        if(uid != Process.myUid()) {</p>
    <p>           mContext.enforceCallingOrSelfPermission(//检查WAKE_LOCK权限</p>
    <p>                          android.Manifest.permission.WAKE_LOCK,null);</p>
    <p>        }</p>
    <p>        if(ws != null) { </p>
    <p>            //如果ws不为空，需要检查调用进程是否有UPDATE_DEVICE_STATS的权限</p>
    <p>           enforceWakeSourcePermission(uid, pid);</p>
    <p>        }</p>
    <p>        longident = Binder.clearCallingIdentity();</p>
    <p>        try{</p>
    <p>           synchronized (mLocks) {调用acquireWakeLockLocked函数</p>
    <p>               acquireWakeLockLocked(flags, lock, uid, pid, tag, ws);</p>
    <p>           }</p>
    <p>        } ......</p>
    <p>    }</p>
</div>
<p>接下来分析acquireWakeLockLocked函数。由于此段代码较长，宜分段来看。</p>
<h4>1.  acquireWakeLockLocked分析之一</h4>
<p>开始分析之前，有必要先介绍另外一个数据结构，它为PowerManagerService的内部类，名字也为WakeLock。其定义如下：</p>
<p>[--&gt;PowerManagerService.java]</p>
<div>
    <p>class WakeLock implements IBinder.DeathRecipient</p>
</div>
<p>PMS的WakeLock实现了DeathRecipient接口。根据前面Binder系统的知识可知，当Binder服务端死亡后，Binder系统会向注册了讣告接收的Binder客户端发送讣告通知，因此客户端可以做一些资源清理工作。在本例中，PM.WakeLock是Binder服务端，而PMS.WakeLock是Binder客户端。假如PM.WakeLock所在进程在release唤醒锁（即WakeLock）之前死亡，PMS.WakeLock的binderDied函数则会被调用，这样，PMS也能及时进行释放（release）工作。对于系统的重要资源来说，采用这种安全保护措施尤其必要。</p>
<p>回到acquireWakeLockLocked函数，先看第一段代码：</p>
<p>[--&gt;PowerManagerService.java::acquireWakeLockLocked]</p>
<div>
    <p>public void acquireWakeLockLocked(int flags,IBinder lock, int uid,</p>
    <p>                        int pid, Stringtag,WorkSource ws) {</p>
    <p>  ......</p>
    <p>  //mLocks是一个ArrayList，保存PMS.WakeLock对象</p>
    <p>  int index= mLocks.getIndex(lock);</p>
    <p>  WakeLockwl;</p>
    <p>  booleannewlock;</p>
    <p>  booleandiffsource;</p>
    <p>  WorkSourceoldsource;</p>
    <p>  if (index&lt; 0) {</p>
    <p>     //创建一个PMS.WakeLock对象，保存客户端acquire传来的参数</p>
    <p>    wl = new WakeLock(flags, lock, tag, uid, pid);</p>
    <p>    switch(wl.flags &amp; LOCK_MASK)</p>
    <p>    {    //将flags转换成对应的minState</p>
    <p>      casePowerManager.FULL_WAKE_LOCK:</p>
    <p>       if(mUseSoftwareAutoBrightness) {</p>
    <p>        wl.minState = SCREEN_BRIGHT;</p>
    <p>       }else {</p>
    <p>         wl.minState = (mKeyboardVisible ? ALL_BRIGHT: SCREEN_BUTTON_BRIGHT);</p>
    <p>        }</p>
    <p>       break;</p>
    <p>      casePowerManager.SCREEN_BRIGHT_WAKE_LOCK:</p>
    <p>        wl.minState = SCREEN_BRIGHT;</p>
    <p>         break;</p>
    <p>       casePowerManager.SCREEN_DIM_WAKE_LOCK:</p>
    <p>        wl.minState = SCREEN_DIM;</p>
    <p>        break;</p>
    <p>       case PowerManager.PARTIAL_WAKE_LOCK:</p>
    <p>       //PROXIMITY_SCREEN_OFF_WAKE_LOCK在SDK中并未输出，原因是有部分手机并没有接近</p>
    <p>       //传感器</p>
    <p>       casePowerManager.PROXIMITY_SCREEN_OFF_WAKE_LOCK:</p>
    <p>        break;</p>
    <p>      default:</p>
    <p>         return;</p>
    <p>      }</p>
    <p>   mLocks.addLock(wl);//将PMS.WakeLock对象保存到mLocks中</p>
    <p>    if (ws!= null) {</p>
    <p>       wl.ws = new WorkSource(ws);</p>
    <p>     }</p>
    <p>     newlock= true;  //设置几个参数信息，newlock表示新创建了一个PMS.WakeLock对象</p>
    <p>    diffsource = false;</p>
    <p>    oldsource = null;</p>
    <p> }else{</p>
    <p>   //如果之前保存有PMS.WakeLock，则要判断新传入的WorkSource和之前保存的WorkSource</p>
    <p>   //是否一样。此处不讨论这种情况</p>
    <p>   ...... </p>
    <p>}</p>
</div>
<p>在上面代码中，很重要一部分是将前面flags信息转成PMS.WakeLock的成员变量minState，下面是对转换关系的总结。</p>
<p>·  FULL_WAKE_LOCK：当启用mUseSoftwareAutoBrightness时，minState为SCREEN_BRIGHT（表示屏幕全亮），否则为ALL_BRIGHT（屏幕、键盘、按键全亮。注意，只有在打开键盘时才能选择此项）或SCREEN_BUTTON_BRIGHT（屏幕、按键全亮）。</p>
<p>·  SCREEN_BRIGHT_WAKE_LOCK：minState为SCREEN_BRIGHT，表示屏幕全亮。</p>
<p>·  SCREEN_DIM_WAKE_LOCK：minState为SCREEN_DIM，表示屏幕Dim。</p>
<p>·  对PARTIAL_WAKE_LOCK和PROXIMITY_SCREEN_OFF_WAKE_LOCK情况不做处理。</p>
<p>该做的准备工作都做了，下面来看第二阶段的工作是什么。</p>
<h4>2.  acquireWakeLockLocked分析之二</h4>
<p>代码如下：</p>
<div>
    <p>  //isScreenLock用于判断flags是否和屏幕有关，除PARTIAL_WAKE_LOCK外，其他WAKE_LOCK</p>
    <p>  //都和屏幕有关</p>
    <p>if (isScreenLock(flags)) {</p>
    <p>  if ((flags&amp; LOCK_MASK) == PowerManager.PROXIMITY_SCREEN_OFF_WAKE_LOCK) {</p>
    <p>      mProximityWakeLockCount++;//引用计数控制</p>
    <p>       if(mProximityWakeLockCount == 1) {</p>
    <p>        enableProximityLockLocked();//使能Proximity传感器</p>
    <p>        }</p>
    <p>   } else {</p>
    <p>   if((wl.flags &amp; PowerManager.ACQUIRE_CAUSES_WAKEUP) != 0) {</p>
    <p>     ......//ACQUIRE_CAUSES_WAKEUP标志处理</p>
    <p>  } else {</p>
    <p>   //①gatherState返回一个状态，稍后分析该函数</p>
    <p>  mWakeLockState = (mUserState | mWakeLockState) &amp;mLocks.gatherState();</p>
    <p>  }</p>
    <p>   //②设置电源状态，</p>
    <p>   setPowerState(mWakeLockState | mUserState);</p>
    <p>   }</p>
    <p> }</p>
</div>
<p>以上代码列出了两个关键函数，一个是gatherState，另外一个是setPowerState，下面来分析它们。</p>
<h5>（1） gatherState分析</h5>
<p>gatherState函数的代码如下：</p>
<p>[--&gt;PowerManagerService.java::gatherState]</p>
<div>
    <p>int gatherState()</p>
    <p>{</p>
    <p>    intresult = 0;</p>
    <p>    int N =this.size();</p>
    <p>    for (inti=0; i&lt;N; i++) {</p>
    <p>     WakeLock wl = this.get(i);</p>
    <p>     if(wl.activated) </p>
    <p>        if(isScreenLock(wl.flags)) </p>
    <p>          result |= wl.minState;//对系统中所有活跃PMS.WakeLock的状态进行或操作</p>
    <p>  }</p>
    <p>   returnresult;</p>
    <p> }</p>
</div>
<p>由以上代码可知，gatherState将统计当前系统内部活跃WakeLock的minState。这里为什么要“使用”或“操作”呢？举个例子，假如WakeLock A的minState为SCREEN_DIM，而WakeLock B的minState为SCREEN_BRIGHT，二者共同作用，最终的屏幕状态显然应该是SCREEN_BRIGHT。</p>
<div>
    <p><strong>提示</strong>读者也可参考PowerManagerService中SCREEN_DIM等变量的定义。</p>
</div>
<p>下面来看setPowerState，本章前面曾两次对该函数避而不谈，现在该见识见识它了。</p>
<h5>（2） setPowerState分析</h5>
<p>setPowerState用于设置电源状态，先来看其在代码中的调用：</p>
<div>
    <p>setPowerState(mWakeLockState | mUserState);</p>
</div>
<p>在以上代码中除了mWakeLockState外，还有一个mUserState。根据前面对gatherState函数的介绍可知，mWakeLockState的值来源于系统当前活跃WakeLock的minState。那么mUserState代表什么呢？</p>
<p>mUserState代表用户触发事件导致的电源状态。例如，按Home键后，将该值设置为SCREEN_BUTTON_BRIGHT（假设手机没有键盘）。很显然，此时系统的电源状态应该是mUserState和mWakeLockState的组合。</p>
<div>
    <p><strong>提示</strong> “一个小小的变量背后代表了一个很重要的case”，读者能体会到吗？</p>
</div>
<p>下面来看setPowerState的代码，这段代码较长，也适合分段来看。第一段代码如下：</p>
<p>[--&gt;PowerManagerService.java::setPowerState]</p>
<div>
    <p>private void setPowerState(int state)</p>
    <p>{//调用另外一个同名函数</p>
    <p> setPowerState(state, false,WindowManagerPolicy.OFF_BECAUSE_OF_TIMEOUT);</p>
    <p>}</p>
    <p>//setPowerState</p>
    <p>private void setPowerState(int newState, booleannoChangeLights, int reason)</p>
    <p>{</p>
    <p> synchronized (mLocks) {</p>
    <p>  int err;</p>
    <p>  if (noChangeLights)//在这种情况中，noChangeLights为false</p>
    <p>    newState = (newState &amp; ~LIGHTS_MASK) | (mPowerState &amp;LIGHTS_MASK);</p>
    <p>  </p>
    <p>  if(mProximitySensorActive)//如果打开了接近感应器，就不需要在这里点亮屏幕了</p>
    <p>    newState = (newState &amp; ~SCREEN_BRIGHT);</p>
    <p> </p>
    <p>  if(batteryIsLow())//判断是否处于低电状态</p>
    <p>     newState |= BATTERY_LOW_BIT;</p>
    <p>   else </p>
    <p>     newState &amp;= ~BATTERY_LOW_BIT;</p>
    <p> ......</p>
    <p>  //如果还没启动完成，则需要将newState置为ALL_BRIGHT。细心的读者有没有发现，在手机开机过程中</p>
    <p>  //键盘、屏幕、按键等都会全部点亮一会儿呢？</p>
    <p>  if(!mBootCompleted &amp;&amp; !mUseSoftwareAutoBrightness) </p>
    <p>      newState |= ALL_BRIGHT;</p>
    <p>   booleanoldScreenOn = (mPowerState &amp; SCREEN_ON_BIT) != 0;</p>
    <p>   boolean newScreenOn = (newState &amp;SCREEN_ON_BIT) != 0;</p>
    <p> </p>
    <p>   finalboolean stateChanged = mPowerState != newState;</p>
</div>
<p>第一段代码主要用于得到一些状态值，例如在新状态下屏幕是否需要点亮（newScreenOn）等。再来看第二段代码，它将根据第一段的状态值完成对应的工作。</p>
<p>[--&gt;PowerManagerService::setPowerState]</p>
<div>
    <p>   if(oldScreenOn != newScreenOn) {</p>
    <p>      if(newScreenOn) {</p>
    <p>         if(mStillNeedSleepNotification) {</p>
    <p>            //对sendNotificationLocked函数的分析见后文</p>
    <p>            sendNotificationLocked(false,</p>
    <p>                                      WindowManagerPolicy.OFF_BECAUSE_OF_USER);</p>
    <p>        }// mStillNeedSleepNotification判断</p>
    <p>     booleanreallyTurnScreenOn = true;</p>
    <p>     if(mPreventScreenOn)// mPreventScreenOn是何方神圣？</p>
    <p>         reallyTurnScreenOn= false;</p>
    <p>    if(reallyTurnScreenOn) {</p>
    <p>     err = setScreenStateLocked(true);//点亮屏幕</p>
    <p>     ......//通知mBatteryStats做电量统计</p>
    <p>       mBatteryStats.noteScreenBrightness(getPreferredBrightness());</p>
    <p>      mBatteryStats.noteScreenOn();</p>
    <p>   } else {//reallyTurnScreenOn为false</p>
    <p>      setScreenStateLocked(false);//关闭屏幕</p>
    <p>       err =0;</p>
    <p>   }</p>
    <p>    if (err == 0) {</p>
    <p>     sendNotificationLocked(true, -1);</p>
    <p>      if(stateChanged)</p>
    <p>          updateLightsLocked(newState, 0);//点亮按键灯或者键盘灯</p>
    <p>     mPowerState |= SCREEN_ON_BIT;</p>
    <p>  }</p>
    <p> }</p>
</div>
<p>以上代码看起来比较简单，就是根据情况点亮或关闭屏幕。事实果真的如此吗？的还记得前面所说“一个小小的变量背后代表一个很重要的case”这句话吗？是的，这里也有一个很重要的case，由mPreventScreenOn表达。这是什么意思呢？</p>
<p>PMS提供了一个函数叫preventScreenOn，该函数（在SDK中未公开）使应用程序可以阻止屏幕点亮。为什么会有这种操作呢？难道是因为该应用很丑，以至于不想让别人看见？根据该函数的解释，在两个应用之间进行切换时（尤其是正在启动一个Activity却又接到来电通知时），很容易出现闪屏现象，会严重影响用户体验。因此提供了此函数，由应用来调用并处理它。</p>
<div>
    <p><strong>注意</strong>闪屏的问题似乎解决了，但事情还没完，这个解决方案还引入了另外一个问题：假设应用忘记重新使屏幕点亮，手机岂不是一直就黑屏了？为此，在代码中增加了一段处理逻辑，即如果5秒钟后应用还没有使屏幕点亮，PMS将自己设置mPreventScreenOn为false。</p>
    <p>Google怎么会写这种代码？还好，代码开发者也意识到这是一个很难看的方法，只是目前还没有一个比较完美的解决方案而已。</p>
</div>
<p>继续看setPowerState最后的代码：</p>
<div>
    <p>  else {//newScreenOn为false的情况</p>
    <p>    ......//更新键盘灯、按键灯的状态</p>
    <p>   //从mHandler中移除mAutoBrightnessTask，这和光传感器有关。此处不讨论</p>
    <p>    mHandler.removeCallbacks(mAutoBrightnessTask);</p>
    <p>    mBatteryStats.noteScreenOff();//通知BatteryStatsService,屏幕已关</p>
    <p>   mPowerState = (mPowerState &amp; ~LIGHTS_MASK) | (newState &amp; LIGHTS_MASK);</p>
    <p>   updateNativePowerStateLocked();</p>
    <p>   } </p>
    <p>  }//if(oldScreenOn != newScreenOn)判断结束</p>
    <p>  else if(stateChanged) {//屏幕的状态不变，但是light的状态有可能变化，所以</p>
    <p>  updateLightsLocked(newState, 0);//单独更新light的状态</p>
    <p>   }</p>
    <p>  mPowerState= (mPowerState &amp; ~LIGHTS_MASK) | (newState &amp; LIGHTS_MASK);</p>
    <p>  updateNativePowerStateLocked();</p>
    <p>}//setPowerState完毕</p>
</div>
<p>setPowerState函数是在PMS中真正设置屏幕及Light状态的地方，其内部将通过Power类与这些硬件交互。相关内容见5.3.3节。</p>
<h5>（3） sendNotificationLocked函数分析</h5>
<p>sendNotificationLocked函数用于触发SCREEN_ON/OFF广播的发送，来看以下代码：</p>
<p>[--&gt;PowerManagerService.java::sendNotificationLocked]</p>
<div>
    <p>private void sendNotificationLocked(boolean on,int why) {</p>
    <p>  ......</p>
    <p>  if (!on) {</p>
    <p>    mStillNeedSleepNotification = false;</p>
    <p>  }</p>
    <p>  int index= 0;</p>
    <p>  while(mBroadcastQueue[index] != -1) {</p>
    <p>       index++;</p>
    <p>  }</p>
    <p>  // mBroadcastQueue和mBroadcastWhy均定义为int数组，成员个数为3，它们有什么作用呢</p>
    <p> mBroadcastQueue[index] = on ? 1 : 0;</p>
    <p> mBroadcastWhy[index] = why;</p>
    <p>  /* mBroadcastQueue数组一共有3个元素，根据代码中的注释，其作用如下：</p>
    <p>    当取得的index为2时，即0,1元素已经有值，由于屏幕ON/OFF请求是配对的，所以在这种情况</p>
    <p>    下只需要处理最后一次的请求。例如0元素为ON，1元素为OFF，2元素为ON，则可以去掉0，</p>
    <p>    1的请求，而直接处理2的请求，即屏幕ON。对于那种频繁按Power键的操作，通过这种方式可以</p>
    <p>    节省一次切换操作</p>
    <p>  */</p>
    <p>  if (index== 2) {</p>
    <p>     if (!on&amp;&amp; mBroadcastWhy[0] &gt; why) mBroadcastWhy[0] = why;</p>
    <p>     //处理index为2的情况，见上文的说明</p>
    <p>    mBroadcastQueue[0] = on ? 1 : 0;</p>
    <p>    mBroadcastQueue[1] = -1;</p>
    <p>    mBroadcastQueue[2] = -1;</p>
    <p>     mBroadcastWakeLock.release();</p>
    <p>     index =0;</p>
    <p>   }</p>
    <p>   /*</p>
    <p>     如果index为1，on为false，即屏幕发出关闭请求，则无需处理。根据注释中的说明，</p>
    <p>     在此种情况，屏幕已经处于OFF状态，所以无需处理。为什么在此种情况下屏幕已经关闭了呢？ </p>
    <p>   */</p>
    <p>   if (index== 1 &amp;&amp; !on) {</p>
    <p>       mBroadcastQueue[0] = -1;</p>
    <p>       mBroadcastQueue[1] = -1;</p>
    <p>       index = -1;</p>
    <p>       mBroadcastWakeLock.release();</p>
    <p>   }</p>
    <p> </p>
    <p>   if(mSkippedScreenOn) {</p>
    <p>      updateLightsLocked(mPowerState, SCREEN_ON_BIT);</p>
    <p>    }</p>
    <p>   //如果index不为负数，则抛送mNotificationTask给mHandler处理</p>
    <p>   if (index&gt;= 0) {</p>
    <p>      mBroadcastWakeLock.acquire();</p>
    <p>       mHandler.post(mNotificationTask);</p>
    <p>    }</p>
    <p> }</p>
</div>
<p>sendNotificationLocked函数相当诡异，主要是mBroadcastQueue数组的使用让人感到困惑。其目的在于减少不必要的屏幕切换和广播发送，但是为什么index为1时，屏幕处于OFF状态呢？下面来分析mNotificationTask，希望它能回答这个问题。</p>
<p>[--&gt;PowerManagerService.java::mNotificationTask]</p>
<div>
    <p>private Runnable mNotificationTask = newRunnable()</p>
    <p>{</p>
    <p>  publicvoid run()</p>
    <p> {</p>
    <p>   while(true) {//此处是一个while循环</p>
    <p>    intvalue;</p>
    <p>    int why;</p>
    <p>   WindowManagerPolicy policy;</p>
    <p>   synchronized (mLocks) {</p>
    <p>       value =mBroadcastQueue[0];//取mBroadcastQueue第一个元素</p>
    <p>       why= mBroadcastWhy[0];</p>
    <p>       for(int i=0; i&lt;2; i++) {//将后面的元素往前挪一位</p>
    <p>           mBroadcastQueue[i] = mBroadcastQueue[i+1];</p>
    <p>           mBroadcastWhy[i] = mBroadcastWhy[i+1];</p>
    <p>        }</p>
    <p>      policy = getPolicyLocked();//policy指向PhoneWindowManager</p>
    <p>      if(value == 1 &amp;&amp; !mPreparingForScreenOn) {</p>
    <p>             mPreparingForScreenOn = true;</p>
    <p>              mBroadcastWakeLock.acquire();</p>
    <p>         }</p>
    <p>      }// synchronized结束</p>
    <p>    if(value == 1) {//value为1，表示发出屏幕ON请求</p>
    <p>       mScreenOnStart = SystemClock.uptimeMillis();</p>
    <p>        //和WindowManagerService交互，和锁屏界面有关</p>
    <p>         //mScreenOnListener为回调通知对象</p>
    <p>         policy.screenTurningOn(mScreenOnListener); </p>
    <p>         ActivityManagerNative.getDefault().wakingUp();//和AMS交互</p>
    <p>         if (mContext != null &amp;&amp;ActivityManagerNative.isSystemReady()) {</p>
    <p>           //发送SCREEN_ON广播</p>
    <p>            mContext.sendOrderedBroadcast(mScreenOnIntent,null,</p>
    <p>              mScreenOnBroadcastDone, mHandler, 0, null, null);</p>
    <p>        }......</p>
    <p>      }elseif (value == 0) {</p>
    <p>         mScreenOffStart = SystemClock.uptimeMillis();</p>
    <p>          policy.screenTurnedOff(why);//通知WindowManagerService</p>
    <p>          ActivityManagerNative.getDefault().goingToSleep();//和AMS交互</p>
    <p>           if(mContext != null &amp;&amp; ActivityManagerNative.isSystemReady()) {</p>
    <p>                        //发送屏幕OFF广播</p>
    <p>                mContext.sendOrderedBroadcast(mScreenOffIntent, null,</p>
    <p>                                mScreenOffBroadcastDone, mHandler, 0, null,null);</p>
    <p>            } </p>
    <p>       }elsebreak；</p>
    <p>     }</p>
    <p> };</p>
</div>
<p>mNotificationTask比较复杂，但是它对mBroadcastQueue的处理比较有意思，每次取出第一个元素值后，将后续元素往前挪一位。这种处理方式能解决之前提出的那个问题吗？</p>
<p>说实话，目前笔者也没找到能解释index为1时，屏幕一定处于OFF的证据。如果有哪位读者找到证据，不妨分享一下。</p>
<p>另外，mNotificationTask和ActivityManagerService及WindowManagerService都有交互。因为这两个服务内部也使用了WakeLock，所以需要通知它们释放WakeLock，否则会导致不必要的电力资源消耗。具体内容只能留待以后分析相关服务时再来讨论了。</p>
<h5>（4） acquireWakeLocked第二阶段工作总结</h5>
<p>acquireWakeLocked第二阶段工作是处理和屏幕相关的WAKE_LOCK方面的工作（isScreenLock返回为true的情况）。其中一个重要的函数就是setPowerState，该函数将根据不同的状态设置屏幕光、键盘灯等硬件设备。注意，和硬件交互相关的工作是通过Power类提供的接口完成的。</p>
<h4>3. acquireWakeLocked分析之三</h4>
<p>acquireWakeLocked处理WAKE_LOCK为PARTIAL_WAKE_LOCK的情况。来看以下代码：</p>
<p>[--&gt;PowerManagerService.java::acquiredWakeLockLocked]</p>
<div>
    <p>else if ((flags &amp; LOCK_MASK) == PowerManager.PARTIAL_WAKE_LOCK){</p>
    <p>    if(newlock) {</p>
    <p>   mPartialCount++;</p>
    <p>   }</p>
    <p>   //获取kernel层的PARTIAL_WAKE_LOCK，该函数后续再分析</p>
    <p>   Power.acquireWakeLock(Power.PARTIAL_WAKE_LOCK,PARTIAL_NAME);</p>
    <p>  }//else if判断结束</p>
    <p>   if(diffsource) {</p>
    <p>   noteStopWakeLocked(wl, oldsource);</p>
    <p>  }</p>
    <p>  if(newlock || diffsource) {</p>
    <p>      noteStartWakeLocked(wl, ws);//通知BatteryStatsService做电量统计</p>
    <p> }</p>
</div>
<p>当客户端使用PARTIAL_WAKE_LOCK时，PMS会调用Power.acquireWakeLock申请一个内核的WakeLock。</p>
<h4>4.  acquireWakeLock总结</h4>
<p>acquireWakeLock有三个阶段的工作，总结如下：</p>
<p>·  如果对应的WakeLock不存在，则创建一个WakeLock对象，同时将WAKE_LOCK标志转换成对应的minState；否则，从mLocks中查找对应的WakeLock对象，然后更新其中的信息。</p>
<p>·  当WAKE_LOCK标志和屏幕有关时，需要做相应的处理，例如点亮屏幕、打开按键灯等。实际上这些工作不仅影响电源管理，还会影响到用户感受，所以其中还穿插了一些和用户体验有关的处理逻辑（如上面注释的mPreventScreenOn变量）。</p>
<p>·  当WAKE_LOCK和PARTIAL_WAKE_LOCK有关时，仅简单调用Power的acquireWakeLock即可，其中涉及和Linux Kernel电源管理系统的交互。</p>
<h3>5.3.3  Power类及LightService类介绍</h3>
<p>根据前面的分析，PMS有时需要进行点亮屏幕，打开键盘灯等操作，为此Android提供了Power类及LightService满足PMS的要求。这两个类比较简单，但是其背后的Kernel层相对复杂一些。本章仅分析用户空间的内容，有兴趣的读者不妨以此为入口，深入研究Kernel层的实现。</p>
<h4>1.  Power类介绍</h4>
<p>Power类提供了6个函数，如下所示：</p>
<p>[--&gt;Power.java]</p>
<div>
    <p>int setScreenState(boolean on);//打开或关闭屏幕光</p>
    <p>int setLastUserActivityTimeout(long ms);//设置超时时间</p>
    <p>void reboot(String reason);//用于手机重启，内部调用rebootNative</p>
    <p>void shutdown();//已作废，建议不要调用</p>
    <p>void acquireWakeLock(int lock, String id);//获取Kernel层的WakeLock</p>
    <p>void releaseWakeLock(String id);//释放Kernel层的WakeLock</p>
</div>
<p>这些函数固有的实现代码如下：</p>
<p>[--&gt;android_os_Power.cpp]</p>
<div>
    <p>static void acquireWakeLock(JNIEnv *env, jobjectclazz, jint lock, jstring idObj)</p>
    <p>{</p>
    <p> ......</p>
    <p>    constchar *id = env-&gt;GetStringUTFChars(idObj, NULL);</p>
    <p>   acquire_wake_lock(lock, id);//调用此函数和Kernel层交互</p>
    <p>   env-&gt;ReleaseStringUTFChars(idObj, id);</p>
    <p>}</p>
    <p> </p>
    <p>static void releaseWakeLock(JNIEnv *env, jobjectclazz, jstring idObj)</p>
    <p>{</p>
    <p>    constchar *id = env-&gt;GetStringUTFChars(idObj, NULL);</p>
    <p>   release_wake_lock(id);//释放Kernel层的WakeLock</p>
    <p>    env-&gt;ReleaseStringUTFChars(idObj,id);</p>
    <p>}</p>
    <p> </p>
    <p>static int setLastUserActivityTimeout(JNIEnv *env,jobject clazz, jlong timeMS)</p>
    <p>{</p>
    <p>    returnset_last_user_activity_timeout(timeMS/1000);//设置超时时间</p>
    <p>}</p>
    <p> </p>
    <p>static int setScreenState(JNIEnv *env, jobjectclazz, jboolean on)</p>
    <p>{</p>
    <p>    return set_screen_state(on);//开启或关闭屏幕光</p>
    <p>}</p>
    <p> </p>
    <p>static void android_os_Power_shutdown(JNIEnv *env,jobject clazz)</p>
    <p>{</p>
    <p>   android_reboot(ANDROID_RB_POWEROFF, 0, 0);//关机</p>
    <p>}</p>
    <p> </p>
    <p>static void android_os_Power_reboot(JNIEnv *env,jobject clazz, jstring reason)</p>
    <p>{</p>
    <p>    if (reason== NULL) {</p>
    <p>       android_reboot(ANDROID_RB_RESTART, 0, 0);//重启</p>
    <p>    } else {</p>
    <p>       const char *chars = env-&gt;GetStringUTFChars(reason, NULL);</p>
    <p>       android_reboot(ANDROID_RB_RESTART2, 0, (char *) chars);//重启</p>
    <p>       env-&gt;ReleaseStringUTFChars(reason, chars);</p>
    <p>    }</p>
    <p>   jniThrowIOException(env, errno);</p>
    <p>}</p>
</div>
<p>Power类提供了和内核交互的通道，读者仅作了解即可。</p>
<h4>2.  LightService介绍</h4>
<p>LightService.java比较简单，这里直接介绍Native层的实现，主要关注HAL层的初始化函数init_native及操作函数setLight_native。</p>
<p>首先来看初始化函数init_native，其代码如下：</p>
<p>[com_android_server_LightService.cpp::init_native]</p>
<div>
    <p>static jint init_native(JNIEnv *env, jobjectclazz)</p>
    <p>{</p>
    <p>    int err;</p>
    <p>   hw_module_t* module;</p>
    <p>    Devices*devices;</p>
    <p>    </p>
    <p>    devices= (Devices*)malloc(sizeof(Devices));</p>
    <p>    //初始化硬件相关的模块，模块名为“lights”</p>
    <p>    err =hw_get_module(LIGHTS_HARDWARE_MODULE_ID,</p>
    <p>                             (hw_module_tconst**)&amp;module);</p>
    <p>    if (err== 0) {</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_BACKLIGHT]//背光</p>
    <p>               = get_device(module, LIGHT_ID_BACKLIGHT);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_KEYBOARD]//键盘灯</p>
    <p>               = get_device(module, LIGHT_ID_KEYBOARD);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_BUTTONS]//按键灯</p>
    <p>               = get_device(module, LIGHT_ID_BUTTONS);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_BATTERY]//电源指示灯</p>
    <p>               = get_device(module, LIGHT_ID_BATTERY);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_NOTIFICATIONS] //通知灯</p>
    <p>               = get_device(module, LIGHT_ID_NOTIFICATIONS);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_ATTENTION] //警示灯</p>
    <p>               = get_device(module, LIGHT_ID_ATTENTION);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_BLUETOOTH] //蓝牙提示灯</p>
    <p>               = get_device(module, LIGHT_ID_BLUETOOTH);</p>
    <p>       devices-&gt;lights[LIGHT_INDEX_WIFI] //WIFI提示灯</p>
    <p>               = get_device(module, LIGHT_ID_WIFI);</p>
    <p>    } else {</p>
    <p>       memset(devices, 0, sizeof(Devices));</p>
    <p>    }</p>
    <p> </p>
    <p>    return(jint)devices;</p>
    <p>}</p>
</div>
<p>Android系统想得很周到，提供了多达8种不同类型的灯。可是有多少手机包含了所有的灯呢？</p>
<p>PMS点亮或关闭灯时，将调用setLight_native函数，其代码如下：</p>
<p>[com_android_server_LightService.cpp::setLight_native]</p>
<div>
    <p>static void setLight_native(JNIEnv *env, jobjectclazz, int ptr,</p>
    <p>        intlight, int colorARGB, int flashMode, int onMS, int offMS, </p>
    <p>        intbrightnessMode)</p>
    <p>{</p>
    <p>    Devices*devices = (Devices*)ptr;</p>
    <p>   light_state_t state;</p>
    <p>    ......</p>
    <p>   memset(&amp;state, 0, sizeof(light_state_t));</p>
    <p>   state.color = colorARGB;   //设置颜色</p>
    <p>   state.flashMode = flashMode; //设置闪光模式</p>
    <p>   state.flashOnMS = onMS;  //和闪光模式有关，例如亮2秒，灭2秒</p>
    <p>   state.flashOffMS = offMS; </p>
    <p>   state.brightnessMode = brightnessMode;//</p>
    <p>    //传递给HAL层模块进行处理</p>
    <p>   devices-&gt;lights[light]-&gt;set_light(devices-&gt;lights[light],&amp;state);</p>
    <p>}</p>
</div>
<h3>5.3.4  WakeLock总结</h3>
<p>相信读者此时已经对WakeLock机制有了比较清晰的认识，此处以flags标签为出发点，对WakeLock的知识点进行总结。</p>
<p>·  如果flags和屏幕有关（即除PARTIAL_WAKE_LOCK外），则需要更新屏幕、灯光状态。其中，屏幕操作通过Power类完来成，灯光操作则通过LightService类来完成。</p>
<p>·  如果FLAGS是PARTIAL_WAKE_LOCK，则需要通过Power提供的接口获取Kernel层的WakeLock。</p>
<p>·  在WakeLock工作流程中还混杂了用户体验、光传感器、接近传感器方面的处理逻辑。这部分代码集中体现在setPowerState函数中。感兴趣的读者可进行深入研究。</p>
<p>·  WakeLock还要通知BatteryStatsService，以帮助其统计电量使用情况。这方面内容放到本章最后再做分析。</p>
<p>另外，PMS在JNI层也保存了当前屏幕状态信息，这是通过updateNativePowerStateLocked完成的，其代码如下：</p>
<div>
    <p>private void updateNativePowerStateLocked() {</p>
    <p>       nativeSetPowerState(//调用native函数，传入两个参数</p>
    <p>               (mPowerState &amp; SCREEN_ON_BIT) != 0,</p>
    <p>               (mPowerState &amp; SCREEN_BRIGHT) == SCREEN_BRIGHT);</p>
    <p>    }</p>
    <p>//jni层实现代码如下</p>
    <p>static void android_server_PowerManagerService_nativeSetPowerState(</p>
    <p>       JNIEnv* env,jobject serviceObj, jboolean screenOn, jbooleanscreenBright) {</p>
    <p>   AutoMutex _l(gPowerManagerLock);</p>
    <p>   gScreenOn = screenOn;//屏幕是否开启</p>
    <p>   gScreenBright = screenBright; //屏幕光是否全亮</p>
    <p>}</p>
</div>
<p>PMS的updateNativePowerStateLocked函数曾一度让笔者感到非常困惑，主要原因是初看此函数名，感觉它极可能会和Kernel层的电源管理系统交互。等深入JNI层代码后发现，其功能仅是保存两个全局变量，和Kernel压根儿没有关系。其实，和Kernel层电源管理系统交互的主要是Power类。此处的两个变量是为了方便Native层代码查询当前屏幕状态而设置的，以后分析Andorid输入系统时就会搞清楚它们的作用了。</p>
<h2>5.4  userActivity及Power按键处理分析</h2>
<p>本节介绍userActivity函数及PMS对Power按键的处理流程。</p>
<h3>5.4.1  userActivity分析</h3>
<p>前面曾经提到过userActivity的作用，此处举一个例子加深读者对它的印象：</p>
<p>打开手机，并解锁进入桌面。如果在规定时间内不操作手机，那么屏幕将变暗，最后关闭。在此过程中，如果触动屏幕，屏幕又会重新变亮。这个触动屏幕的操作将导致userActivity函数被调用。</p>
<p>在上述例子中实际上包含了两方面的内容：</p>
<p>·  不操作手机，屏幕将变暗，最后关闭。在PMS中，这是一个状态切换的过程。</p>
<p>·  操作手机，将触发userActivity，此后屏幕的状态将重置。</p>
<p>来看以下代码：</p>
<p>[--&gt;PowerManagerService.java::userActivity]</p>
<div>
    <p> public voiduserActivity(long time, boolean noChangeLights) {</p>
    <p>    ......//检查调用进程是否有DEVICE_POWER的权限</p>
    <p>   userActivity(time, -1, noChangeLights, OTHER_EVENT, false);</p>
    <p> }</p>
</div>
<p>此处将调用另外一个同名函数。注意第三个参数的值OTHER_EVENT。系统一共定义了三种事件，分别是OTHER_EVENT（除按键、触摸屏外的事件）、BUTTON_EVENT（按键事件）和TOUCH_EVENT（触摸屏事件）。它们主要为BatteryStatsService进行电量统计时使用，例如触摸屏事件的耗电量和按键事件的耗电量等。</p>
<p>[--&gt;PowerManagerService.java::userActivity]</p>
<div>
    <p>private void userActivity(long time, long timeoutOverride,</p>
    <p>              boolean noChangeLights,inteventType, boolean force) {</p>
    <p> </p>
    <p>   if(((mPokey &amp; POKE_LOCK_IGNORE_TOUCH_EVENTS) != 0) &amp;&amp;</p>
    <p>                 (eventType == TOUCH_EVENT)) {</p>
    <p>   //mPokey和输入事件的处理策略有关。如果此处的if判断得到满足，表示忽略TOUCH_EVENT</p>
    <p>   return;</p>
    <p>  }</p>
    <p> </p>
    <p>   synchronized (mLocks) {</p>
    <p>     if(isScreenTurningOffLocked()) {</p>
    <p>          return;</p>
    <p>      }</p>
    <p>    if(mProximitySensorActive &amp;&amp; mProximityWakeLockCount == 0)</p>
    <p>          mProximitySensorActive = false;//控制接近传感器</p>
    <p>    if(mLastEventTime &lt;= time || force) {</p>
    <p>         mLastEventTime = time;</p>
    <p>          if((mUserActivityAllowed &amp;&amp; !mProximitySensorActive) || force) {</p>
    <p>               if (eventType == BUTTON_EVENT &amp;&amp; !mUseSoftwareAutoBrightness) {</p>
    <p>                     mUserState =(mKeyboardVisible ? ALL_BRIGHT :</p>
    <p>                                      SCREEN_BUTTON_BRIGHT);</p>
    <p>                   } else {</p>
    <p>                        mUserState |=SCREEN_BRIGHT;//设置用户事件导致的mUserState</p>
    <p>                   }</p>
    <p>                        ......//通知BatteryStatsService进行电量统计</p>
    <p>                        mBatteryStats.noteUserActivity(uid,eventType);</p>
    <p>               //重新计算WakeLock状态</p>
    <p>                mWakeLockState = mLocks.reactivateScreenLocksLocked();</p>
    <p>               setPowerState(mUserState | mWakeLockState, noChangeLights,</p>
    <p>                           WindowManagerPolicy.OFF_BECAUSE_OF_USER);</p>
    <p>                //重新开始屏幕计时</p>
    <p>                setTimeoutLocked(time, timeoutOverride, SCREEN_BRIGHT);</p>
    <p>               }</p>
    <p>           }</p>
    <p>        }</p>
    <p>        //mPolicy指向PhoneWindowManager，用于和WindowManagerService交互</p>
    <p>        if(mPolicy != null) {</p>
    <p>           mPolicy.userActivity();</p>
    <p>        }</p>
    <p>    }</p>
</div>
<p>有了前面分析的基础，相信很多读者都会觉得userActivity函数很简单。在前面的代码中，通过setPowerState点亮了屏幕，那么经过一段时间后发生的屏幕状态切换在哪儿进行呢？来看setTimeoutLocked函数的代码：</p>
<p>[--&gt;PowerManagerService.java::setTimeoutLocked]</p>
<div>
    <p>private void setTimeoutLocked(long now, final longoriginalTimeoutOverride,</p>
    <p>                                    intnextState) {</p>
    <p>   //在本例中，nextState为SCREEN_BRIGHT，originalTimeoutOverride为-1</p>
    <p>   longtimeoutOverride = originalTimeoutOverride;</p>
    <p>   if(mBootCompleted) {</p>
    <p>       synchronized (mLocks) {</p>
    <p>        long when = 0;</p>
    <p>         if(timeoutOverride &lt;= 0) {</p>
    <p>            switch (nextState)</p>
    <p>            {</p>
    <p>               case SCREEN_BRIGHT:</p>
    <p>                 when = now + mKeylightDelay;//得到一个超时时间</p>
    <p>                  break;</p>
    <p>               case SCREEN_DIM:</p>
    <p>                 if (mDimDelay &gt;= 0) {</p>
    <p>                     when = now + mDimDelay;</p>
    <p>                      break;</p>
    <p>                  } ......</p>
    <p>                case SCREEN_OFF:</p>
    <p>                  synchronized (mLocks) {</p>
    <p>                      when = now +mScreenOffDelay;</p>
    <p>                     }</p>
    <p>                       break;</p>
    <p>                  default:</p>
    <p>                      when = now;</p>
    <p>                      break;</p>
    <p>           }</p>
    <p>        }......//处理timeoutOverride大于零的情况，无非就是设置状态和超时时间</p>
    <p>      mHandler.removeCallbacks(mTimeoutTask);</p>
    <p>      mTimeoutTask.nextState = nextState;</p>
    <p>      mTimeoutTask.remainingTimeoutOverride = timeoutOverride &gt; 0</p>
    <p>                        ? (originalTimeoutOverride- timeoutOverride)</p>
    <p>                        : -1;</p>
    <p>       //抛送一个mTimeoutTask交给mHandler执行，执行时间为when秒后</p>
    <p>      mHandler.postAtTime(mTimeoutTask, when);</p>
    <p>      mNextTimeout = when; //调试用</p>
    <p>      }</p>
    <p>    }</p>
    <p>}</p>
</div>
<p>接下来看mTimeOutTask的代码：</p>
<div>
    <p>private class TimeoutTask implements Runnable</p>
    <p>{</p>
    <p>   intnextState;</p>
    <p>   longremainingTimeoutOverride;</p>
    <p>   publicvoid run()</p>
    <p>   {</p>
    <p>     synchronized (mLocks) {</p>
    <p>        if(nextState == -1)return;</p>
    <p>        </p>
    <p>       mUserState = this.nextState;</p>
    <p>        //调用setPowerState去真正改变屏幕状态</p>
    <p>        setPowerState(this.nextState| mWakeLockState);</p>
    <p> </p>
    <p>        long now = SystemClock.uptimeMillis();</p>
    <p>        switch (this.nextState)</p>
    <p>         {</p>
    <p>           case SCREEN_BRIGHT:</p>
    <p>            if (mDimDelay &gt;= 0) {//设置下一个状态为SCREEN_DIM</p>
    <p>                setTimeoutLocked(now,remainingTimeoutOverride, SCREEN_DIM);</p>
    <p>                break;</p>
    <p>             }</p>
    <p>          case SCREEN_DIM://设置下一个状态为SCREEN_OFF</p>
    <p>            setTimeoutLocked(now, remainingTimeoutOverride, SCREEN_OFF);</p>
    <p>            break;</p>
    <p>         }......//省略花括号</p>
    <p> }</p>
</div>
<p>TimeoutTask就是用来切换屏幕状态的，相信不少读者已经在网络上见过一个和PMS屏幕状态切换相关的图（其实就是TimeoutTask的工作流程解释），对此，本章就不再介绍了，希望读者能通过直接阅读源码加深理解。</p>
<h3>5.4.2  Power按键处理分析</h3>
<p>按键处理属于本书后续将会分析的输入系统的范围，此处摘出和Power键相关的代码进行分析，代码如下：</p>
<p>[--&gt;com_android_server_InputManager.cpp::handleInterceptActions]</p>
<div>
    <p>voidNativeInputManager::handleInterceptActions(jint wmActions, nsecs_t when,</p>
    <p>                uint32_t&amp; policyFlags) {</p>
    <p>      //按下Power键并松开后，将设置wmActions为WM_ACTION_GO_TO_SLEEP，表示需要休眠</p>
    <p>      if(wmActions &amp; WM_ACTION_GO_TO_SLEEP) {</p>
    <p>      //利用JNI调用PMS的goToSleep函数</p>
    <p>     android_server_PowerManagerService_goToSleep(when);</p>
    <p>   }</p>
    <p>  //一般的输入事件将触发userActivity函数被调用，此时将唤醒手机</p>
    <p>  if(wmActions &amp; WM_ACTION_POKE_USER_ACTIVITY) {</p>
    <p>      //利用JNI调用PMS的userActivity函数。相关内容在前一节已经分析过了</p>
    <p>     android_server_PowerManagerService_userActivity(when,</p>
    <p>                                  POWER_MANAGER_BUTTON_EVENT);</p>
    <p>    }</p>
    <p>      ......//其他处理</p>
    <p>}</p>
</div>
<p>由以上代码中的注释可知，当按下Power键并松开时<a>[②]</a>，将触发PMS的goToSleep函数被调用。下面来看goToSleep函数的代码：</p>
<p>[--&gt;PowerManagerService.java::goToSleep]</p>
<div>
    <p>public void goToSleep(long time)</p>
    <p>{</p>
    <p>    goToSleepWithReason(time,WindowManagerPolicy.OFF_BECAUSE_OF_USER);</p>
    <p>}</p>
    <p>public void goToSleepWithReason(long time, intreason)</p>
    <p>{</p>
    <p>  mContext.enforceCallingOrSelfPermission(//检查调用进程是否有DEVICE_POWER权限</p>
    <p>            android.Manifest.permission.DEVICE_POWER,null);</p>
    <p>  synchronized (mLocks) {</p>
    <p>        goToSleepLocked(time, reason);//调用goToSleepLocked函数</p>
    <p>   }</p>
    <p> }</p>
</div>
<p>[--&gt;PowerManagerService.java::goToSleepLocked]</p>
<div>
    <p>private void goToSleepLocked(long time, intreason) {</p>
    <p> if(mLastEventTime &lt;= time) {</p>
    <p>    mLastEventTime = time;</p>
    <p>    mWakeLockState = SCREEN_OFF;</p>
    <p>     int N= mLocks.size();</p>
    <p>     intnumCleared = 0;</p>
    <p>     boolean proxLock = false;</p>
    <p>     for(int i=0; i&lt;N; i++) {</p>
    <p>      WakeLock wl = mLocks.get(i);</p>
    <p>       if(isScreenLock(wl.flags)) {</p>
    <p>          if(((wl.flags &amp; LOCK_MASK) ==</p>
    <p>                PowerManager.PROXIMITY_SCREEN_OFF_WAKE_LOCK)</p>
    <p>               &amp;&amp; reason == WindowManagerPolicy.OFF_BECAUSE_OF_PROX_SENSOR) {</p>
    <p>                proxLock = true;//判断goToSleep的原因是否与接近传感器有关</p>
    <p>            } else{</p>
    <p>               mLocks.get(i).activated = false;//禁止和屏幕相关的WakeLock</p>
    <p>               numCleared++;</p>
    <p>             }</p>
    <p>        }// isScreenLock判断结束</p>
    <p>        }//for循环结束</p>
    <p>       if(!proxLock) {</p>
    <p>           mProxIgnoredBecauseScreenTurnedOff = true;</p>
    <p>       }</p>
    <p>       mStillNeedSleepNotification = true;</p>
    <p>       mUserState = SCREEN_OFF;</p>
    <p>       setPowerState(SCREEN_OFF, false, reason);//关闭屏幕</p>
    <p>       cancelTimerLocked();//从mHandler中撤销mTimeoutTask任务</p>
    <p>    }</p>
    <p> }</p>
</div>
<p>掌握了前面的基础知识就会感到Power键的处理流程真的是很简单，读者是否也有同感呢？</p>
<p> </p>
<h2>5.5  BatteryService及BatteryStatsServic分析</h2>
<p>从前面介绍PMS的代码中发现，PMS和系统中其他两个服务BatterService及BatteryStatsService均有交互，其中：</p>
<p>·  BatteryService提供接口用于获取电池信息，充电状态等。</p>
<p>·  BatteryStatsService主要用做用电统计，通过它可知谁是系统中的耗电大户。</p>
<p>下面先来介绍稍简单的BatteryService。</p>
<h3>5.5.1 BatteryService分析</h3>
<p>BatteryService由SystemServer创建，代码如下：</p>
<div>
    <p>battery = new BatteryService(context, lights);</p>
    <p>ServiceManager.addService("battery",battery);</p>
</div>
<p>下面来看BatteryService的构造函数：</p>
<p>[--&gt;BatteryService.java]</p>
<div>
    <p>public BatteryService(Context context,LightsService lights) {</p>
    <p>  mContext =context;</p>
    <p>  mLed = newLed(context, lights);//提示灯控制，感兴趣的读者可自行阅读相关代码</p>
    <p>  //BatteryService也需要和BatteryStatsService交互</p>
    <p> mBatteryStats = BatteryStatsService.getService();</p>
    <p>  //获取一些配置参数</p>
    <p> mCriticalBatteryLevel = mContext.getResources().getInteger(</p>
    <p>    com.android.internal.R.integer.config_criticalBatteryWarningLevel);</p>
    <p> </p>
    <p> mLowBatteryWarningLevel = mContext.getResources().getInteger(</p>
    <p>    com.android.internal.R.integer.config_lowBatteryWarningLevel);</p>
    <p> </p>
    <p> mLowBatteryCloseWarningLevel = mContext.getResources().getInteger(</p>
    <p>     com.android.internal.R.integer.config_lowBatteryCloseWarningLevel);</p>
    <p> </p>
    <p>  //启动uevent监听对象，监视power_supply信息</p>
    <p> mPowerSupplyObserver.startObserving("SUBSYSTEM=power_supply");</p>
    <p> </p>
    <p>  //如果下列文件存在，那么启动另一个uevent监听对象。该uevent事件来自invalid charger</p>
    <p>  //switch设备（即不匹配的充电设备）</p>
    <p> if (newFile("/sys/devices/virtual/switch/invalid_charger/state").exists()) {</p>
    <p>     mInvalidChargerObserver.startObserving(</p>
    <p>              "DEVPATH=/devices/virtual/switch/invalid_charger");</p>
    <p>  }</p>
    <p>   update();//①查询HAL层，获取此时的电池信息</p>
    <p>}</p>
</div>
<p>BatteryService定义了3个非常重要的阈值，分别是：</p>
<p>·  mCriticalBatteryLevel表示严重低电，其值为4。当电量低于该值时会强制关机。该值由config.xml中的config_criticalBatteryWarningLevel控制。</p>
<p>·  mLowBatteryWarningLevel表示低电，值为15，当电量低于该值时，系统会报警，例如闪烁LED灯。该值由config.xml中的config_lowBatteryWarningLevel控制。</p>
<p>·  mLowBatteryCloseWarningLevel表示一旦电量大于此值，就脱离低电状态，即可停止警示灯。该值为20，表示由config.xml中的config_lowBatteryCloseWarningLevel控制。</p>
<p>在BatteryService构造函数的最后调用了update函数，该函数将查询系统电池信息，以更新BatteryService内部的成员变量。此函数代码如下：</p>
<p>[--&gt;BatteryService.java::update]</p>
<div>
    <p>private synchronized final void update() {</p>
    <p> native_update();//到Native层查询并更新内部变量的值</p>
    <p> processValues();//处理更新后的状态</p>
    <p>}</p>
</div>
<h4>1.  native_update函数分析</h4>
<p>native_update的实现代码如下：</p>
<p>[--&gt;com_android_server_BatteryService.cpp]</p>
<div>
    <p>static voidandroid_server_BatteryService_update(JNIEnv* env, jobject obj)</p>
    <p>{</p>
    <p>   setBooleanField(env, obj, gPaths.acOnlinePath, gFieldIds.mAcOnline);</p>
    <p>    ......//获取电池信息，并通过JNI设置到Java层对应的变量中</p>
    <p>   setIntField(env, obj, gPaths.batteryTemperaturePath,</p>
    <p>                  gFieldIds.mBatteryTemperature);</p>
    <p>    </p>
    <p>    constint SIZE = 128;</p>
    <p>    charbuf[SIZE];</p>
    <p>    //获取信息，以下参数并不是所有手机都支持的</p>
    <p>    if(readFromFile(gPaths.batteryStatusPath, buf, SIZE) &gt; 0)</p>
    <p>       env-&gt;SetIntField(obj, gFieldIds.mBatteryStatus,getBatteryStatus(buf));</p>
    <p>    else</p>
    <p>       env-&gt;SetIntField(obj, gFieldIds.mBatteryStatus,</p>
    <p>                            gConstants.statusUnknown);</p>
    <p>   ......</p>
    <p>}</p>
</div>
<p>一共有哪些电池信息呢？如表5-4所示。</p>
<p>表5-4  Android系统中的电池信息</p>
<div>
    <table>
        <tbody>
            <tr>
                <td>
                    <p>变量名</p>
                </td>
                <td>
                    <p>功能</p>
                </td>
                <td>
                    <p>备注</p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mAcOnline</p>
                </td>
                <td>
                    <p>是否用外接充电器充电</p>
                </td>
                <td>
                    <p>即用交流电充电</p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mUsbOnline</p>
                </td>
                <td>
                    <p>是否用USB供电</p>
                </td>
                <td>
                    <p>即用USB供电</p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryStatus</p>
                </td>
                <td>
                    <p>电池状态</p>
                </td>
                <td>
                    <p>共有5个状态<sup>，</sup>详细内容可参考com_android_server_BatteryService.cpp中BatteryManagerConstants的定义</p>
                    <p> </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryHealth</p>
                </td>
                <td>
                    <p>电池健康状态</p>
                </td>
                <td>
                    <p>共7个状态<sup>，</sup>详细内容可参考com_android_server_BatteryService.cpp中BatteryManagerConstants的定义</p>
                    <p> </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryPresent</p>
                </td>
                <td>
                    <p>是否使用电池</p>
                </td>
                <td>
                    <p>有些手机在没有电池的情况下可直接利用USB/交流供电</p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryLevel</p>
                </td>
                <td>
                    <p>电池电量</p>
                </td>
                <td>
                    <p> </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryVoltage</p>
                </td>
                <td>
                    <p>电池电压</p>
                </td>
                <td>
                    <p> </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryTemperature</p>
                </td>
                <td>
                    <p>电池温度</p>
                </td>
                <td>
                    <p> </p>
                </td>
            </tr>
            <tr>
                <td>
                    <p>mBatteryTechnology</p>
                </td>
                <td>
                    <p>电池制造技术</p>
                </td>
                <td>
                    <p>一般为“Li-poly”即锂电池技术</p>
                </td>
            </tr>
        </tbody>
    </table>
</div>
<p>mBatteryStatus和mBatteryHealth均有几种不同状态，详细信息可查看getBatteryStatus和getBatteryHealth函数的实现。</p>
<p>上述信息均通过从/sys/class/power_supply目录读取对应文件得到。和以往使用固定路径（可能是Android 2.2版本之前）不同的是，先读取power_supply目录中各个子目录中的type文件，然后根据type文件的内容，再做对应处理：</p>
<p>·  如果type文件的内容为“Mains”：则读取对应子目录中的online文件，可判断是否为AC充电。</p>
<p>·  如果type文件的内容为“Battery”：则从对应子目录中其他的文件中读取电池相关的信息，例如从temp文件获取电池温度，从technology文件读取电池制造技术等。</p>
<p>·  如果type文件的内容为“USB”：读取该子目录中的online文件内容，可判断是否为USB充电。</p>
<div>
    <p><strong>提示</strong> 读者可通过dumpsys battery查看自己手机的电池信息。</p>
</div>
<h4>2.  processValues分析</h4>
<p>获取了电池信息后，BatteryService就要做一些处理，此项工作通过processValues完成，其代码如下：</p>
<p>[--&gt;BatteryService.java::processValues]</p>
<div>
    <p>private void processValues() {</p>
    <p>   longdischargeDuration = 0;</p>
    <p>  mBatteryLevelCritical = mBatteryLevel &lt;= mCriticalBatteryLevel;</p>
    <p>   if (mAcOnline) {</p>
    <p>     mPlugType = BatteryManager.BATTERY_PLUGGED_AC;</p>
    <p>    } elseif (mUsbOnline) {</p>
    <p>     mPlugType = BatteryManager.BATTERY_PLUGGED_USB;</p>
    <p>    } else {</p>
    <p>     mPlugType = BATTERY_PLUGGED_NONE;</p>
    <p>   }</p>
    <p>   //通知BatteryStatsService，该函数以后再分析</p>
    <p>  mBatteryStats.setBatteryState(mBatteryStatus, mBatteryHealth,</p>
    <p>              mPlugType, mBatteryLevel, mBatteryTemperature, mBatteryVoltage</p>
    <p>              );</p>
    <p>   shutdownIfNoPower();//如果电量不够，弹出关机对话框</p>
    <p>  shutdownIfOverTemp();//如果电池过热，弹出关机对话框</p>
    <p>   ......//根据当前电池信息与上次电池信息比较，判断是否需要发送广播等</p>
    <p>   if (比较前后两次电池信息是否发生变化) {</p>
    <p>     ......//记录信息到日志文件</p>
    <p>     Intent statusIntent = new Intent();</p>
    <p>     statusIntent.setFlags(</p>
    <p>             Intent.FLAG_RECEIVER_REGISTERED_ONLY_BEFORE_BOOT);</p>
    <p>     if (mPlugType != 0 &amp;&amp; mLastPlugType ==0) {</p>
    <p>       statusIntent.setAction(Intent.ACTION_POWER_CONNECTED);</p>
    <p>               mContext.sendBroadcast(statusIntent); </p>
    <p>      }......</p>
    <p>    if(sendBatteryLow) {</p>
    <p>        mSentLowBatteryBroadcast = true;//发送低电提醒</p>
    <p>       statusIntent.setAction(Intent.ACTION_BATTERY_LOW);</p>
    <p>       mContext.sendBroadcast(statusIntent);</p>
    <p>     } ......</p>
    <p>     mLed.updateLightsLocked();//更新LED灯状态</p>
    <p>    mLastBatteryStatus= mBatteryStatus;//保存新的电池信息</p>
    <p>    ......</p>
    <p>}</p>
</div>
<p>processValues函数非常简单，此处不再详述。另外，当电池信息发生改变时，系统会发送uevent事件给BatteryService，此时BatteryService只要重新调用update即可完成工作。</p>
<h3>5.5.2 BatteryStatsService分析</h3>
<p>BatteryStatsService（为书写方便，以后简称BSS）主要功能是收集系统中各模块和应用进程用电量情况。抽象地说，BSS就是一块电表，不过这块电表不只是显示总的耗电量，而是分门别类地显示耗电量，力图做到更为精准。</p>
<p>和其他服务不太一样的是，BSS的创建和注册是在ActivityManagerService中进行的，相关代码如下：</p>
<p>[--&gt;ActivityManagerService.java::ActivityManagerService构造函数]</p>
<div>
    <p>private ActivityManagerService() {</p>
    <p>     ......//创建BSS对象，传递一个File对象，指向/data/system/batterystats.bin</p>
    <p>     mBatteryStatsService= new BatteryStatsService(new File(</p>
    <p>               systemDir, "batterystats.bin").toString());</p>
    <p>}</p>
</div>
<p>[--&gt;ActivityManagerService.java::main]</p>
<div>
    <p>//调用BSS的publish函数，在内部将其注册到ServiceManager</p>
    <p>m.mBatteryStatsService.publish(context);</p>
</div>
<p>下面来分析BSS的构造函数，见识一下这块电表的样子。</p>
<h4>1. BatteryStatsService介绍</h4>
<p>让人大跌眼镜的是，BSS其实只是一个壳，具体功能委托BatteryStatsImpl（以后简称BSImpl）来实现，代码如下：</p>
<p>[--&gt;BatteryStatsService.java::BatteryStatsService构造函数]</p>
<div>
    <p>BatteryStatsService(String filename) {</p>
    <p>    mStats = new BatteryStatsImpl(filename);</p>
    <p>}</p>
</div>
<p>图5-2展示了BSS及BSImpl的家族图谱。</p>

![image](images/chapter5/image002.png)

<p>图5-2  BSS及BSImpl家族图谱</p>
<p>由图5-2可知：</p>
<p>·  BSS通过成员变量mStats指向一个BSImpl类型的对象。</p>
<p>·  BSImpl从BatteryStats类派生。更重要的是，该类实现了Parcelable接口，由此可知，BSImpl对象的信息可以写到Parcel包中，从而可通过Binder在进程间传递。实际上，在Android手机的设置中查到的用电信息就是来自BSImpl的。</p>
<p>BSS的getStatistics函数提供了查询系统用电信息的接口，代码如下：</p>
<div>
    <p>public byte[] getStatistics() {</p>
    <p>   mContext.enforceCallingPermission(//检查调用进程是否有BATTERY_STATS权限</p>
    <p>        android.Manifest.permission.BATTERY_STATS, null);</p>
    <p>  Parcel out= Parcel.obtain();</p>
    <p> mStats.writeToParcel(out, 0);//将BSImpl信息写到数据包中</p>
    <p>  byte[]data = out.marshall();//序列化为一个buffer，然后通过Binder传递</p>
    <p> out.recycle();</p>
    <p>  returndata;</p>
    <p> }</p>
</div>
<p>由此可以看出，电量统计的核心类是BSImpl，下面就来分析它。</p>
<h4>2.  初识BSImpl </h4>
<p>BSImpl功能是进行电量统计，那么是否存在计量工具呢？答案是肯定的，并且BSImpl使用了不止一种的计量工具。</p>
<h5>（1） 计量工具和统计对象介绍</h5>
<p>BSImpl一共使用了4种计量工具，如图5-3所示。</p>

![image](images/chapter5/image003.png)

<p>图5-3  计量工具图例</p>
<p>由图5-3可知：</p>
<p>·  一共有两大类计量工具，Counter用于计数，Timer用于计时。</p>
<p>·  BSImpl实现了StopwatchTimer（即所谓的秒表）、SamplingTimer（抽样计时）、Counter和SamplingCounter（抽样计数）等4个具体的计量工具。</p>
<p>·  BSImpl中定义了一个Unpluggable接口。当手机插上USB线充电（不论是由AC还是由USB供电）时，该接口的plug函数被调用。反之，当拔去USB线时，该接口的unplug函数被调用。设置这个接口的目的是为了满足BSImpl对各种情况下系统用电量的统计要求。关于Unpluggable接口的作用，在后续内容中可以能见到。</p>
<p>虽然只有4种计量工具（笔者觉得已经相当多了），但是可以在很多地方使用它们。下面先来认识部分被挂牌要求统计用电量的对象，如表5-5所示。</p>
<p>表5-5  用电量统计项</p>
<table>
    <tbody>
        <tr>
            <td>
                <p>成员变量名</p>
            </td>
            <td>
                <p>类型</p>
            </td>
            <td>
                <p>备注</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mScreenOnTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>统计屏幕开启耗电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mScreenBrightnessTimer[]</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>统计各级屏幕亮度（共5级）情况下的耗电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mInputEventCounter</p>
            </td>
            <td>
                <p>Counter</p>
            </td>
            <td>
                <p>统计输入事件耗电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mPhoneOnTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>统计通话耗电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mPhoneSignalStrengthsTimer[]</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>统计手机信号各级强度耗电量，共5级</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mPhoneSignalScanningTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>统计搜索手机信号耗电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mPhoneDataConnectionsTimer[]</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>统计手机使用各种数据通信方式（如GPRS、CDMA等）的用电量，一共15级</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mWifiOnTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>Wifi用电量（包括使用网络和开启Wifi功能却没有使用网络的情况）</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mGlobalWifiRunningTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>使用Wifi的用电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mAudioOnTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>使用Audio的耗电量</p>
            </td>
        </tr>
        <tr>
            <td>
                <p>mVideoOnTimer</p>
            </td>
            <td>
                <p>StopwatchTimer</p>
            </td>
            <td>
                <p>使用Video的耗电量</p>
            </td>
        </tr>
    </tbody>
</table>
<p>表5-5中的电量统计项已经够多了吧？还不止这些，为了做到更精确，Android还希望能统计每个进程在各种情况下的耗电量。这是一项庞大的工程，怎么做到的呢？来看下一节的内容。</p>
<h5>（2） BatteryStats.Uid介绍</h5>
<p>在Android 4.0中，和进程相关的用电量统计并非以单个PID为划分单元，而是以Uid为组，相关类结构如图5-4所示。</p>

![image](images/chapter5/image004.png)

<p>图5-4  BatteryStats.Uid家族</p>
<p>由图5-4可知：</p>
<p>·  Wakelock用于统计该Uid对应进程使用wakeLock的情况。</p>
<p>·  Proc用于统计Uid中某个进程的电量使用情况。</p>
<p>·  Pkg用于统计某个特定Package的使用情况，其内部类Serv用于统计该Pkg中Service的用电情况。</p>
<p>·  Sensor用于统计传感器用电情况。</p>
<p>基于以上的了解，以后分析将会轻松很多，下面来分析它的代码。</p>
<h4>3.  BSImpl流程分析</h4>
<h5>（1） 构造函数分析</h5>
<p>先分析构造函数，代码如下：</p>
<p>[--&gt;BatteryStatsImpl.java::BatteryStatsImpl构造函数]</p>
<div>
    <p>public BatteryStatsImpl(String filename) {</p>
    <p>   //JournaledFile为日志文件对象，内部包含两个文件，原始文件和临时文件。目的是双备份，</p>
    <p>   //以防止在读写过程中文件信息丢失或出错</p>
    <p>   mFile =new JournaledFile(new File(filename), new File(filename + ".tmp"));</p>
    <p>   mHandler= new MyHandler();//创建一个Handler对象</p>
    <p>  mStartCount++;</p>
    <p>   //创建表5-5中的用电统计项对象</p>
    <p>  mScreenOnTimer = new StopwatchTimer(null, -1, null, mUnpluggables);</p>
    <p>   for (inti=0; i&lt;NUM_SCREEN_BRIGHTNESS_BINS; i++) {</p>
    <p>      mScreenBrightnessTimer[i] = new StopwatchTimer(null, -100-i, null,</p>
    <p>               mUnpluggables);</p>
    <p>   }</p>
    <p>  mInputEventCounter = new Counter(mUnpluggables);</p>
    <p>   ......</p>
    <p>   mOnBattery= mOnBatteryInternal = false;//设置这两位成员变量为false</p>
    <p>  initTimes();//①初始化统计时间</p>
    <p>  mTrackBatteryPastUptime = 0;</p>
    <p>  mTrackBatteryPastRealtime = 0;</p>
    <p>   mUptimeStart= mTrackBatteryUptimeStart =</p>
    <p>                             SystemClock.uptimeMillis()* 1000;</p>
    <p>   mRealtimeStart= mTrackBatteryRealtimeStart =</p>
    <p>                                SystemClock.elapsedRealtime()* 1000;</p>
    <p>  mUnpluggedBatteryUptime = getBatteryUptimeLocked(mUptimeStart);</p>
    <p>  mUnpluggedBatteryRealtime = getBatteryRealtimeLocked(mRealtimeStart);</p>
    <p>  mDischargeStartLevel = 0;</p>
    <p>  mDischargeUnplugLevel = 0;</p>
    <p>  mDischargeCurrentLevel = 0;</p>
    <p>  initDischarge();     //②初始化和电池level有关的成员变量</p>
    <p>  clearHistoryLocked();//③删除用电统计的历史记录</p>
    <p> }</p>
</div>
<p>要看懂这段代码比较困难，主要原因是变量太多，并且没有注释说明。只能根据名字来推测了。在以上代码中除了计量工具外，还出现了三大类变量：</p>
<p>·  用于统计时间的变量，例如mUptimeStart、mTrackBatteryPastUptime等。这些参数的初始化函数为initTimes。注意，系统时间分为uptime和realtime。uptime和realtime的时间起点都从系统启动开始算（since the system was booted），但是uptime不包括系统休眠时间，而realtime包括系统休眠时间<a>[③]</a>。</p>
<p>·  用于记录各种情况下电池电量的变量，如mDischargeStartLevel、mDischargeCurrentLevel等，这些成员变量的初始化函数为initDischarge。</p>
<p>·  用于保存历史记录的HistroryItem，在clearHistoryLocked函数中初始化，主要有mHistory、mHistoryEnd等成员变量（这些成员在clearHistoryLocked函数中出现）。</p>
<p>上述这些成员变量的具体作用，只有通过后文的分析才能弄清楚。这里先介绍StopwacherTimer。</p>
<div>
    <p>//调用方式</p>
    <p>mPhoneSignalScanningTimer = newStopwatchTimer(null, -200+1,</p>
    <p>                                   null,mUnpluggables);</p>
    <p>//mUnpluggables类型为ArrayList&lt;Unpluggable&gt;，用于保存插拔USB线时需要对应更新用电</p>
    <p>//信息的统计对象</p>
    <p>// StopwatchTimer的构造函数</p>
    <p>StopwatchTimer(Uid uid, int type,ArrayList&lt;StopwatchTimer&gt; timerPool,</p>
    <p>                 ArrayList&lt;Unpluggable&gt;unpluggables) {</p>
    <p>   //在本例中，uid为0，type为负数，timerPool为空，unpluggables为mUnpluggables</p>
    <p>  super(type, unpluggables); </p>
    <p>   mUid =uid;</p>
    <p>  mTimerPool = timerPool;</p>
    <p>}</p>
    <p>// Timer的构造函数</p>
    <p>Timer(int type, ArrayList&lt;Unpluggable&gt;unpluggables) {</p>
    <p>     mType =type;</p>
    <p>    mUnpluggables = unpluggables;</p>
    <p>    unpluggables.add(this);</p>
    <p>}</p>
</div>
<p>在StopwatchTimer中比较难理解的就是unpluggables，根据注释说明，当拔插USB线时，需要更新用电统计的对象，应该将其加入到mUnpluggables数组中。</p>
<p>在启动秒表时，调用它的startRunningLocked函数，并传入BSImpl实例，代码如下：</p>
<div>
    <p>void startRunningLocked(BatteryStatsImpl stats) {</p>
    <p>  if(mNesting++ == 0) {//嵌套调用控制</p>
    <p>        // getBatteryRealtimeLocked函数返回总的电池使用时间</p>
    <p>       mUpdateTime = stats.getBatteryRealtimeLocked(</p>
    <p>                            SystemClock.elapsedRealtime()* 1000);</p>
    <p>         if (mTimerPool != null) {//不讨论这种情况</p>
    <p>         }</p>
    <p>        mCount++;</p>
    <p>        mAcquireTime = mTotalTime;//计数控制，请读者阅读相关注释说明</p>
    <p>       }</p>
    <p>   }</p>
</div>
<p>当停用秒表时，调用它的stopRunningLocked函数，代码如下：</p>
<div>
    <p>void stopRunningLocked(BatteryStatsImpl stats) {</p>
    <p>  if (mNesting == 0) {</p>
    <p>     return; //嵌套控制</p>
    <p>  }</p>
    <p>  if(--mNesting == 0) {</p>
    <p>       if(mTimerPool != null) {//不讨论这种情况</p>
    <p>        }else {</p>
    <p>        final long realtime = SystemClock.elapsedRealtime() * 1000;</p>
    <p>         //计算此次启动/停止周期的时间</p>
    <p>        final long batteryRealtime = stats.getBatteryRealtimeLocked(realtime);</p>
    <p>          mNesting = 1;</p>
    <p>          //mTotalTime代表从启动开始该秒停表一共记录的时间</p>
    <p>         mTotalTime = computeRunTimeLocked(batteryRealtime);</p>
    <p>         mNesting = 0;</p>
    <p>          }</p>
    <p>       if (mTotalTime == mAcquireTime)  mCount--;</p>
    <p>     }</p>
    <p> }</p>
</div>
<p>在StopwatchTimer中定义了很多的时间参数，无非就是用于记录各种时间，例如总耗时、最近一次工作周期的耗时等。如果不是工作需要（例如研究Settings应用中和BatteryInfo相关的内容），读者仅需了解它的作用即可。</p>
<h5>（2） ActivityManagerService和BSS交互</h5>
<p>ActivityManagerService创建BSS后，还要进行几项操作，具体代码分别如下：</p>
<p>[--&gt;ActivityManagerService.java::ActivityManagerService构造函数]</p>
<div>
    <p>mBatteryStatsService = new BatteryStatsService(newFile(</p>
    <p>               systemDir, "batterystats.bin").toString());</p>
    <p>  //操作通过BSImpl创建的JournaledFile文件</p>
    <p> mBatteryStatsService.getActiveStatistics().readLocked();</p>
    <p> mBatteryStatsService.getActiveStatistics().writeAsyncLocked();</p>
    <p>  //BSImpl的getIsOnBattery返回mOnBattery变量，初始化值为false</p>
    <p>  mOnBattery= DEBUG_POWER ? true</p>
    <p>           : mBatteryStatsService.getActiveStatistics().getIsOnBattery();</p>
    <p>   //设置回调，该回调也是用于信息统计，只能留到介绍ActivityManagerService时再来分析了</p>
    <p>mBatteryStatsService.getActiveStatistics().setCallback(this);</p>
</div>
<p>[--&gt;ActivityManagerService.java::main函数]</p>
<div>
    <p>m.mBatteryStatsService.publish(context);</p>
</div>
<p>[--&gt;BatteryStatsService.java::publish]</p>
<div>
    <p>public void publish(Context context) {</p>
    <p>  mContext =context;</p>
    <p>  //注意，BSS服务叫做batteryinfo，而BatteryService服务叫做battery</p>
    <p> ServiceManager.addService("batteryinfo", asBinder());</p>
    <p>  //PowerProfile见下文解释</p>
    <p> mStats.setNumSpeedSteps(new PowerProfile(mContext).getNumSpeedSteps());</p>
    <p>  //设置通信信号扫描超时时间</p>
    <p> mStats.setRadioScanningTimeout(mContext.getResources().getInteger(</p>
    <p>             com.android.internal.R.integer.config_radioScanningTimeout)</p>
    <p>              * 1000L);</p>
    <p>    }</p>
</div>
<p>在以上代码中，比较有意思的是PowerProfile类，它将解析Android 4.0源码/frameworks/base/core/res/res/xml/power_profile.xml文件。此XML文件存储的是各种操作（和硬件相关）的耗电情况，如图5-5所示。</p>

![image](images/chapter5/image005.png)

<p>图5-5  PowerProfile文件示例</p>
<p>由图5-5可知，该文件保存了各种操作的耗电情况，以mAh（毫安）为单位。PowerProfile的getNumSpeedSteps将返回CPU支持的频率值，目前在该XML中只定义了一个值，即400MHz。</p>
<div>
    <p><strong>注意</strong>在编译时，各厂家会将特定硬件平台的power_profile.xml复制到输出目录。此处展示的power_profile.xml和硬件平台无关。</p>
</div>
<h5>（3） BatteryService和BSS交互</h5>
<p>BatteryService在它的processValues函数中和BSS交互，代码如下：</p>
<p>[--&gt;BatteryService.java]</p>
<div>
    <p>private void processValues() {</p>
    <p>   ......</p>
    <p>   mBatteryStats.setBatteryState(mBatteryStatus,mBatteryHealth, mPlugType,</p>
    <p>                 mBatteryLevel, mBatteryTemperature,mBatteryVoltage);</p>
    <p>}</p>
</div>
<p>BSS的工作由BSImpl来完成，所以直接setBatteryState函数的代码：</p>
<p>[--&gt;BatteryStatsImpl.java::setBatteryState]</p>
<div>
    <p>public void setBatteryState(int status, inthealth, int plugType, int level,</p>
    <p>                                int temp, int volt) {</p>
    <p>  synchronized(this) {</p>
    <p>      boolean onBattery = plugType == BATTERY_PLUGGED_NONE;//判断是否为电池供电</p>
    <p>       intoldStatus = mHistoryCur.batteryStatus;</p>
    <p>       ......</p>
    <p>        if(onBattery) {</p>
    <p>            //mDischargeCurrentLevel记录当前使用电池供电时的电池电量</p>
    <p>            mDischargeCurrentLevel = level;</p>
    <p>            mRecordingHistory = true;//mRecordingHistory表示需要记录一次历史值</p>
    <p>         }</p>
    <p>       //此时,onBattery为当前状态，mOnBattery为历史状态</p>
    <p>      if(onBattery != mOnBattery) { </p>
    <p>          mHistoryCur.batteryLevel = (byte)level;</p>
    <p>          mHistoryCur.batteryStatus = (byte)status;</p>
    <p>           mHistoryCur.batteryHealth = (byte)health;</p>
    <p>           ......//更新mHistoryCur中的电池信息</p>
    <p>               setOnBatteryLocked(onBattery, oldStatus, level);</p>
    <p>           } else {</p>
    <p>               boolean changed = false;</p>
    <p>               if (mHistoryCur.batteryLevel != level) {</p>
    <p>                   mHistoryCur.batteryLevel = (byte)level;</p>
    <p>                   changed = true;</p>
    <p>               }</p>
    <p>               ......//判断电池信息是否发生变化</p>
    <p>               if (changed) {//如果发生变化，则需要增加一次历史记录</p>
    <p>                   addHistoryRecordLocked(SystemClock.elapsedRealtime());</p>
    <p>               }</p>
    <p>           }</p>
    <p>           if (!onBattery &amp;&amp; status == BatteryManager.BATTERY_STATUS_FULL){</p>
    <p>               mRecordingHistory = false;</p>
    <p>           }</p>
    <p>        }</p>
    <p>    }</p>
</div>
<p>setBatteryState函数的工作主要有两项：</p>
<p>·  判断当前供电状态是否发生变化，由onBattery和mOnBattery进行比较。其中onBattery用于判断当前是否为电池供电，mOnBattery为上次调用该函数时得到的判断值。如果供电状态发生变化（其实就是经历一次USB拔插过程），则调用setOnBatteryLocked函数。</p>
<p>·  如果供电状态未发生变化，则需要判断电池信息是否发生变化，例如电量和电压等。如果发生变化，则调用addHistoryRecordLocked。该函数用于记录一次历史信息。</p>
<p>接下来看setOnBatteryLocked函数的代码：</p>
<p>[--&gt;BatteryStatsImpl.java::setOnBatteryLocked]</p>
<div>
    <p>void setOnBatteryLocked(boolean onBattery, intoldStatus, int level) {</p>
    <p>   boolean doWrite = false;</p>
    <p>   //发送一个消息给mHandler，将在内部调用ActivityManagerService设置的回调函数</p>
    <p>   Message m= mHandler.obtainMessage(MSG_REPORT_POWER_CHANGE);</p>
    <p>   m.arg1 =onBattery ? 1 : 0;</p>
    <p>  mHandler.sendMessage(m);</p>
    <p>  mOnBattery = mOnBatteryInternal = onBattery;</p>
    <p>   longuptime = SystemClock.uptimeMillis() * 1000;</p>
    <p>   longmSecRealtime = SystemClock.elapsedRealtime();</p>
    <p>   longrealtime = mSecRealtime * 1000;</p>
    <p>   if(onBattery) {</p>
    <p>       //关于电量信息统计，有一个值得注意的地方：当oldStatus为满电状态，或当前电量</p>
    <p>      //大于90，或mDischargeCurrentLevel小于20并且当前电量大于80时，要清空统计</p>
    <p>      //信息，以开始新的统计。也就是说在满足特定条件的情况下，电量使用统计信息会清零并重</p>
    <p>     //新开始。读者不妨用自己手机一试</p>
    <p>       if(oldStatus == BatteryManager.BATTERY_STATUS_FULL || level &gt;= 90</p>
    <p>           || (mDischargeCurrentLevel &lt; 20 &amp;&amp; level &gt;= 80)) {</p>
    <p>           doWrite = true;</p>
    <p>           resetAllStatsLocked();</p>
    <p>           mDischargeStartLevel = level;</p>
    <p>       }</p>
    <p>        //读取/proc/wakelock文件，该文件反映了系统wakelock的使用状态，</p>
    <p>        //感兴趣的读者可自行研究</p>
    <p>       updateKernelWakelocksLocked();</p>
    <p> </p>
    <p>       mHistoryCur.batteryLevel = (byte)level;</p>
    <p>       mHistoryCur.states &amp;= ~HistoryItem.STATE_BATTERY_PLUGGED_FLAG;</p>
    <p>        //添加一条历史记录</p>
    <p>        addHistoryRecordLocked(mSecRealtime);</p>
    <p>        //mTrackBatteryUptimeStart表示使用电池的开始时间，由uptime表示</p>
    <p>       mTrackBatteryUptimeStart = uptime;</p>
    <p>        // mTrackBatteryRealtimeStart表示使用电池的开始时间，由realtime表示</p>
    <p>       mTrackBatteryRealtimeStart = realtime;</p>
    <p>        //mUnpluggedBatteryUptime记录总的电池使用时间（不论中间插拔多少次）</p>
    <p>       mUnpluggedBatteryUptime = getBatteryUptimeLocked(uptime);</p>
    <p>        // mUnpluggedBatteryRealtime记录总的电池使用时间</p>
    <p>       mUnpluggedBatteryRealtime = getBatteryRealtimeLocked(realtime);</p>
    <p>        //记录电量</p>
    <p>        mDischargeCurrentLevel =mDischargeUnplugLevel = level;</p>
    <p>        if(mScreenOn) {</p>
    <p>            mDischargeScreenOnUnplugLevel = level;</p>
    <p>            mDischargeScreenOffUnplugLevel = 0;</p>
    <p>          }else {</p>
    <p>              mDischargeScreenOnUnplugLevel = 0;</p>
    <p>             mDischargeScreenOffUnplugLevel = level;</p>
    <p>          }</p>
    <p>         mDischargeAmountScreenOn = 0;</p>
    <p>         mDischargeAmountScreenOff = 0;</p>
    <p>          //调用doUnplugLocked函数</p>
    <p>         doUnplugLocked(mUnpluggedBatteryUptime, mUnpluggedBatteryRealtime);</p>
    <p>        }else {</p>
    <p>            ......//处理使用USB充电的情况，请读者在上面讨论的基础上自行分析</p>
    <p>        }</p>
    <p>      ......//记录信息到文件</p>
    <p>      }</p>
    <p>    }</p>
</div>
<p>doUnplugLocked函数将更新对应信息，该函数比较简单，无须赘述。另外，addHistoryRecordLocked函数用于增加一条历史记录（由HistoryItem表示），读者也可自行研究。</p>
<p>从本节的分析可知，Android将电量统计分得非常细，例如由电池供电的情况需要统计，由USB/AC充电的情况也要统计，因此有setBatteryState函数的存在。</p>
<h5>（4） PowerManagerService和BSS交互</h5>
<p>PMS和BSS交互是最多的，此处以noteScreenOn和noteUserActivity为例，来介绍BSS到底是如何统计电量的。</p>
<p>先来看noteScreenOn函数。当开启屏幕时，PMS会调用BSS的noteScreenOn以通知屏幕开启，该函数在内部调用BSImpl的noteScreenOnLocked，其代码如下：</p>
<p>[--&gt;BatteryStatsImpl.java::noteScreenOnLocked]</p>
<div>
    <p>public void noteScreenOnLocked() {</p>
    <p>   if(!mScreenOn) {</p>
    <p>     mHistoryCur.states |= HistoryItem.STATE_SCREEN_ON_FLAG;</p>
    <p>      //增加一条历史记录</p>
    <p>      addHistoryRecordLocked(SystemClock.elapsedRealtime());</p>
    <p>     mScreenOn = true;</p>
    <p>     //启动mScreenOnTime秒停表，内部就是记录时间，读者可自行研究</p>
    <p>     mScreenOnTimer.startRunningLocked(this);</p>
    <p>      if(mScreenBrightnessBin &gt;= 0)//启动对应屏幕亮度的秒停表（参考表5-5）</p>
    <p>       mScreenBrightnessTimer[mScreenBrightnessBin].startRunningLocked(this);</p>
    <p>      //屏幕开启也和内核WakeLock有关，所以这里一样要更新WakeLock的用电统计</p>
    <p>     noteStartWakeLocked(-1, -1, "dummy", WAKE_TYPE_PARTIAL);</p>
    <p>      if(mOnBatteryInternal)</p>
    <p>        updateDischargeScreenLevelsLocked(false, true);</p>
    <p>     }</p>
    <p> }</p>
</div>
<p>再来看noteUserActivity，当有输入事件触发PMS的userActivity时，该函数被调用,代码如下,：</p>
<p>[--&gt;BatteryStatsImpl.java::noteUserActivityLocked]</p>
<div>
    <p>//BSS的noteUserActivity将调用BSImpl的noteUserActivityLocked</p>
    <p>public void noteUserActivityLocked(int uid, intevent) {</p>
    <p>        getUidStatsLocked(uid).noteUserActivityLocked(event);</p>
    <p>}</p>
</div>
<p>先是调用getUidStatsLocked以获取一个Uid对象，如果该Uid是首次出现的，则要在内部创建一个Uid对象。直接来了解Uid的noteUserActivityLocked函数：</p>
<div>
    <p>public void noteUserActivityLocked(int type) {</p>
    <p> if(mUserActivityCounters == null) {</p>
    <p>    initUserActivityLocked();</p>
    <p>   }</p>
    <p>  if (type&lt; 0) type = 0;</p>
    <p>  else if(type &gt;= NUM_USER_ACTIVITY_TYPES) </p>
    <p>     type= NUM_USER_ACTIVITY_TYPES-1;</p>
    <p>   // noteUserActivityLocked只是调用对应type的Counter的stepAtomic函数</p>
    <p>   //每个Counter内部都有个计数器，stepAtomic使该计数器增1</p>
    <p>  mUserActivityCounters[type].stepAtomic();</p>
    <p>}</p>
</div>
<p>mUserActivityCounters为一个7元Counter数组，该数组对应7种不同的输入事件类型，在代码中，由BSImpl的成员变量USER_ACTIVITY_TYPES表示，如下所示：</p>
<div>
    <p>static final String[] USER_ACTIVITY_TYPES = {</p>
    <p> "other", "cheek", "touch","long_touch", "touch_up", "button", "unknown"</p>
    <p>};</p>
</div>
<p>另外，在LocalPowerManager中，也定义了相关的type值，如下所示：</p>
<p>[--&gt;LocalPowerManager.java]</p>
<div>
    <p>public interface LocalPowerManager {</p>
    <p>    publicstatic final int OTHER_EVENT = 0;</p>
    <p>    publicstatic final int BUTTON_EVENT = 1;</p>
    <p>    publicstatic final int TOUCH_EVENT = 2; //目前只使用这三种事件</p>
    <p>    ......</p>
    <p>}</p>
</div>
<h3>5.5.3 BatteryService及BatteryStatsService总结</h3>
<p>本节重点讨论了BatteryService和BatteryStatsService。其中，BatteryService和系统中的供电系统交互，通过它可获取电池状态等信息。而BatteryStatsService用于统计系统用电量的情况。就难度而言，BSS较为复杂，原因是Android试图对系统耗电量作非常细致的统计，导致统计项非常繁杂。另外，电量统计大多采用被动通知的方式（即需要其他服务主动调用BSS提供的noteXXXOn/noteXXXOff函数），这种实现方法一方面加重了其他服务的负担，另一方面影响了这些服务未来的功能扩展。</p>
<div>
    <p><strong>注意</strong>虽然Google费尽心血来完善电量统计，但这并不是解决耗电量大的根本途径。另外，读者可分析Settings程序中电量统计图的绘制以加深对各种统计对象的理解。Settings中和电量相关的文件在Android 4.0源码的/packages/apps/Settings/src/com/android/settings/fuelgauge/目录中。</p>
</div>
<h2>5.6  本章学习指导</h2>
<p>本章的难度其实在BSS中，而PMS和BatteryService相对较简单。在这三项服务中， PMS是核心。读者在研究PMS时，要注意把握以下几个方面：</p>
<p>·  PMS的初期工作流程，即构造函数、init函数、systemReady函数和BootCompleted函数等。</p>
<p>·  PMS功能在于根据当前系统状态（包括mUserState和mWakeLockState）去操作屏幕和灯光。而触发状态改变的有WakeLock的获取和释放，userActivity函数的调用，因此读者也要搞清楚PMS在这两个方面的工作原理。</p>
<p>·  PMS还有一部分功能和传感器有关，其功能无非还是根据状态操作屏幕和灯光。除非工作需要，否则只需要简单了解这部分的工作流程即可。</p>
<p>对BSS来说，复杂之处在于它定义了很多成员变量和数据类型，并且没有一份电量统计标准的说明文档，因此笔者认为，读者只要搞清楚那几个计量工具和各个统计项的作用即可，如果在其他服务的代码中看到和BSS交互的函数，那么只需知道原因和目的即可。</p>
<p>另外，电源管理需要HAL层和Linux内核提供支持，感兴趣的读者不妨以本章知识为切入点，对底层技术进行一番深入剖析。</p>
<h2>5.7  本章小结</h2>
<p>电源管理系统的核心是PowerManagerService，还包括BatteryService和BatteryStatsService。本章对Android平台中的电源管理系统进行了较详细的分析，其中：</p>
<p>·  对于PMS，本章分析了它的初始化流程、WakeLock获取流程、userActivity函数的工作流程及Power按键处理流程。</p>
<p>·  BatteryService功能较为简单，读者大概了解即可。</p>
<p>·  对于BatteryStatsService，本章对它内部的数据结构、统计对象等进行了较详细的介绍，并对其工作流程展开了分析。建议读者结合Settings应用中的相关代码，加深对其中各种计量工具及统计对象的理解。</p>
<p> </p>
<div>
    <br />
    <hr />
    <div>
        <p><a>[①]</a> config.xml文件的全路径是4.0源码/frameworks/base/core/res/res/values/config.xml。</p>
    </div>
    <div>
        <p><a>[②]</a>必须在一定时间内完成按下和松开Power键的操作，否则系统会认为是关机操作。详情将在卷Ⅲ<span>输入系统一章</span>的分析。</p>
    </div>
    <div>
        <p><a>[③]</a>读者可阅读SDK文档中关于SystemClock类的说明。</p>
    </div>
</div>
<div>
    <p>版权声明：本文为博主原创文章，未经博主允许不得转载。</p>
</div>
