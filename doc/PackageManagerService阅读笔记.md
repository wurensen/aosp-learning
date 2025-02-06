# PackageManagerService阅读笔记

> 以下分析基于android-28 8.1.0源码

由SystemServer创建PMS服务。

## PackageManagerService

PMS构造函数初始化：

```java
public PackageManagerService(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    LockGuard.installLock(mPackages, LockGuard.INDEX_PACKAGES);
    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "create package manager");
    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
            SystemClock.uptimeMillis());

    if (mSdkVersion <= 0) {
        Slog.w(TAG, "**** ro.build.version.sdk not set!");
    }

    mContext = context;

    mFactoryTest = factoryTest;
    mOnlyCore = onlyCore;
    mMetrics = new DisplayMetrics();
    mInstaller = installer;

    // Create sub-components that provide services / data. Order here is important.
    synchronized (mInstallLock) {
    synchronized (mPackages) {
        // Expose private service for system components to use.
        LocalServices.addService(
                PackageManagerInternal.class, new PackageManagerInternalImpl());
        sUserManager = new UserManagerService(context, this,
                new UserDataPreparer(mInstaller, mInstallLock, mContext, mOnlyCore), mPackages);
        mPermissionManager = PermissionManagerService.create(context,
                new DefaultPermissionGrantedCallback() {
                    @Override
                    public void onDefaultRuntimePermissionsGranted(int userId) {
                        synchronized(mPackages) {
                            mSettings.onDefaultRuntimePermissionsGrantedLPr(userId);
                        }
                    }
                }, mPackages /*externalLock*/);
        mDefaultPermissionPolicy = mPermissionManager.getDefaultPermissionGrantPolicy();
        mSettings = new Settings(mPermissionManager.getPermissionSettings(), mPackages);
    }
    }
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

    mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
            "*dexopt*");
    DexManager.Listener dexManagerListener = DexLogger.getListener(this,
            installer, mInstallLock);
    mDexManager = new DexManager(mContext, this, mPackageDexOptimizer, installer, mInstallLock,
            dexManagerListener);
    mArtManagerService = new ArtManagerService(mContext, this, installer, mInstallLock);
    mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());

    mOnPermissionChangeListeners = new OnPermissionChangeListeners(
            FgThread.get().getLooper());

    getDefaultDisplayMetrics(context, mMetrics);

    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "get system config");
    SystemConfig systemConfig = SystemConfig.getInstance();
    mAvailableFeatures = systemConfig.getAvailableFeatures();
    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

    mProtectedPackages = new ProtectedPackages(mContext);

    synchronized (mInstallLock) {
    // writer
    synchronized (mPackages) {
        mHandlerThread = new ServiceThread(TAG,
                Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
        mHandlerThread.start();
        mHandler = new PackageHandler(mHandlerThread.getLooper());
        mProcessLoggingHandler = new ProcessLoggingHandler();
        Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);
        mInstantAppRegistry = new InstantAppRegistry(this);

        ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
        final int builtInLibCount = libConfig.size();
        for (int i = 0; i < builtInLibCount; i++) {
            String name = libConfig.keyAt(i);
            String path = libConfig.valueAt(i);
            addSharedLibraryLPw(path, null, name, SharedLibraryInfo.VERSION_UNDEFINED,
                    SharedLibraryInfo.TYPE_BUILTIN, PLATFORM_PACKAGE_NAME, 0);
        }

        SELinuxMMAC.readInstallPolicy();

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "loadFallbacks");
        FallbackCategoryProvider.loadFallbacks();
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "read user settings");
        // 通过Settings.readLPw从缓存文件data/system/packages.xml中读取缓存的信息
		// 如果没有缓存，当成首次开机，mFirstBoot设置为true
        mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

        // Clean up orphaned packages for which the code path doesn't exist
        // and they are an update to a system app - caused by bug/32321269
        final int packageSettingCount = mSettings.mPackages.size();
        for (int i = packageSettingCount - 1; i >= 0; i--) {
            PackageSetting ps = mSettings.mPackages.valueAt(i);
            if (!isExternal(ps) && (ps.codePath == null || !ps.codePath.exists())
                    && mSettings.getDisabledSystemPkgLPr(ps.name) != null) {
                mSettings.mPackages.removeAt(i);
                mSettings.enableSystemPackageLPw(ps.name);
            }
        }

        if (mFirstBoot) {
            requestCopyPreoptedFiles();
        }

        String customResolverActivity = Resources.getSystem().getString(
                R.string.config_customResolverActivity);
        if (TextUtils.isEmpty(customResolverActivity)) {
            customResolverActivity = null;
        } else {
            mCustomResolverComponentName = ComponentName.unflattenFromString(
                    customResolverActivity);
        }

        long startTime = SystemClock.uptimeMillis();

        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                startTime);

        final String bootClassPath = System.getenv("BOOTCLASSPATH");
        final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");

        if (bootClassPath == null) {
            Slog.w(TAG, "No BOOTCLASSPATH found!");
        }

        if (systemServerClassPath == null) {
            Slog.w(TAG, "No SYSTEMSERVERCLASSPATH found!");
        }

        File frameworkDir = new File(Environment.getRootDirectory(), "framework");

        final VersionInfo ver = mSettings.getInternalVersion();
        // 判断构建指纹是否一致，不一致认为是ota升级（注意：rom打包后，增量构建打出来的差分包构建指纹可能一致，取决于构建指纹包含的信息有没有产生变化，如果一致此时测试升级会被认为是未升级，mIsUpgrade被设置为false）
        mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);
        if (mIsUpgrade) {
            logCriticalInfo(Log.INFO,
                    "Upgrading from " + ver.fingerprint + " to " + Build.FINGERPRINT);
        }

        ...

        // 如果是升级，会删除缓存目录：/data/system/package_cache，该目录下会缓存每个应用的PackageParser.Package
        mCacheDir = preparePackageParserCache(mIsUpgrade);

        // Set flag to monitor and not change apk file paths when
        // scanning install directories.
        int scanFlags = SCAN_BOOTING | SCAN_INITIAL;

        if (mIsUpgrade || mFirstBoot) {
            scanFlags = scanFlags | SCAN_FIRST_BOOT_OR_UPGRADE;
        }

        // Collect vendor/product overlay packages. (Do this before scanning any apps.)
        // For security and version matching reason, only consider
        // overlay packages if they reside in the right directory.
        scanDirTracedLI(new File(VENDOR_OVERLAY_DIR),
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_VENDOR,
                0);
        scanDirTracedLI(new File(PRODUCT_OVERLAY_DIR),
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_PRODUCT,
                0);

        mParallelPackageParserCallback.findStaticOverlayPackages();

        // Find base frameworks (resource packages without code).
        scanDirTracedLI(frameworkDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_NO_DEX
                | SCAN_AS_SYSTEM
                | SCAN_AS_PRIVILEGED,
                0);

        // Collect privileged system packages.
        final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
        scanDirTracedLI(privilegedAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_PRIVILEGED,
                0);

        // 扫描/system/app目录下的应用
        // Collect ordinary system packages.
        final File systemAppDir = new File(Environment.getRootDirectory(), "app");
        scanDirTracedLI(systemAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM,
                0);

        // Collect privileged vendor packages.
        File privilegedVendorAppDir = new File(Environment.getVendorDirectory(), "priv-app");
        try {
            privilegedVendorAppDir = privilegedVendorAppDir.getCanonicalFile();
        } catch (IOException e) {
            // failed to look up canonical path, continue with original one
        }
        scanDirTracedLI(privilegedVendorAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_VENDOR
                | SCAN_AS_PRIVILEGED,
                0);

        // Collect ordinary vendor packages.
        File vendorAppDir = new File(Environment.getVendorDirectory(), "app");
        try {
            vendorAppDir = vendorAppDir.getCanonicalFile();
        } catch (IOException e) {
            // failed to look up canonical path, continue with original one
        }
        scanDirTracedLI(vendorAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_VENDOR,
                0);

        // Collect privileged odm packages. /odm is another vendor partition
        // other than /vendor.
        File privilegedOdmAppDir = new File(Environment.getOdmDirectory(),
                    "priv-app");
        try {
            privilegedOdmAppDir = privilegedOdmAppDir.getCanonicalFile();
        } catch (IOException e) {
            // failed to look up canonical path, continue with original one
        }
        scanDirTracedLI(privilegedOdmAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_VENDOR
                | SCAN_AS_PRIVILEGED,
                0);

        // Collect ordinary odm packages. /odm is another vendor partition
        // other than /vendor.
        File odmAppDir = new File(Environment.getOdmDirectory(), "app");
        try {
            odmAppDir = odmAppDir.getCanonicalFile();
        } catch (IOException e) {
            // failed to look up canonical path, continue with original one
        }
        scanDirTracedLI(odmAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_VENDOR,
                0);

        // Collect all OEM packages.
        final File oemAppDir = new File(Environment.getOemDirectory(), "app");
        scanDirTracedLI(oemAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_OEM,
                0);

        // Collected privileged product packages.
        File privilegedProductAppDir = new File(Environment.getProductDirectory(), "priv-app");
        try {
            privilegedProductAppDir = privilegedProductAppDir.getCanonicalFile();
        } catch (IOException e) {
            // failed to look up canonical path, continue with original one
        }
        scanDirTracedLI(privilegedProductAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_PRODUCT
                | SCAN_AS_PRIVILEGED,
                0);

        // Collect ordinary product packages.
        File productAppDir = new File(Environment.getProductDirectory(), "app");
        try {
            productAppDir = productAppDir.getCanonicalFile();
        } catch (IOException e) {
            // failed to look up canonical path, continue with original one
        }
        scanDirTracedLI(productAppDir,
                mDefParseFlags
                | PackageParser.PARSE_IS_SYSTEM_DIR,
                scanFlags
                | SCAN_AS_SYSTEM
                | SCAN_AS_PRODUCT,
                0);
		...
    }
    ...
}
```



## 扫描指定scanDir下的package过程

### 开始扫描

PMS通过`scanDirTracedLI()->scanDirLI()`来扫描解析Package信息：

```java
private void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime) {
    final File[] files = scanDir.listFiles();
    if (ArrayUtils.isEmpty(files)) {
        Log.d(TAG, "No files in app dir " + scanDir);
        return;
    }

    if (DEBUG_PACKAGE_SCANNING) {
        Log.d(TAG, "Scanning app dir " + scanDir + " scanFlags=" + scanFlags
                + " flags=0x" + Integer.toHexString(parseFlags));
    }
    // 创建ParallelPackageParser来并发执行解析，mCacheDir在PMS构造函数中进行初始化，mCacheDir决定了后续是走缓存还是实际解析
    try (ParallelPackageParser parallelPackageParser = new ParallelPackageParser(
            mSeparateProcesses, mOnlyCore, mMetrics, mCacheDir,
            mParallelPackageParserCallback)) {
        // Submit files for parsing in parallel
        int fileCount = 0;
        for (File file : files) {
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            if (!isPackage) {
                // Ignore entries which are not packages
                continue;
            }
            // 提交解析任务
            parallelPackageParser.submit(file, parseFlags);
            fileCount++;
        }

        ...
    }
}
```

`ParallelPackageParser.submit`提交解析任务：

```java
/**
 * Submits the file for parsing
 * @param scanFile file to scan
 * @param parseFlags parse falgs
 */
public void submit(File scanFile, int parseFlags) {
    mService.submit(() -> {
        ParseResult pr = new ParseResult();
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parallel parsePackage [" + scanFile + "]");
        try {
            // 创建解析器
            PackageParser pp = new PackageParser();
            pp.setSeparateProcesses(mSeparateProcesses);
            pp.setOnlyCoreApps(mOnlyCore);
            pp.setDisplayMetrics(mMetrics);
            // 设置缓存所在目录
            pp.setCacheDir(mCacheDir);
            pp.setCallback(mPackageParserCallback);
            pr.scanFile = scanFile;
            // 执行解析得到package
            pr.pkg = parsePackage(pp, scanFile, parseFlags);
        } catch (Throwable e) {
            pr.throwable = e;
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
        try {
            // 放入队列
            mQueue.put(pr);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            // Propagate result to callers of take().
            // This is helpful to prevent main thread from getting stuck waiting on
            // ParallelPackageParser to finish in case of interruption
            mInterruptedInThread = Thread.currentThread().getName();
        }
    });
}

@VisibleForTesting
protected PackageParser.Package parsePackage(PackageParser packageParser, File scanFile,
        int parseFlags) throws PackageParser.PackageParserException {
    // 调用解析，优先使用缓存
    return packageParser.parsePackage(scanFile, parseFlags, true /* useCaches */);
}
```

```java
public Package parsePackage(File packageFile, int flags, boolean useCaches)
        throws PackageParserException {
    // 判断是否需要使用缓存
    Package parsed = useCaches ? getCachedResult(packageFile, flags) : null;
    if (parsed != null) {
        return parsed;
    }

    long parseTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
    if (packageFile.isDirectory()) {
        // 如果packageFile是目录，如系统某app存放格式就是：/system/app/Xxx/Xxx.apk，此时传入的packageFile为/system/app/Xxx
        parsed = parseClusterPackage(packageFile, flags);
    } else {
        parsed = parseMonolithicPackage(packageFile, flags);
    }

    long cacheTime = LOG_PARSE_TIMINGS ? SystemClock.uptimeMillis() : 0;
    // 缓存解析结果到缓存目录
    cacheResult(packageFile, flags, parsed);
    if (LOG_PARSE_TIMINGS) {
        parseTime = cacheTime - parseTime;
        cacheTime = SystemClock.uptimeMillis() - cacheTime;
        if (parseTime + cacheTime > LOG_PARSE_TIMINGS_THRESHOLD_MS) {
            Slog.i(TAG, "Parse times for '" + packageFile + "': parse=" + parseTime
                    + "ms, update_cache=" + cacheTime + " ms");
        }
    }
    return parsed;
}
```

### Package缓存

通过`getCachedResult`获取缓存：

```java
/**
 * Returns the cached parse result for {@code packageFile} for parse flags {@code flags},
 * or {@code null} if no cached result exists.
 */
private Package getCachedResult(File packageFile, int flags) {
    if (mCacheDir == null) {
        return null;
    }

    // 获取缓存key：Xxx-x
    final String cacheKey = getCacheKey(packageFile, flags);
    // 缓存文件，如：/data/system/package_cache/1/Xxx-X
    final File cacheFile = new File(mCacheDir, cacheKey);

    try {
        // 如果缓存失效，直接返回null
        // If the cache is not up to date, return null.
        if (!isCacheUpToDate(packageFile, cacheFile)) {
            return null;
        }

        // 读取缓存的package
        final byte[] bytes = IoUtils.readFileAsByteArray(cacheFile.getAbsolutePath());
        Package p = fromCacheEntry(bytes);
        if (mCallback != null) {
            String[] overlayApks = mCallback.getOverlayApks(p.packageName);
            if (overlayApks != null && overlayApks.length > 0) {
                for (String overlayApk : overlayApks) {
                    // If a static RRO is updated, return null.
                    if (!isCacheUpToDate(new File(overlayApk), cacheFile)) {
                        return null;
                    }
                }
            }
        }
        return p;
    } catch (Throwable e) {
        Slog.w(TAG, "Error reading package cache: ", e);

        // If something went wrong while reading the cache entry, delete the cache file
        // so that we regenerate it the next time.
        cacheFile.delete();
        return null;
    }
}

/**
 * Returns the cache key for a specificied {@code packageFile} and {@code flags}.
 */
private String getCacheKey(File packageFile, int flags) {
    StringBuilder sb = new StringBuilder(packageFile.getName());
    sb.append('-');
    sb.append(flags);
	// 缓存key规则就是packageFile-flags
    return sb.toString();
}

/**
 * Given a {@code packageFile} and a {@code cacheFile} returns whether the
 * cache file is up to date based on the mod-time of both files.
 */
private static boolean isCacheUpToDate(File packageFile, File cacheFile) {
    try {
        // NOTE: We don't use the File.lastModified API because it has the very
        // non-ideal failure mode of returning 0 with no excepions thrown.
        // The nio2 Files API is a little better but is considerably more expensive.
        final StructStat pkg = android.system.Os.stat(packageFile.getAbsolutePath());
        final StructStat cache = android.system.Os.stat(cacheFile.getAbsolutePath());
        // 比较packageFile的修改时间和缓存文件的修改时间，如果packageFile的时间更大，返回false
        return pkg.st_mtime < cache.st_mtime;
    } catch (ErrnoException ee) {
        ...
        return false;
    }
}
```

通过` cacheResult(packageFile, flags, parsed)`保存缓存：

```java
/**
 * Caches the parse result for {@code packageFile} with flags {@code flags}.
 */
private void cacheResult(File packageFile, int flags, Package parsed) {
    if (mCacheDir == null) {
        return;
    }

    try {
        final String cacheKey = getCacheKey(packageFile, flags);
        final File cacheFile = new File(mCacheDir, cacheKey);

        if (cacheFile.exists()) {
            if (!cacheFile.delete()) {
                Slog.e(TAG, "Unable to delete cache file: " + cacheFile);
            }
        }

        final byte[] cacheEntry = toCacheEntry(parsed);

        if (cacheEntry == null) {
            return;
        }

        try (FileOutputStream fos = new FileOutputStream(cacheFile)) {
            fos.write(cacheEntry);
        } catch (IOException ioe) {
            Slog.w(TAG, "Error writing cache entry.", ioe);
            cacheFile.delete();
        }
    } catch (Throwable e) {
        Slog.w(TAG, "Error saving package cache.", e);
    }
}
```

### 处理结果

```java
private void scanDirLI(File scanDir, int parseFlags, int scanFlags, long currentTime) {
    ...
    try (ParallelPackageParser parallelPackageParser = new ParallelPackageParser(
            mSeparateProcesses, mOnlyCore, mMetrics, mCacheDir,
            mParallelPackageParserCallback)) {
        // Submit files for parsing in parallel
        int fileCount = 0;
        for (File file : files) {
            final boolean isPackage = (isApkFile(file) || file.isDirectory())
                    && !PackageInstallerService.isStageName(file.getName());
            if (!isPackage) {
                // Ignore entries which are not packages
                continue;
            }
            // 提交解析任务
            parallelPackageParser.submit(file, parseFlags);
            fileCount++;
        }

        // Process results one by one
        for (; fileCount > 0; fileCount--) {
            // 获取解析结果
            ParallelPackageParser.ParseResult parseResult = parallelPackageParser.take();
            Throwable throwable = parseResult.throwable;
            int errorCode = PackageManager.INSTALL_SUCCEEDED;

            if (throwable == null) {
                // TODO(toddke): move lower in the scan chain
                // Static shared libraries have synthetic package names
                if (parseResult.pkg.applicationInfo.isStaticSharedLibrary()) {
                    renameStaticSharedLibraryPackage(parseResult.pkg);
                }
                try {
                    if (errorCode == PackageManager.INSTALL_SUCCEEDED) {
                        // 解析成功，执行包以及子包的扫描
                        scanPackageChildLI(parseResult.pkg, parseFlags, scanFlags,
                                currentTime, null);
                    }
                } catch (PackageManagerException e) {
                    errorCode = e.error;
                    Slog.w(TAG, "Failed to scan " + parseResult.scanFile + ": " + e.getMessage());
                }
            } else if (throwable instanceof PackageParser.PackageParserException) {
                PackageParser.PackageParserException e = (PackageParser.PackageParserException)
                        throwable;
                errorCode = e.error;
                Slog.w(TAG, "Failed to parse " + parseResult.scanFile + ": " + e.getMessage());
            } else {
                throw new IllegalStateException("Unexpected exception occurred while parsing "
                        + parseResult.scanFile, throwable);
            }

            ...
        }
    }
}

/**
 *  Scans a package and returns the newly parsed package.
 *  @throws PackageManagerException on a parse error.
 */
private PackageParser.Package scanPackageChildLI(PackageParser.Package pkg,
        final @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
        @Nullable UserHandle user)
                throws PackageManagerException {
    // If the package has children and this is the first dive in the function
    // we scan the package with the SCAN_CHECK_ONLY flag set to see whether all
    // packages (parent and children) would be successfully scanned before the
    // actual scan since scanning mutates internal state and we want to atomically
    // install the package and its children.
    if ((scanFlags & SCAN_CHECK_ONLY) == 0) {
        if (pkg.childPackages != null && pkg.childPackages.size() > 0) {
            scanFlags |= SCAN_CHECK_ONLY;
        }
    } else {
        scanFlags &= ~SCAN_CHECK_ONLY;
    }

    // Scan the parent
    PackageParser.Package scannedPkg = addForInitLI(pkg, parseFlags,
            scanFlags, currentTime, user);

    // Scan the children
    final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
    for (int i = 0; i < childCount; i++) {
        PackageParser.Package childPackage = pkg.childPackages.get(i);
        addForInitLI(childPackage, parseFlags, scanFlags,
                currentTime, user);
    }


    if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
        return scanPackageChildLI(pkg, parseFlags, scanFlags, currentTime, user);
    }

    return scannedPkg;
}
```