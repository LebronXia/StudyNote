- **包扫描的过程**：经过这个过程，Android就能将一个APK文件的静态信息转化为可以管理的数据结构
- **包查询的过程**：Intent的定义和解析是包查询的核心，通过包查询服务可以获取到一个包的信息
- **包安装的过程**：这个过程是包管理者接纳一个新入成员的体现

在涉及到一个包的安装过程的时候，往往都绕不开`PackageManagerService`，它是Android系统中最常用的服务之一。主要负责系统中Package的管理，应用程序的安装、卸载、信息查询等工作。

开始前想先了解下什么是PMS(**Android 11**)：

## 一、 PMS的启动过程

PMS也是在`systemServer`中创建的，涉及到PMS的创建部分：

```java
//SystemServer

private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        ......
  
          //创建PMS对象
 try {
            Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
            mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        } finally {
            Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
        }
  
    	//PMS是不是首次启动	
  mFirstBoot = mPackageManagerService.isFirstBoot();
  mPackageManager = mSystemContext.getPackageManager();
}
  
  private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
    ......
     //完成dex优化
  mPackageManagerService.updatePackagesIfNeeded();
  //完成磁盘维护
  mPackageManagerService.performFstrimIfNeeded();
  //通知系统进入就绪状态
  mPackageManagerService.systemReady();
  }

```

在这个启动过程中，PMS的`main函数`是整个的核心，现在我们就继续进行分析：

```java
public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
  
  //对PMS对象
  PackageManagerService m = new PackageManagerService(injector, onlyCore, factoryTest);
        t.traceEnd(); // "create package manager"
  
  //将 package 服务注册到 ServiceManager，这是 binder 服务的常规注册流程
    ServiceManager.addService("package", m);
  final PackageManagerNative pmn = m.new PackageManagerNative();
        ServiceManager.addService("package_native", pmn);
    return m;
  
}
```

main方法的代码不多，主要是调用到了PMS的构造函数，而由于PMS中做了很多工作，这也就是Android启动速度慢的主要原因之一。

在其构造方法中，主要的工作是扫描Android系统中几个目标文件夹中的apk，之后建立合适的数据结构和管理如Package信息，四大组件信息，权限信息等各种信息。PKMS的工作流程可以分为五个阶段来分析：

```java
 public PackageManagerService(Injector injector, boolean onlyCore, boolean factoryTest) {
   ......
   EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());
   ......
   EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);
   ......
   EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                        SystemClock.uptimeMillis());
   ......
   EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());
   ......
   EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());
 }
```

可以看到系统根据EventLog将PKMS的初始化分为以下几个阶段：

- `BOOT_PROGRESS_PMS_START` 启动阶段
- `BOOT_PROGRESS_PMS_SYSTEM_SCAN_START`启动阶段
- `BOOT_PROGRESS_PMS_DATA_SCAN_START`启动阶段
- `BOOT_PROGRESS_PMS_SCAN_END`启动阶段
- `BOOT_PROGRESS_PMS_READY`启动阶段

### 1.1 BOOT_PROGRESS_PMS_START 阶段

```java
PackageManager.disableApplicationInfoCache();
        PackageManager.disablePackageInfoCache();

        PackageManager.corkPackageInfoCache();

        final TimingsTraceAndSlog t = new TimingsTraceAndSlog(TAG + "Timing",
                Trace.TRACE_TAG_PACKAGE_MANAGER);
        mPendingBroadcasts = new PendingPackageBroadcasts();

        mInjector = injector;
        mInjector.bootstrap(this);
        mLock = injector.getLock();
        mInstallLock = injector.getInstallLock();
        LockGuard.installLock(mLock, LockGuard.INDEX_PACKAGES);
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());

        if (mSdkVersion <= 0) {
            Slog.w(TAG, "**** ro.build.version.sdk not set!");
        }

        mContext = injector.getContext();
        mFactoryTest = factoryTest;  //开机模式
        mOnlyCore = onlyCore;   //是否对包做dex优化
        mMetrics = new DisplayMetrics();  //分辨率的配置
        mInstaller = injector.getInstaller(); //保存installer对象

        // Create sub-components that provide services / data. Order here is important.
        t.traceBegin("createSubComponents");

        // Expose private service for system components to use.
        mPmInternal = new PackageManagerInternalImpl();
        LocalServices.addService(PackageManagerInternal.class, mPmInternal);
        mUserManager = injector.getUserManagerService();
        mComponentResolver = injector.getComponentResolver();
        mPermissionManager = injector.getPermissionManagerServiceInternal();
//创建setting对象
        mSettings = injector.getSettings();
			//权限管理服务
        mPermissionManagerService = (IPermissionManager) ServiceManager.getService("permissionmgr");
        mIncrementalManager =
                (IncrementalManager) mContext.getSystemService(Context.INCREMENTAL_SERVICE);
        PlatformCompat platformCompat = mInjector.getCompatibility();
        mPackageParserCallback = new PackageParser2.Callback() {
            @Override
            public boolean isChangeEnabled(long changeId, @NonNull ApplicationInfo appInfo) {
                return platformCompat.isChangeEnabled(changeId, appInfo);
            }

            @Override
            public boolean hasFeature(String feature) {
                return PackageManagerService.this.hasSystemFeature(feature, 0);
            }
        };

        // CHECKSTYLE:ON IndentationCheck
        t.traceEnd();

        t.traceBegin("addSharedUsers");
//添加system phone log nfc bluetooth shell se net networkstack这8种shareUserId带mSetting
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.se", SE_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.networkstack", NETWORKSTACK_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        t.traceEnd();

        String separateProcesses = SystemProperties.get("debug.separate_processes");
        if (separateProcesses != null && separateProcesses.length() > 0) {
            if ("*".equals(separateProcesses)) {
                mDefParseFlags = PackageParser.PARSE_IGNORE_PROCESSES;
                mSeparateProcesses = null;
                Slog.w(TAG, "Running with debug.separate_processes: * (ALL)");
            } else {
                mDefParseFlags = 0;
                mSeparateProcesses = separateProcesses.split(",");
                Slog.w(TAG, "Running with debug.separate_processes: "
                        + separateProcesses);
            }
        } else {
            mDefParseFlags = 0;
            mSeparateProcesses = null;
        }
//DexOpt的优化
        mPackageDexOptimizer = new PackageDexOptimizer(mInstaller, mInstallLock, mContext,
                "*dexopt*");
        mDexManager =
                new DexManager(mContext, this, mPackageDexOptimizer, mInstaller, mInstallLock);
        //mArt虚拟机管理服务
				mArtManagerService = new ArtManagerService(mContext, this, mInstaller, mInstallLock);
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());

        mViewCompiler = new ViewCompiler(mInstallLock, mInstaller);

        getDefaultDisplayMetrics(mInjector.getDisplayManager(), mMetrics);

        t.traceBegin("get system config");
        SystemConfig systemConfig = SystemConfig.getInstance();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
        ApplicationPackageManager.invalidateHasSystemFeatureCache();
        t.traceEnd();

        mProtectedPackages = new ProtectedPackages(mContext);

        mApexManager = ApexManager.getInstance();
        mAppsFilter = mInjector.getAppsFilter();

        final List<ScanPartition> scanPartitions = new ArrayList<>();
        final List<ApexManager.ActiveApexInfo> activeApexInfos = mApexManager.getActiveApexInfos();
        for (int i = 0; i < activeApexInfos.size(); i++) {
            final ScanPartition scanPartition = resolveApexToScanPartition(activeApexInfos.get(i));
            if (scanPartition != null) {
                scanPartitions.add(scanPartition);
            }
        }

        mDirsToScanAsSystem = new ArrayList<>();
        mDirsToScanAsSystem.addAll(SYSTEM_PARTITIONS);
        mDirsToScanAsSystem.addAll(scanPartitions);
        Slog.d(TAG, "Directories scanned as system partitions: " + mDirsToScanAsSystem);

        // CHECKSTYLE:OFF IndentationCheck
        synchronized (mInstallLock) {
        // writer
        synchronized (mLock) {
          //启动packageManger线程，负责apk的安装、卸载
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            mHandler = new PackageHandler(mHandlerThread.getLooper());
          //进程记录handler
            mProcessLoggingHandler = new ProcessLoggingHandler();
          //watchdog监听ServiceThread是否超时， 
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);
          //mInstant应用注册
            mInstantAppRegistry = new InstantAppRegistry(this);
					//共享lib库配置
            ArrayMap<String, SystemConfig.SharedLibraryEntry> libConfig
                    = systemConfig.getSharedLibraries();
            final int builtInLibCount = libConfig.size();
            for (int i = 0; i < builtInLibCount; i++) {
                String name = libConfig.keyAt(i);
                SystemConfig.SharedLibraryEntry entry = libConfig.valueAt(i);
                addBuiltInSharedLibraryLocked(entry.filename, name);
            }

            // Now that we have added all the libraries, iterate again to add dependency
            // information IFF their dependencies are added.
            long undefinedVersion = SharedLibraryInfo.VERSION_UNDEFINED;
            for (int i = 0; i < builtInLibCount; i++) {
                String name = libConfig.keyAt(i);
                SystemConfig.SharedLibraryEntry entry = libConfig.valueAt(i);
                final int dependencyCount = entry.dependencies.length;
                for (int j = 0; j < dependencyCount; j++) {
                    final SharedLibraryInfo dependency =
                        getSharedLibraryInfoLPr(entry.dependencies[j], undefinedVersion);
                    if (dependency != null) {
                        getSharedLibraryInfoLPr(name, undefinedVersion).addDependency(dependency);
                    }
                }
            }
					//读取安装相关SELinux策略
            SELinuxMMAC.readInstallPolicy();

            t.traceBegin("loadFallbacks");
            FallbackCategoryProvider.loadFallbacks();
            t.traceEnd();

            t.traceBegin("read user settings");
          //读取并解析/data/system下的XML文件
            mFirstBoot = !mSettings.readLPw(mInjector.getUserManagerInternal().getUsers(false));
            t.traceEnd();

            //清理代码路径不存在的孤立软件包
            final int packageSettingCount = mSettings.mPackages.size();
            for (int i = packageSettingCount - 1; i >= 0; i--) {
                PackageSetting ps = mSettings.mPackages.valueAt(i);
                if (!isExternal(ps) && (ps.codePath == null || !ps.codePath.exists())
                        && mSettings.getDisabledSystemPkgLPr(ps.name) != null) {
                    mSettings.mPackages.removeAt(i);
                    mSettings.enableSystemPackageLPw(ps.name);
                }
            }
					//如果不是时候首次启动,也不是CORE应用，则拷贝预编译的DEX文件
            if (!mOnlyCore && mFirstBoot) {
                requestCopyPreoptedFiles();
            }
					
          //获取资配置
            String customResolverActivityName = Resources.getSystem().getString(
                    R.string.config_customResolverActivity);
            if (!TextUtils.isEmpty(customResolverActivityName)) {
                mCustomResolverComponentName = ComponentName.unflattenFromString(
                        customResolverActivityName);
            }
				//获取扫描开始的时间
            long startTime = SystemClock.uptimeMillis();
```

在第一阶段主要做了初始化的工作，为后续做扫描工作的准备。这里面你可能会发现一个复杂的数据结构Settings以及它的`addShareUserLPw`函数。而`Settiing`在里面又起了什么作用？

#### 1.1.1 Setting

首先我们先分析`addShareUserLPw`函数：

```java
//使用了系统进程使用的用户id
mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
                
```

这里面涉及到`SYSTEM_UID` :

UID为用户ID的缩写，GID为用户组ID的缩写，这两个概念均与Linux系统中进程的权限管理有关。一般说来，每一个进程都会有一个对应的UID（即表示该进程属于哪个user，不同user有不同权限）。一个进程也可分属不同的用户组（每个用户组都有对应的权限）。

接着我们再来看下：

```java
SharedUserSetting addSharedUserLPw(String name, int uid, int pkgFlags, int pkgPrivateFlags) {
	//mSharedUsers是一个hashMap,key为字符串，值为SharedUserSetting对象        
  SharedUserSetting s = mSharedUsers.get(name);
        if (s != null) {
            if (s.userId == uid) {
                return s;
            }
            PackageManagerService.reportSettingsProblem(Log.ERROR,
                    "Adding duplicate shared user, keeping first: " + name);
            return null;
        }
  		//创建一个新的ShareUserSettiing对象，并设置userId为uid
        s = new SharedUserSetting(name, pkgFlags, pkgPrivateFlags);
        s.userId = uid;
        if (registerExistingAppIdLPw(uid, s, name)) {
            mSharedUsers.put(name, s);
            return s;
        }
        return null;
    }

```

在这个方法里面会创建`ShareUserSetting`对象并添加到`Settings`的成员变量`mShareUsers`中。在Android系统中，多个package可以通过设置`shareUserId属性`可以在同一个进程中运行，共享同一个UID；

可能会问`ShareuserSetting`是什么？可以从`AndroidfManifest.xml`中找到：

```xml
<manifestxmlns:android="http://schemas.android.com/apk/res/android"
       package="com.android.systemui"
       coreApp="true"
       android:sharedUserId="android.uid.system"
       android:process="system">
```

在这个里面声明了一个`android：shareUserId`的属性，它有两个作用：

- 两个或多个声明统一中`shareUserIds`的APK可共享彼此的数据，并且可运行在同一进程中
- 通过声明特定的`shareUserIds`，该APK所在进程将赋予指定的UID

### 1.2  BOOT_PROGRESS_PMS_SYSTEM_SCAN_START阶段

经过前面阶段的准备，PKMS服务相关的初始化环境已经构建OK，现在要开始真正干活开始系统扫描阶段了。这一部分主要做了：

1. 从`init.rc`中获取环境变量`BOOTCLASSPATH` 和 `SYSTEMSERVERCLASSPATH`；

2. 对于旧版本升级的情况，将安装时获取权限变更为运行时申请权限；

3. 扫描system/vendor/product/odm/oem等目录的priv-app、app、overlay包；

4. 清除安装时临时文件以及其他不必要的信息。

```java
long startTime = SystemClock.uptimeMillis();
					//写入开始扫描系统应用的日志
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);
	//获取环境变量，init.rc
            final String bootClassPath = System.getenv("BOOTCLASSPATH");
            final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");

            if (bootClassPath == null) {
                Slog.w(TAG, "No BOOTCLASSPATH found!");
            }

            if (systemServerClassPath == null) {
                Slog.w(TAG, "No SYSTEMSERVERCLASSPATH found!");
            }
//获取system/framework目录
            File frameworkDir = new File(Environment.getRootDirectory(), "framework");
// 获取内部版本
            final VersionInfo ver = mSettings.getInternalVersion();
// 判断fingerprint是否有更新            
mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);
            if (mIsUpgrade) {
                logCriticalInfo(Log.INFO,
                        "Upgrading from " + ver.fingerprint + " to " + Build.FINGERPRINT);
            }

            // 对于Android M之前版本升级上来的情况，需将系统应用程序权限从安装升级到运行时
            mPromoteSystemApps =
                    mIsUpgrade && ver.sdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1;

            // 对于Android N之前版本升级上来的情况，需像首次启动一样处理package
            mIsPreNUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N;

            mIsPreNMR1Upgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N_MR1;
            mIsPreQUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.Q;

            // 在扫描之前保存预先存在的系统package的名称，不希望自动为新系统应用授予运行时权限
            if (isDeviceUpgrading()) {
                mExistingPackages = new ArraySet<>(mSettings.mPackages.size());
                for (PackageSetting ps : mSettings.mPackages.values()) {
                    mExistingPackages.add(ps.name);
                }
            }

// 准备解析package的缓存
            mCacheDir = preparePackageParserCache();

            // 设置flag，而不在扫描安装时更改文件路径
            int scanFlags = SCAN_BOOTING | SCAN_INITIAL;

            if (mIsUpgrade || mFirstBoot) {
                scanFlags = scanFlags | SCAN_FIRST_BOOT_OR_UPGRADE;
            }

            final int systemParseFlags = mDefParseFlags | PackageParser.PARSE_IS_SYSTEM_DIR;
            final int systemScanFlags = scanFlags | SCAN_AS_SYSTEM;

            PackageParser2 packageParser = new PackageParser2(mSeparateProcesses, mOnlyCore,
                    mMetrics, mCacheDir, mPackageParserCallback);

            ExecutorService executorService = ParallelPackageParser.makeExecutorService();
            
            mApexManager.scanApexPackagesTraced(packageParser, executorService);
            
					//扫描以下路径
            for (int i = mDirsToScanAsSystem.size() - 1; i >= 0; i--) {
                final ScanPartition partition = mDirsToScanAsSystem.get(i);
                if (partition.getOverlayFolder() == null) {
                    continue;
                }
                scanDirTracedLI(partition.getOverlayFolder(), systemParseFlags,
                        systemScanFlags | partition.scanFlag, 0,
                        packageParser, executorService);
            }

            scanDirTracedLI(frameworkDir, systemParseFlags,
                    systemScanFlags | SCAN_NO_DEX | SCAN_AS_PRIVILEGED, 0,
                    packageParser, executorService);
            if (!mPackages.containsKey("android")) {
                throw new IllegalStateException(
                        "Failed to load frameworks package; check log for warnings");
            }
            for (int i = 0, size = mDirsToScanAsSystem.size(); i < size; i++) {
                final ScanPartition partition = mDirsToScanAsSystem.get(i);
                if (partition.getPrivAppFolder() != null) {
                    scanDirTracedLI(partition.getPrivAppFolder(), systemParseFlags,
                            systemScanFlags | SCAN_AS_PRIVILEGED | partition.scanFlag, 0,
                            packageParser, executorService);
                }
                scanDirTracedLI(partition.getAppFolder(), systemParseFlags,
                        systemScanFlags | partition.scanFlag, 0,
                        packageParser, executorService);
            }

            mOverlayConfig = OverlayConfig.initializeSystemInstance(
                    consumer -> mPmInternal.forEachPackage(
                            pkg -> consumer.accept(pkg, pkg.isSystem())));

            // Prune any system packages that no longer exist.
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<>();
            
            final List<String> stubSystemApps = new ArrayList<>();
					//删除不存在的package
            if (!mOnlyCore) {
                // do this first before mucking with mPackages for the "expecting better" case
                final Iterator<AndroidPackage> pkgIterator = mPackages.values().iterator();
                while (pkgIterator.hasNext()) {
                    final AndroidPackage pkg = pkgIterator.next();
                    if (pkg.isStub()) {
                        stubSystemApps.add(pkg.getPackageName());
                    }
                }

                final Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
                while (psit.hasNext()) {
                    PackageSetting ps = psit.next();

                    // 如果不是系统应用，则不被允许disable
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                        continue;
                    }

                    // 如果应用被扫描，则不允许被擦除
                    final AndroidPackage scannedPkg = mPackages.get(ps.name);
                    if (scannedPkg != null) {
                        // 如果系统应用被扫描且存在disable应用列表中，则只能通过OTA升级添加
                        if (mSettings.isDisabledSystemPackageLPr(ps.name)) {
                            logCriticalInfo(Log.WARN,
                                    "Expecting better updated system app for " + ps.name
                                    + "; removing system app.  Last known"
                                    + " codePath=" + ps.codePathString
                                    + ", versionCode=" + ps.versionCode
                                    + "; scanned versionCode=" + scannedPkg.getLongVersionCode());
                            removePackageLI(scannedPkg, true);
                            mExpectingBetter.put(ps.name, ps.codePath);
                        }

                        continue;
                    }

                    if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
                        psit.remove();
                        logCriticalInfo(Log.WARN, "System package " + ps.name
                                + " no longer exists; it's data will be wiped");

                        // Assume package is truly gone and wipe residual permissions.
                        mPermissionManager.updatePermissions(ps.name, null);

                        // Actual deletion of code and data will be handled by later
                        // reconciliation step
                    } else {
                        // we still have a disabled system package, but, it still might have
                        // been removed. check the code path still exists and check there's
                        // still a package. the latter can happen if an OTA keeps the same
                        // code path, but, changes the package name.
                        final PackageSetting disabledPs =
                                mSettings.getDisabledSystemPkgLPr(ps.name);
                        if (disabledPs.codePath == null || !disabledPs.codePath.exists()
                                || disabledPs.pkg == null) {
                            possiblyDeletedUpdatedSystemApps.add(ps.name);
                        } else {
                            // We're expecting that the system app should remain disabled, but add
                            // it to expecting better to recover in case the data version cannot
                            // be scanned.
                            mExpectingBetter.put(disabledPs.name, disabledPs.codePath);
                        }
                    }
                }
            }

            final int cachedSystemApps = PackageCacher.sCachedPackageReadCount.get();

            ......
```

在这类可以看到进行了扫描指定目录写下的apk文件，这个是PKMS的三大核心功能，它会通过调用PackageParser来完成对应用App中Apk的`AndroidManifest.xml`的文件的解析，生成 `Application`，`activity`，`service`，`broadcast`，`provider`等四大组件信息，并将上述解析得到的四大组件信息注册到PKMS中，供Android系统查询并使用。

### 1.3 BOOT_PROGRESS_PMS_DATA_SCAN_START阶段

在这里不仅仅解析核心应用的情况下，还处理data目录的应用信息，及时更新，去除不必要的数据。

```java
if (!mOnlyCore) {
                EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                        SystemClock.uptimeMillis());
                scanDirTracedLI(sAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0,
                        packageParser, executorService);

            }

            packageParser.close();

            List<Runnable> unfinishedTasks = executorService.shutdownNow();
            if (!unfinishedTasks.isEmpty()) {
                throw new IllegalStateException("Not all tasks finished before calling close: "
                        + unfinishedTasks);
            }

            if (!mOnlyCore) {
                // 移除通过OTA删除的更新系统应用程序的禁用package设置
                // 如果更新不再存在，则完全删除该应用。否则，撤消其系统权限
                for (int i = possiblyDeletedUpdatedSystemApps.size() - 1; i >= 0; --i) {
                    final String packageName = possiblyDeletedUpdatedSystemApps.get(i);
                    final AndroidPackage pkg = mPackages.get(packageName);
                    final String msg;

                    mSettings.removeDisabledSystemPackageLPw(packageName);

                   ......
                }

                //确保期望在userdata分区上显示的所有系统应用程序实际显示
                // 如果从未出现过，需要回滚以恢复系统版本
                for (int i = 0; i < mExpectingBetter.size(); i++) {
                    final String packageName = mExpectingBetter.keyAt(i);
                    if (!mPackages.containsKey(packageName)) {
                        final File scanFile = mExpectingBetter.valueAt(i);

                       ......
                        mSettings.enableSystemPackageLPw(packageName);

                        try {
                          //扫描APK
                            scanPackageTracedLI(scanFile, reparseFlags, rescanFlags, 0, null);
                        } catch (PackageManagerException e) {
                            Slog.e(TAG, "Failed to parse original system package: "
                                    + e.getMessage());
                        }
                    }
                }

                // 解压缩并安装任何存根系统应用程序。必须最后执行此操作以确保替换或禁用所有存根
                installSystemStubPackages(stubSystemApps, scanFlags);

               ......

            // 获取storage manager包名.
            mStorageManagerPackage = getStorageManagerPackageName();

            // 解决受保护的action过滤器。只允许setup wizard（开机向导）为这些action设置高优先级过滤器
            mSetupWizardPackage = getSetupWizardPackageNameImpl();
            mComponentResolver.fixProtectedFilterPriorities();

            ......

            // 更新客户端以确保持有正确的共享库路径
            updateAllSharedLibrariesLocked(null, null, Collections.unmodifiableMap(mPackages));

           ......

            // 读取并更新要保留的package的上次使用时间
            mPackageUsage.read(mSettings.mPackages);
            mCompilerStats.read();
```

### 1.4 **BOOT_PROGRESS_PMS_SCAN_END**阶段

主要做了：

1. 当sdk版本不一致时，需要更新权限
2. 当这是ota的首次启动，正常启动则需要清楚目录的缓存代码
3. 当权限和其他默认项都完成更新，则清理相关信息
4. 信息写回`packages.xml`

```java
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());
            Slog.i(TAG, "Time to scan packages: "
                    + ((SystemClock.uptimeMillis()-startTime)/1000f)
                    + " seconds");

            // // 如果自上次启动以来，平台SDK已改变，则需要重新授予应用程序权限以捕获出现的任何新权限
            final boolean sdkUpdated = (ver.sdkVersion != mSdkVersion);
            if (sdkUpdated) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
            }
            mPermissionManager.updateAllPermissions(
                    StorageManager.UUID_PRIVATE_INTERNAL, sdkUpdated);
            ver.sdkVersion = mSdkVersion;

            // 如果这是第一次启动或来自Android M之前的版本的升级，并且它是正常启动，那需要在所有已定义的用户中初始化默认的首选应用程序
            if (!mOnlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : mInjector.getUserManagerInternal().getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            // 在启动期间确实为系统用户准备存储，因为像SettingsProvider和SystemUI这样的核心系统应用程序无法等待用户启动
            final int storageFlags;
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
            List<String> deferPackages = reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL,
                    UserHandle.USER_SYSTEM, storageFlags, true /* migrateAppData */,
                    true /* onlyCoreApps */);
            mPrepareAppDataFuture = SystemServerInitThreadPool.submit(() -> {
                TimingsTraceLog traceLog = new TimingsTraceLog("SystemServerTimingAsync",
                        Trace.TRACE_TAG_PACKAGE_MANAGER);
                traceLog.traceBegin("AppDataFixup");
                try {
                    mInstaller.fixupAppData(StorageManager.UUID_PRIVATE_INTERNAL,
                            StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                } catch (InstallerException e) {
                    Slog.w(TAG, "Trouble fixing GIDs", e);
                }
                traceLog.traceEnd();

              ......

            // 如果是在OTA之后首次启动，并且正常启动，那需要清除代码缓存目录，但不清除应用程序配置文件
            if (mIsUpgrade && !mOnlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        // No apps are running this early, so no need to freeze
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                FLAG_STORAGE_DE | FLAG_STORAGE_CE | FLAG_STORAGE_EXTERNAL
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY
                                        | Installer.FLAG_CLEAR_APP_DATA_KEEP_ART_PROFILES);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            // 安装Android-Q前的非系统应用程序在Launcher中隐藏他们的图标
            if (!mOnlyCore && mIsPreQUpgrade) {
                Slog.i(TAG, "Whitelisting all existing apps to hide their icons");
                int size = mSettings.mPackages.size();
                for (int i = 0; i < size; i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                        continue;
                    }
                    ps.disableComponentLPw(PackageManager.APP_DETAILS_ACTIVITY_CLASS_NAME,
                            UserHandle.USER_SYSTEM);
                }
            }

            // 仅在权限或其它默认配置更新后清除
            mPromoteSystemApps = false;

            // 所有变更均在扫描过程中完成
            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            //降级去读取
            mSettings.writeLPr();
            t.traceEnd();
```

### 1.5 BOOT_PROGRESS_PMS_READY阶段

1. 初始化PackageInstallerService
2. GC回收写下内存

```java
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());

            if (!mOnlyCore) {
              //PermissionController 主持 缺陷许可证的授予和角色管理，所以这是核心系统的一个关键部分
                mRequiredVerifierPackage = getRequiredButNotReallyRequiredVerifierLPr();
                ......
                mServicesExtensionPackageName = getRequiredServicesExtensionPackageLPr();
                mSharedSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SHARED,
                        SharedLibraryInfo.VERSION_UNDEFINED);
            } else {
                mRequiredVerifierPackage = null;
                mRequiredInstallerPackage = null;
                mRequiredUninstallerPackage = null;
                mIntentFilterVerifierComponent = null;
                mIntentFilterVerifier = null;
                mServicesExtensionPackageName = null;
                mSharedSystemSharedLibraryPackageName = null;
            }

            // PermissionController hosts default permission granting and role management, so it's a
            // critical part of the core system.
            mRequiredPermissionControllerPackage = getRequiredPermissionControllerLPr();

            mSettings.setPermissionControllerVersion(
                    getPackageInfo(mRequiredPermissionControllerPackage, 0,
                            UserHandle.USER_SYSTEM).getLongVersionCode());

            ......
            // 阅读并更新dex文件的用法
            // 在PM init结束时执行此操作，以便所有程序包都已协调其数据目录
            // 此时知道了包的代码路径，因此可以验证磁盘文件并构建内部缓存
            // 使用文件预计很小，因此与其他活动（例如包扫描）相比，加载和验证它应该花费相当小的时间
            final Map<Integer, List<PackageInfo>> userPackages = new HashMap<>();
            for (int userId : userIds) {
                userPackages.put(userId, getInstalledPackages(/*flags*/ 0, userId).getList());
            }
            mDexManager.load(userPackages);
            if (mIsUpgrade) {
                FrameworkStatsLog.write(
                        FrameworkStatsLog.BOOT_TIME_EVENT_DURATION_REPORTED,
                        BOOT_TIME_EVENT_DURATION__EVENT__OTA_PACKAGE_MANAGER_INIT_TIME,
                        SystemClock.uptimeMillis() - startTime);
            }
        } // synchronized (mLock)
        } // synchronized (mInstallLock)
        // CHECKSTYLE:ON IndentationCheck

        mModuleInfoProvider = new ModuleInfoProvider(mContext, this);

        // Uncork cache invalidations and allow clients to cache package information.
        PackageManager.uncorkPackageInfoCache();

        // Now after opening every single application zip, make sure they
        // are all flushed.  Not really needed, but keeps things nice and
        // tidy.
        t.traceBegin("GC");
        Runtime.getRuntime().gc();
        t.traceEnd();

        // 打开应用之后，及时回收处理
        mInstaller.setWarnIfHeld(mLock);
      // 上面的初始扫描在持有mPackage锁的同时对installd进行了多次调用
        PackageParser.readConfigUseRoundIcon(mContext.getResources());
        mServiceStartWithDelay = SystemClock.uptimeMillis() + (60 * 1000L);
```

到这里基本把PMS构造函数流程看了一遍，到`BOOT_PEOGRESS_END`就算安装结束了。

## 二、 APK的安装过程

一讲到安装流程，它有四种安装方式：

1. 系统应用和预制应用安装，开机时完成，没有安装界面，在`PKMS`的构造函数中欧冠完成安装
2. 网络下载应用安装，通过应用商店来完成，调用`PackageManager.installPackages()`，有安装界面
3. ADB工具安装，没有安装界面，它通过启动pm脚本的形式，然后调用`com.android.commands.pm.Pm类`，之后调用到PMS.installStage()完成安装
4. 第三方应用安装，通过SD卡里的APK文件安装，有安装界面，由`packageinstaller.apk`应用处理安装及卸载过程的界面

> 均是通过**PackageInstallObserver来监听安装是否成功**。

下面我们通过点击下载应用安装来了解安装的过程：

先说个大概：

- 1.将APK的信息通过IO流的形式写入到`PackageInstaller.Session`中。
- 2.调用`PackageInstaller.Session`的commit方法，将APK的信息交由PKMS处理。
- 3.拷贝APK
- 4.最后进行安装

在点击一个未安装的apk后，会弹出安装界面，点击确定按钮后，会进入`PackageInstallerActivity`界面，后面会触发bindUi方法，弹出底部安装界面。这个主要是由`bindUi`构成，上面会有取消和安装两个按钮，点击之后就会调用`startInstall()`进行安装

```java
//PackageInstallerActivity.java
private void bindUi() {
        mAlert.setIcon(mAppSnippet.icon);
        mAlert.setTitle(mAppSnippet.label);
        mAlert.setView(R.layout.install_content_view);
        mAlert.setButton(DialogInterface.BUTTON_POSITIVE, getString(R.string.install),
                (ignored, ignored2) -> {
                    if (mOk.isEnabled()) {
                        if (mSessionId != -1) {
                            mInstaller.setPermissionsResult(mSessionId, true);
                            finish();
                        } else {
                          //进行APK安装
                            startInstall();
                        }
                    }
                }, null);
        mAlert.setButton(DialogInterface.BUTTON_NEGATIVE, getString(R.string.cancel),
                (ignored, ignored2) -> {
                    // Cancel and finish
                    setResult(RESULT_CANCELED);
                    if (mSessionId != -1) {
                      //如果mSessionId存在，执行setPermissionsResult()完成取消安装
                        mInstaller.setPermissionsResult(mSessionId, false);
                    }
                    finish();
                }, null);
        setupAlert();

       ......
    }

//点击”安装“，跳转 InstallInstalling - 开始安装
private void startInstall() {
    // Start subactivity to actually install the application
    Intent newIntent = new Intent();
    newIntent.putExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO, mPkgInfo.applicationInfo);
    newIntent.setData(mPackageURI);
    newIntent.setClass(this, InstallInstalling.class);
    String installerPackageName = getIntent().getStringExtra( Intent.EXTRA_INSTALLER_PACKAGE_NAME);
    ...
    if (installerPackageName != null) {
        newIntent.putExtra(Intent.EXTRA_INSTALLER_PACKAGE_NAME, installerPackageName);
    }
    newIntent.addFlags(Intent.FLAG_ACTIVITY_FORWARD_RESULT);
    startActivity(newIntent);
    finish();
}

```

在`startInstall`方法组装了一个Intent，并跳转到`InstallInstalling`这个Activity，并关闭掉当前的`PackageInstallerActivity`。在`InstallInstalling`主要用于向包管理器发送包的信息并处理包管理的回调。

### 2.1  PackageInstaller安装APK

在启动`InstallInstalling`后，进入`onCreate`方法:

```java
//InstallInstalling
protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        ApplicationInfo appInfo = getIntent()
                .getParcelableExtra(PackageUtil.INTENT_ATTR_APPLICATION_INFO);
        mPackageURI = getIntent().getData();

          ......
            setupAlert();
            requireViewById(R.id.installing).setVisibility(View.VISIBLE);

            if (savedInstanceState != null) {
                mSessionId = savedInstanceState.getInt(SESSION_ID);
                mInstallId = savedInstanceState.getInt(INSTALL_ID);

                try {
                  //.根据mInstallId向InstallEventReceiver注册一个观察者，launchFinishBasedOnResult会接收到安装事件的回调，无论安装成功或者失败都会关闭当前的Activity(InstallInstalling)。如果savedInstanceState为null，代码的逻辑也是类似的
                    InstallEventReceiver.addObserver(this, mInstallId,
                            this::launchFinishBasedOnResult);
                } catch (EventResultPersister.OutOfIdsException e) {
                    // Does not happen
                }
            } else {
                ......

                File file = new File(mPackageURI.getPath());
                try {
                    PackageParser.PackageLite pkg = PackageParser.parsePackageLite(file, 0);
                    params.setAppPackageName(pkg.packageName);
                    params.setInstallLocation(pkg.installLocation);
                    params.setSize(
                            PackageHelper.calculateInstalledSize(pkg, false, params.abiOverride));
                } catch (PackageParser.PackageParserException e) {
                    Log.e(LOG_TAG, "Cannot parse package " + file + ". Assuming defaults.");
                    Log.e(LOG_TAG,
                            "Cannot calculate installed size " + file + ". Try only apk size.");
                    params.setSize(file.length());
                } catch (IOException e) {
                    Log.e(LOG_TAG,
                            "Cannot calculate installed size " + file + ". Try only apk size.");
                    params.setSize(file.length());
                }

                try {
                  //向InstallEventReceiver注册一个观察者返回一个新的mInstallId,
                  //其中InstallEventReceiver继承自BroadcastReceiver，用于接收安装事件并回调给EventResultPersister。
                    mInstallId = InstallEventReceiver
                            .addObserver(this, EventResultPersister.GENERATE_NEW_ID,
                                    this::launchFinishBasedOnResult);
                } catch (EventResultPersister.OutOfIdsException e) {
                    launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
                }

                try {
                  //PackageInstaller的createSession方法内部会通过IPackageInstaller与PackageInstallerService进行进程间通信，最终调用的是PackageInstallerService的createSession方法来创建并返回mSessionId
                    mSessionId = getPackageManager().getPackageInstaller().createSession(params);
                } catch (IOException e) {
                    launchFailure(PackageManager.INSTALL_FAILED_INTERNAL_ERROR, null);
                }
            }

            mCancelButton = mAlert.getButton(DialogInterface.BUTTON_NEGATIVE);

            mSessionCallback = new InstallSessionCallback();
        }
    }
```

在`onCreate`中通过`PackageInstaller`通过创建Session并返回mSesionId，接着会在`onResume`中，会开启InstallingAsynTask，把包信息写入mSessionId对应的session，然后提交。

```java
//InstallInstalling
protected void onResume() {
        super.onResume();

        // This is the first onResume in a single life of the activity
        if (mInstallingTask == null) {
            PackageInstaller installer = getPackageManager().getPackageInstaller();
          //获取sessionInfo
            PackageInstaller.SessionInfo sessionInfo = installer.getSessionInfo(mSessionId);

            if (sessionInfo != null && !sessionInfo.isActive()) {
              //创建内部类InstallingAsyncTask的对象，调用execute()，最终进入onPostExecute()
                mInstallingTask = new InstallingAsyncTask();
                mInstallingTask.execute();
            } else {
                // we will receive a broadcast when the install is finished
                mCancelButton.setEnabled(false);
                setFinishOnTouchOutside(false);
            }
        }
    }

  private final class InstallingAsyncTask extends AsyncTask<Void, Void,
            PackageInstaller.Session> {
              
    @Override
    protected PackageInstaller.Session doInBackground(Void... params) {
        
              PackageInstaller.Session session;
            try {
                session = getPackageManager().getPackageInstaller().openSession(mSessionId);
            } catch (IOException e) {
                return null;
            }

            session.setStagingProgress(0);

            try {
                File file = new File(mPackageURI.getPath());

                try (InputStream in = new FileInputStream(file)) {
                    long sizeBytes = file.length();
                  //从session中获取输出流
                    try (OutputStream out = session
                            .openWrite("PackageInstaller", 0, sizeBytes)) {
                        byte[] buffer = new byte[1024 * 1024];
                       ......
                    }
                }

                return session;
            } catch (IOException | SecurityException e) {
               ......
        }

        @Override
        protected void onPostExecute(PackageInstaller.Session session) {
            if (session != null) {
                Intent broadcastIntent = new Intent(BROADCAST_ACTION);
                broadcastIntent.setFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                broadcastIntent.setPackage(getPackageName());
                broadcastIntent.putExtra(EventResultPersister.EXTRA_ID, mInstallId);

                PendingIntent pendingIntent = PendingIntent.getBroadcast(
                        InstallInstalling.this,
                        mInstallId,
                        broadcastIntent,
                        PendingIntent.FLAG_UPDATE_CURRENT);
								//包写入session进行提交
                session.commit(pendingIntent.getIntentSender());
                mCancelButton.setEnabled(false);
                setFinishOnTouchOutside(false);
            } else {
                getPackageManager().getPackageInstaller().abandonSession(mSessionId);

                if (!isCancelled()) {
                    launchFailure(PackageManager.INSTALL_FAILED_INVALID_APK, null);
                }
            }
        }  
   }
```

在`InstallingAsyncTask的doInBackground()`里会根据包的Uri，将APK的信息通过IO流的形式写入到`PackageInstaller.Session`中，最后会在`onPostExecute()`中调用`PackageInstaller.Session`的commit方法，进行安装。

在里面会看到一个`PackageInstaller`，也就是APK安装器。而其实在`ApplicationPackageManager的getPackageInstaller`中创建的：

```java
//ApplicationPackageManager
@Override
    public PackageInstaller getPackageInstaller() {
        synchronized (mLock) {
            if (mInstaller == null) {
                try {
                    mInstaller = new PackageInstaller(mPM.getPackageInstaller(),
                            mContext.getPackageName(), getUserId());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return mInstaller;
        }
    }
```

在这里会传入`mPM.getPackageInstaller()`，也就是`IpacageInstaller`的实例，其具体实现也就是`PackageInstallerService`， 其通过IPC的方式。它在初始化的时候会读取/data/system目录下的`install_sessions`文件，这个文件保存了系统未完成的`Install Session`。PMS则会根据文件的内容创建`PackageInstallerSession`对象并从插入到mSessions中。

```java
//PackageInstallerService.java
 public PackageInstallerService(Context context, PackageManagerService pm,
            Supplier<PackageParser2> apexParserSupplier) {
        mContext = context;
        mPm = pm;
        mPermissionManager = LocalServices.getService(PermissionManagerServiceInternal.class);

        mInstallThread = new HandlerThread(TAG);
        mInstallThread.start();

        mInstallHandler = new Handler(mInstallThread.getLooper());

        mCallbacks = new Callbacks(mInstallThread.getLooper());

        mSessionsFile = new AtomicFile(
                new File(Environment.getDataSystemDirectory(), "install_sessions.xml"),
                "package-session");
   //这个文件保存了系统未完成的`Install Session`
        mSessionsDir = new File(Environment.getDataSystemDirectory(), "install_sessions");
        mSessionsDir.mkdirs();

        mApexManager = ApexManager.getInstance();
        mStagingManager = new StagingManager(this, context, apexParserSupplier);
    }
```


再来看下`Session`，是在于mSeesionId绑定的安装会话，代表着一个在进行中的安装。Session类是对IPackageInstaller.openSession(sessionId) 获取的 PackageInstallerSession（系统服务端）的封装。

Session的创建和打开 具体实现是在 PackageInstallerService中，主要是 初始化apk的安装信息及环境，并创建一个sessionId，将安装Session与sessionId 进行绑定

接着我们回到`InstallingAsyncTask`中，在这里调用了`session.commit`方法：

```java
//PackageInstaller        
public void commit(@NonNull IntentSender statusReceiver) {
            try {
              //调用PackageInstallerSession的commit方法,进入到java框架层
                mSession.commit(statusReceiver, false);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }

//PackageInstallerSession.java
public void commit(@NonNull IntentSender statusReceiver, boolean forTransfer) {
				......
          //如果尚未调用，则会话将被密封。此方法可能会被多次调用以更新状态接收者验证调用者权限
		if (!markAsSealed(statusReceiver, forTransfer)) {
            return;
        }
  		//不同的包
        if (isMultiPackage()) {
            final SparseIntArray remainingSessions = mChildSessionIds.clone();
            final IntentSender childIntentSender =
                    new ChildStatusIntentReceiver(remainingSessions, statusReceiver)
                            .getIntentSender();
            boolean sealFailed = false;
            for (int i = mChildSessionIds.size() - 1; i >= 0; --i) {
                final int childSessionId = mChildSessionIds.keyAt(i);
                // seal all children, regardless if any of them fail; we'll throw/return
                // as appropriate once all children have been processed
                if (!mSessionProvider.getSession(childSessionId)
                        .markAsSealed(childIntentSender, forTransfer)) {
                    sealFailed = true;
                }
            }
            if (sealFailed) {
                return;
            }
        }
        dispatchStreamValidateAndCommit();
}

private void dispatchStreamValidateAndCommit() {
        mHandler.obtainMessage(MSG_STREAM_VALIDATE_AND_COMMIT).sendToTarget();
    }
  
```

`mSession`的类型为`IPackageInstallerSession`，这说明要通过`IPackageInstallerSession`来进行进程间的通信，最终会调用`PackageInstallerSession`的`commit`方法。

在这里发送了一个`MSG_STREAM_VALIDATE_AND_COMMIT`的信号，并在handler中进行处理：

```java
public boolean handleMessage(Message msg) {
  		case MSG_STREAM_VALIDATE_AND_COMMIT:
           handleStreamValidateAndCommit();
           break;
  			case MSG_INSTALL:
                handleInstall(); //
                break;
  		......
}

    private void handleStreamValidateAndCommit() {
        ......
            if (unrecoverableFailure != null) {
                onSessionVerificationFailure(unrecoverableFailure);
                // fail other child sessions that did not already fail
                for (int i = nonFailingSessions.size() - 1; i >= 0; --i) {
                    PackageInstallerSession session = nonFailingSessions.get(i);
                    session.onSessionVerificationFailure(unrecoverableFailure);
                }
            }
        }

        if (!allSessionsReady) {
            return;
        }

        mHandler.obtainMessage(MSG_INSTALL).sendToTarget();
    }
```

在`handleStreamValidateAndCommit`又发送了消息`MSG_INSTALL`,实际上真正在执行的是在`handleInstall`中:

```java
private void handleInstall() {
        ......

        // 对于 multiPackage 会话，请在锁之外读取子会话，因为在持有锁的情况下读取子会话可能会导致死锁 (b123391593)。
        List<PackageInstallerSession> childSessions = getChildSessionsNotLocked();

        try {
            synchronized (mLock) {
                installNonStagedLocked(childSessions);
            }
        } catch (PackageManagerException e) {
            final String completeMsg = ExceptionUtils.getCompleteMessage(e);
            Slog.e(TAG, "Commit of session " + sessionId + " failed: " + completeMsg);
            destroyInternal();
            dispatchSessionFinished(e.error, completeMsg, null);
        }
    }

private void installNonStagedLocked(List<PackageInstallerSession> childSessions)
            throws PackageManagerException {
        ......
            if (!success) {
                sendOnPackageInstalled(mContext, mRemoteStatusReceiver, sessionId,
                        isInstallerDeviceOwnerOrAffiliatedProfileOwnerLocked(), userId, null,
                        failure.error, failure.getLocalizedMessage(), null);
                return;
            }
            mPm.installStage(installingChildSessions);
        } else {
            mPm.installStage(installingSession);
        }
    }
```

最后执行到了`PMS`的`installStage`方法。在上述的过程中，通过`PackageInstaller`维持了`Session`，把安装包写入到`Session`，真正的安装过程就要来看PMS了。

### 2.2  PMS执行安装

```java
//PackageManagerService.java
void installStage(List<ActiveInstallSession> children)
            throws PackageManagerException {
  //创建了类型未INIT_COPY的消息
        final Message msg = mHandler.obtainMessage(INIT_COPY);
  //创建InstallParams，它对应于包的安装数据
        final MultiPackageInstallParams params =
                new MultiPackageInstallParams(UserHandle.ALL, children);
        params.setTraceMethod("installStageMultiPackage")
                .setTraceCookie(System.identityHashCode(params));
        msg.obj = params;

        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStageMultiPackage",
                System.identityHashCode(msg.obj));
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(msg.obj));
  //将InstallParams通过消息发送出去
        mHandler.sendMessage(msg);
    }

```

handler对INIT_COPY的消息进行处理：

```java
//PackageManagerService.java
void doHandleMessage(Message msg) {
            switch (msg.what) {
                case INIT_COPY: {
                    HandlerParams params = (HandlerParams) msg.obj;
                    if (params != null) {
                        if (DEBUG_INSTALL) Slog.i(TAG, "init_copy: " + params);
                        Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                                System.identityHashCode(params));
                        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
                      //执行APK拷贝动作
                        params.startCopy();
                        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                    }
                    break;
                }
                ......
            }
  
  final void startCopy() {
            if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);
            handleStartCopy();
            handleReturnCode();
        }
```

在这里调用了两个方法`handleStartCopy`和`handleReturnCode`,其实现是在`InstallParams `中。

在`handleStartCopy`,做了以下操作：

1. 检查空间大小，如果空间不够则释放无用空间
2. 覆盖原有安装位置的文件，并根据返回结果来确定函数的返回值，并设置installFlags
3. 确定是否有任何已安装的包验证器，如有，则延迟检测。主要分三步：首先新建一个验证Intent，然后设置相关的信息，之后获取验证器列表，最后向每个验证器发送验证Intent

```java
public void handleStartCopy() {
  ......
    //解析包 返回最小的细节:pkgName、versionCode、安装所需空间大小、获取安装位置等
    pkgLite = PackageManagerServiceUtils.getMinimalPackageInfo(mContext,
                    origin.resolvedPath, installFlags, packageAbiOverride);
  ......
    //覆盖原有安装位置的文件，并根据返回结果来确定函数的返回值，并设置installFlags。
    if (ret == PackageManager.INSTALL_SUCCEEDED) {
                int loc = pkgLite.recommendedInstallLocation;
                if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
                    ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                    ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_APK;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                    ret = PackageManager.INSTALL_FAILED_INVALID_URI;
                } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
                    ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;
                } else {
                     .......
                }
            }
  
  //安装参数
  final InstallArgs args = createInstallArgs(this);
            mVerificationCompleted = true;
            mIntegrityVerificationCompleted = true;
            mEnableRollbackCompleted = true;
            mArgs = args;

            if (ret == PackageManager.INSTALL_SUCCEEDED) {
                final int verificationId = mPendingVerificationToken++;

                // apk完整性校验
                if (!origin.existing) {
                    PackageVerificationState verificationState =
                            new PackageVerificationState(this);
                    mPendingVerification.append(verificationId, verificationState);

                    sendIntegrityVerificationRequest(verificationId, pkgLite, verificationState);
                    ret = sendPackageVerificationRequest(
                            verificationId, pkgLite, verificationState);

                    ......
                }

}
```

然后来看下`handleReturnCode`方法：

```java
@Override
        void handleReturnCode() {
            ......
                if (mRet == PackageManager.INSTALL_SUCCEEDED) {
                  //执行APKcopy拷贝
                    mRet = mArgs.copyApk();
                }
          //执行安装
                processPendingInstall(mArgs, mRet);
            }
        }
```

APK的copy过程是如何拷贝的：

```java
//packageManagerService.java
int copyApk() {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "copyApk");
            try {
                return doCopyApk();
            } finally {
                Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
            }
        }

private int doCopyApk() {
            ......

            int ret = PackageManagerServiceUtils.copyPackage(
                    origin.file.getAbsolutePath(), codeFile);
            ......
            return ret;
        }
//继续追踪下去，他会到PackagemanagerSeriveUtils的copyFile方法

//PackagemanagerSeriveUtils
private static void copyFile(String sourcePath, File targetDir, String targetName)
            throws ErrnoException, IOException {
        if (!FileUtils.isValidExtFilename(targetName)) {
            throw new IllegalArgumentException("Invalid filename: " + targetName);
        }
        Slog.d(TAG, "Copying " + sourcePath + " to " + targetName);

        final File targetFile = new File(targetDir, targetName);
        final FileDescriptor targetFd = Os.open(targetFile.getAbsolutePath(),
                O_RDWR | O_CREAT, 0644);
        Os.chmod(targetFile.getAbsolutePath(), 0644);
        FileInputStream source = null;
        try {
            source = new FileInputStream(sourcePath);
            FileUtils.copy(source.getFD(), targetFd);
        } finally {
            IoUtils.closeQuietly(source);
        }
    }
```

在这里就通过文件流的操作，把Apk拷贝到/data/app的目录下了。结束完拷贝之后，就要进入真正的安装了，流程如下：

```java
//PackageManagerService.java
private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        if (args.mMultiPackageInstallParams != null) {
            args.mMultiPackageInstallParams.tryProcessInstallRequest(args, currentStatus);
        } else {
          //安装结果
            PackageInstalledInfo res = createPackageInstalledInfo(currentStatus);
          //创建一个新线程来处理安转参数来进行安装
            processInstallRequestsAsync(
                    res.returnCode == PackageManager.INSTALL_SUCCEEDED,
                    Collections.singletonList(new InstallRequest(args, res)));
        }
    }

//排队执行异步操作
private void processInstallRequestsAsync(boolean success,
            List<InstallRequest> installRequests) {
        mHandler.post(() -> {
            if (success) {
                for (InstallRequest request : installRequests) {
                  //进行检验，如果之前安装失败，则清除无用信息
                    request.args.doPreInstall(request.installResult.returnCode);
                }
                synchronized (mInstallLock) {
                  //安装的核心方法，进行解析apk安装
                    installPackagesTracedLI(installRequests);
                }
                for (InstallRequest request : installRequests) {
                  //再次检验清除无用信息
                  
                    request.args.doPostInstall(
                            request.installResult.returnCode, request.installResult.uid);
                }
            }
            for (InstallRequest request : installRequests) {
              //备份、可能的回滚、发送安装完成先关广播
                restoreAndPostInstall(request.args.user.getIdentifier(), request.installResult,
                        new PostInstallData(request.args, request.installResult, null));
            }
        });
    }
```

看到了核心方法`installPackagesTracedLI`,接着内部执行到了`installPackagesLI`方法：

```java
//PackageMmanagerSerice.java 
private void installPackagesLI(List<InstallRequest> requests) {
   .......
     //分析当前任何状态，分析包并对其进行初始化验证
     prepareResult =
          preparePackageLI(request.args, request.installResult);
      
  ......
    //根据准备阶段解析包的信息上下文，进一步解析
      final ScanResult result = scanPackageTracedLI(
                            prepareResult.packageToScan, prepareResult.parseFlags,
                            prepareResult.scanFlags, System.currentTimeMillis(),
                            request.args.user, request.args.abiOverride);     
  	.......
      //验证扫描后包的信息好状态，确保安装成功
      reconciledPackages = reconcilePackagesLocked(
                            reconcileRequest, mSettings.mKeySetManagerService);
          
  		//提交所有的包并更新系统状态。这是安装流中唯一可以修改系统状态的地方，必须在此阶段之前确定所有可预测的错误
  		commitRequest = new CommitRequest(reconciledPackages,
                            mUserManager.getUserIds());
                    commitPackagesLocked(commitRequest);
  .......
    //完成APK安装
  executePostCommitSteps(commitRequest);
 }
```

由上面代码可知，`installPackagesLI`主要做了以下事情：

1. 分析当前任何状态，分析包并对其进行初始化验证
2. 根据准备阶段解析包的信息上下文，进一步解析
3. 验证扫描后包的信息好状态，确保安装成功
4. 提交所有骚哦欧庙的包并更新系统状态
5. 完成APK安装

在 preparePackageLI() 内使用 PackageParser2.parsePackage() 解析AndroidManifest.xml，获取四大组件等信息；使用ParsingPackageUtils.getSigningDetails() 解析签名信息；重命名包最终路径 等。

完成了解析和校验准备工作后，最后一步就是对apk的安装了。这里调用了`executePostCommitSteps`准备app数据，并执行dex优化。

```java
//PackageManagerService.java
private void executePostCommitSteps(CommitRequest commitRequest) {
  //进行安装
  prepareAppDataAfterInstallLIF(pkg);
   .......
   final boolean performDexopt =
                    (!instantApp || Global.getInt(mContext.getContentResolver(),
                    Global.INSTANT_APP_DEXOPT_ENABLED, 0) != 0)
                    && !pkg.isDebuggable()
                    && (!onIncremental);
  
  //为新的代码路径准备应用程序配置文件
  mArtManagerService.prepareAppProfiles(
                    pkg,
                    resolveUserIds(reconciledPkg.installArgs.user.getIdentifier()),
  
  if (performDexopt) {

             ......
               //其中分配了 dexopt 所需的库文件
               PackageSetting realPkgSetting = result.existingSettingCopied
                        ? result.request.pkgSetting : result.pkgSetting;
                if (realPkgSetting == null) {
                    realPkgSetting = reconciledPkg.pkgSetting;
                }
                
							//执行dex优化
                mPackageDexOptimizer.performDexOpt(pkg, realPkgSetting,
                        null /* instructionSets */,
                        getOrCreateCompilerPackageStats(pkg),
                        mDexManager.getPackageUseInfoOrDefault(packageName),
                        dexoptOptions);
                Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
            }
}
```

在`prepareAppDataAfterInstallLIF`方法中，经过一系列的调用，最中会调用到 `mInstaller.createAppData`，这里也是调用`Installd`守护进程的入口:

```java
public class Installer extends SystemService {
    
   @Override
    public void onStart() {
        if (mIsolated) {
            mInstalld = null;
        } else {
          //通过Binder调用到进程installd
            connect();
        }
    }

    private void connect() {
        IBinder binder = ServiceManager.getService("installd");
        ......

        if (binder != null) {
            mInstalld = IInstalld.Stub.asInterface(binder);
            try {
                invalidateMounts();
            } catch (InstallerException ignored) {
            }
        } else {
            Slog.w(TAG, "installd not found; trying again");
            BackgroundThread.getHandler().postDelayed(() -> {
                connect();
            }, DateUtils.SECOND_IN_MILLIS);
        }
    }
  
     public long createAppData(String uuid, String packageName, int userId, int flags, int appId,
            String seInfo, int targetSdkVersion) throws InstallerException {
        if (!checkBeforeRemote()) return -1;
        try {
          //进行安装操作
            return mInstalld.createAppData(uuid, packageName, userId, flags, appId, seInfo,
                    targetSdkVersion);
        } catch (Exception e) {
            throw InstallerException.from(e);
        }
    }
    }
```

可以看到最终调用了`Installd`的`createAppData`方法进行安装。Installer是Java层提供的Java API接口，Installd 则是在init进程启动的具有root权限的Daemon进程。

在`processInstallRequestsAsync`最后一步时调用了`restoreAndPostInstall`,在安装完成时会发送`POST_INSTALL`消息：

```java
//PackageManagerService.java
private void restoreAndPostInstall(
            int userId, PackageInstalledInfo res, @Nullable PostInstallData data) {
  
  .......
    if (!doRestore) {
            
            if (DEBUG_INSTALL) Log.v(TAG, "No restore - queue post-install for " + token);

            Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "postInstall", token);

            Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0);
            mHandler.sendMessage(msg);
        }
}

void doHandleMessage(Message msg) {
  .......
  case POST_INSTALL: {
    .......
      //处理安装结果
    handlePackagePostInstall(parentRes, grantPermissions,
                                killApp, virtualPreload, grantedPermissions,
                                whitelistedRestrictedPermissions, autoRevokePermissionsMode,
                                didRestore, args.installSource.installerPackageName, args.observer,
                                args.mDataLoaderType);
  }
}

private void handlePackagePostInstall(PackageInstalledInfo res, boolean grantPermissions,
            boolean killApp, boolean virtualPreload,
            String[] grantedPermissions, List<String> whitelistedRestrictedPermissions,
            int autoRevokePermissionsMode,
            boolean launchedForRestore, String installerPackage,
            IPackageInstallObserver2 installObserver, int dataLoaderType) {
  
  ......
    sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, packageName,
                        extras, 0 /*flags*/,
                        null /*targetPackage*/, null /*finishedReceiver*/,
                        updateUserIds, instantUserIds, newBroadcastWhitelist);
}
```

最后发送了`ACTION_PACKAGE_ADDED`广播，launcher接收到这个广播之后就会在桌面上添加应用图标了。

## 小结

本文主要讲了：

1. APK的信息会通过流的形式写入到`PackageInstaller.Session`中，并提交给PMS处理
2. PMS通过向`PackageHandler`发送消息来驱动APK的复制和安装工作
3. 复制APK完成后，会开始进行安装APK的流程，包括安装前的检查、安装APK和安装后的收尾工作
4. 然后通过`Installer`进行安装，安装完成发送广播出来

**参考**

- [Android包管理机制（五）APK是如何被解析的](http://liuwangshu.cn/framework/pms/5-packageparser.html)

- [PackageManagerService启动流程](https://maoao530.github.io/2017/01/10/packagemanager/)

- [Android包管理机制](https://duanqz.github.io/2017-01-04-Package-Manage-Mechanism#31-%E5%AF%B9%E8%B1%A1%E6%9E%84%E5%BB%BA)

- [Android 10.0 PackageManagerService（一）工作原理及启动流程-Android取经之路](https://blog.csdn.net/yiranfeng/article/details/103941371)

- [[深入理解Android卷二 全文-第四章]深入理解PackageManagerService](https://blog.csdn.net/Innost/article/details/47253179)

- [“终于懂了”系列：APK安装过程 完全解析！](https://juejin.cn/post/7028196921143459870#heading-6)

  





