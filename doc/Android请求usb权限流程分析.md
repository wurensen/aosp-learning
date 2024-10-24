# Android请求usb权限流程分析

> 本文基于Android 13（33）源码进行分析。
>
> 如何发现`UsbDevice`不在此介绍。

以下代码为如何请求权限的业务层调用：


```kotlin
private suspend fun requestPermission(usbDevice: UsbDevice): Boolean {
    if (usbManager.hasPermission(usbDevice)) {
        MTSeineLog.d(TAG, "requestUsbPermission: permission granted")
        return true
    }
    // 权限超时：防止已经弹权限，但是页面做了自动跳转导致权限弹窗被盖住，用户无法操作导致一直loading的问题，做个超时机制兜底
    return withTimeoutOrNull(15_000) {
        suspendCancellableCoroutine {
            val permissionAction = "${context.packageName}.USB_PERMISSION"
            MTSeineLog.d(TAG, "start requestUsbPermission: $permissionAction")
            val usbReceiver: BroadcastReceiver = object : BroadcastReceiver() {
                override fun onReceive(context: Context, intent: Intent) {
                    val action = intent.action
                    MTSeineLog.d(TAG, "onReceive: action=$action")
                    if (action == permissionAction) {
                        val granted =
                            intent.getBooleanExtra(UsbManager.EXTRA_PERMISSION_GRANTED, false)
                        MTSeineLog.d(TAG, "requestUsbPermission: granted=$granted")
                        if (it.isActive) {
                            it.resume(granted)
                        }
                    } else {
                        if (it.isActive) {
                            it.resume(false)
                        }
                    }
                    context.unregisterReceiver(this)
                }
            }
            val filter = IntentFilter(permissionAction)
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                context.registerReceiver(
                    usbReceiver,
                    filter,
                    Context.RECEIVER_EXPORTED
                )
            } else {
                context.registerReceiver(usbReceiver, filter)
            }
            // 适配12
            val flags = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
                // 需要设置成mutable，广播中得到的intent才能获取到结果
                PendingIntent.FLAG_MUTABLE
            } else {
                0
            }
            val permissionIntent =
                PendingIntent.getBroadcast(context, 0, Intent(permissionAction), flags)
            usbManager.requestPermission(usbDevice, permissionIntent)
            it.invokeOnCancellation {
                MTSeineLog.w(TAG, "requestPermission: invokeOnCancellation")
                context.unregisterReceiver(usbReceiver)
            }
        }
    } ?: kotlin.run {
        MTSeineLog.w(TAG, "requestPermission: timeout")
        false
    }
}
```

`Android`系统usb权限获取设计成了通过`PendingIntent`结合广播通知的方式，来进行权限结果通知。

## PendingIntent.getBroadcast分析

```java
public static PendingIntent getBroadcast(Context context, int requestCode,
        @NonNull Intent intent, @Flags int flags) {
    return getBroadcastAsUser(context, requestCode, intent, flags, context.getUser());
}

/**
 * @hide
 * Note that UserHandle.CURRENT will be interpreted at the time the
 * broadcast is sent, not when the pending intent is created.
 */
@UnsupportedAppUsage
public static PendingIntent getBroadcastAsUser(Context context, int requestCode,
        Intent intent, int flags, UserHandle userHandle) {
    String packageName = context.getPackageName();
    String resolvedType = intent.resolveTypeIfNeeded(context.getContentResolver());
    checkFlags(flags, packageName);
    try {
        intent.prepareToLeaveProcess(context);
        // 调用AMS获取target
        IIntentSender target =
            ActivityManager.getService().getIntentSenderWithFeature(
                INTENT_SENDER_BROADCAST, packageName,
                context.getAttributionTag(), null, null, requestCode, new Intent[] { intent },
                resolvedType != null ? new String[] { resolvedType } : null,
                flags, null, userHandle.getIdentifier());
        // 封装target返回PendingIntent
        return target != null ? new PendingIntent(target) : null;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

`ActivityManagerService.getIntentSenderWithFeatureAsApp`：

```java
public IIntentSender getIntentSenderWithFeatureAsApp(int type, String packageName,
        String featureId, IBinder token, String resultWho, int requestCode, Intent[] intents,
        String[] resolvedTypes, int flags, Bundle bOptions, int userId, int owningUid) {
    //...
    try {
        //...
        if (type == ActivityManager.INTENT_SENDER_ACTIVITY_RESULT) {
            return mAtmInternal.getIntentSender(type, packageName, featureId, owningUid,
                    userId, token, resultWho, requestCode, intents, resolvedTypes, flags,
                    bOptions);
        }
        return mPendingIntentController.getIntentSender(type, packageName, featureId,
                owningUid, userId, token, resultWho, requestCode, intents, resolvedTypes,
                flags, bOptions);
    } catch (RemoteException e) {
        throw new SecurityException(e);
    }
}
```

`PendingIntentController.getIntentSender`：

```java
public PendingIntentRecord getIntentSender(int type, String packageName,
        @Nullable String featureId, int callingUid, int userId, IBinder token, String resultWho,
        int requestCode, Intent[] intents, String[] resolvedTypes, int flags, Bundle bOptions) {
    synchronized (mLock) {
        //...

        // 判断是否设置了相关标记
        final boolean noCreate = (flags & PendingIntent.FLAG_NO_CREATE) != 0;
        final boolean cancelCurrent = (flags & PendingIntent.FLAG_CANCEL_CURRENT) != 0;
        final boolean updateCurrent = (flags & PendingIntent.FLAG_UPDATE_CURRENT) != 0;
        flags &= ~(PendingIntent.FLAG_NO_CREATE | PendingIntent.FLAG_CANCEL_CURRENT
                | PendingIntent.FLAG_UPDATE_CURRENT);

        // 根据参数创建PendingIntentRecord.Key
        PendingIntentRecord.Key key = new PendingIntentRecord.Key(type, packageName, featureId,
                token, resultWho, requestCode, intents, resolvedTypes, flags,
                SafeActivityOptions.fromBundle(bOptions), userId);
        WeakReference<PendingIntentRecord> ref;
        ref = mIntentSenderRecords.get(key);
        PendingIntentRecord rec = ref != null ? ref.get() : null;
        // 如果已有对应的记录
        if (rec != null) {
            // 如果未设置FLAG_CANCEL_CURRENT标记，执行更新
            if (!cancelCurrent) {
                // 如果设置了FLAG_UPDATE_CURRENT
                if (updateCurrent) {
                    if (rec.key.requestIntent != null) {
                        // 仅更新extras数据
                        rec.key.requestIntent.replaceExtras(intents != null ?
                                intents[intents.length - 1] : null);
                    }
                    if (intents != null) {
                        intents[intents.length - 1] = rec.key.requestIntent;
                        rec.key.allIntents = intents;
                        rec.key.allResolvedTypes = resolvedTypes;
                    } else {
                        rec.key.allIntents = null;
                        rec.key.allResolvedTypes = null;
                    }
                }
                return rec;
            }
            makeIntentSenderCanceled(rec);
            mIntentSenderRecords.remove(key);
            decrementUidStatLocked(rec);
        }
        if (noCreate) {
            return rec;
        }
        // 创建PendingIntentRecord实例
        rec = new PendingIntentRecord(this, key, callingUid);
        // 保存到集合中
        mIntentSenderRecords.put(key, rec.ref);
        incrementUidStatLocked(rec);
        return rec;
    }
}
```

## UsbManager.requestPermission分析

先看下`usbManager.requestPermission`调用的流程：

```java
@RequiresFeature(PackageManager.FEATURE_USB_HOST)
public void requestPermission(UsbDevice device, PendingIntent pi) {
    try {
        // mService为com.android.server.usb.UsbService
        mService.requestDevicePermission(device, mContext.getPackageName(), pi);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```
跨进程调用`usbService.requestDevicePermission`：

```java
@Override
public void requestDevicePermission(UsbDevice device, String packageName, PendingIntent pi) {
    final int uid = Binder.getCallingUid();
    final int pid = Binder.getCallingPid();
    final int userId = UserHandle.getUserId(uid);

    final long token = Binder.clearCallingIdentity();
    try {
        // UsbUserPermissionManager
        getPermissionsForUser(userId).requestPermission(device, packageName, pi, pid, uid);
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
```

调用`UsbUserPermissionManager.requestPermission`：

```java
public void requestPermission(UsbDevice device, String packageName, PendingIntent pi, int pid,
        int uid) {
    Intent intent = new Intent();

    // 如果有权限，立马返回
    // respond immediately if permission has already been granted
    if (hasPermission(device, packageName, pid, uid)) {
        // 返回deivce
        intent.putExtra(UsbManager.EXTRA_DEVICE, device);
        // 返回结果标记权限为true
        intent.putExtra(UsbManager.EXTRA_PERMISSION_GRANTED, true);
        try {
            pi.send(mContext, 0, intent);
        } catch (PendingIntent.CanceledException e) {
            if (DEBUG) Slog.d(TAG, "requestPermission PendingIntent was cancelled");
        }
        return;
    }
    // ...省略

    // 请求权限弹窗
    requestPermissionDialog(device, null,
            mUsbUserSettingsManager.canBeDefault(device, packageName), packageName, pi, uid);
}

/**
 * Creates UI dialog to request permission for the given package to access the device
 * or accessory.
 *
 * @param device       The USB device attached
 * @param accessory    The USB accessory attached
 * @param canBeDefault Whether the calling pacakge can set as default handler
 *                     of the USB device or accessory
 * @param packageName  The package name of the calling package
 * @param uid          The uid of the calling package
 * @param userContext  The context to start the UI dialog
 * @param pi           PendingIntent for returning result
 */
void requestPermissionDialog(@Nullable UsbDevice device,
        @Nullable UsbAccessory accessory,
        boolean canBeDefault,
        @NonNull String packageName,
        int uid,
        @NonNull Context userContext,
        @NonNull PendingIntent pi) {
    final long identity = Binder.clearCallingIdentity();
    try {
        Intent intent = new Intent();
        if (device != null) {
            // 设置device
            intent.putExtra(UsbManager.EXTRA_DEVICE, device);
        } else {
            intent.putExtra(UsbManager.EXTRA_ACCESSORY, accessory);
        }
        // 设置PendingIntent
        intent.putExtra(Intent.EXTRA_INTENT, pi);
        intent.putExtra(Intent.EXTRA_UID, uid);
        intent.putExtra(UsbManager.EXTRA_CAN_BE_DEFAULT, canBeDefault);
        intent.putExtra(UsbManager.EXTRA_PACKAGE, packageName);
        intent.setComponent(
                ComponentName.unflattenFromString(userContext.getResources().getString(
                        com.android.internal.R.string.config_usbPermissionActivity)));
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
		// 系统权限弹窗是个弹窗Activity
        userContext.startActivityAsUser(intent, mUser);
    } catch (ActivityNotFoundException e) {
        Slog.e(TAG, "unable to start UsbPermissionActivity");
    } finally {
        Binder.restoreCallingIdentity(identity);
    }
}
```

根据搜索`config_usbPermissionActivity`找到对应值为`com.android.systemui/com.android.systemui.usb.UsbPermissionActivity`

```java
/**
 * Dialog shown when a package requests access to a USB device or accessory.
 */
public class UsbPermissionActivity extends UsbDialogActivity {

    private boolean mPermissionGranted = false;
    private UsbAudioWarningDialogMessage mUsbPermissionMessageHandler;

    @Inject
    public UsbPermissionActivity(UsbAudioWarningDialogMessage usbAudioWarningDialogMessage) {
        mUsbPermissionMessageHandler = usbAudioWarningDialogMessage;
    }

    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        mUsbPermissionMessageHandler.init(UsbAudioWarningDialogMessage.TYPE_PERMISSION,
                mDialogHelper);
    }

    @Override
    protected void onResume() {
        super.onResume();
        final boolean useRecordWarning = mDialogHelper.isUsbDevice()
                && (mDialogHelper.deviceHasAudioCapture()
                && !mDialogHelper.packageHasAudioRecordingPermission());

        final int titleId = mUsbPermissionMessageHandler.getPromptTitleId();
        final String title = getString(titleId, mDialogHelper.getAppName(),
                mDialogHelper.getDeviceDescription());
        final int messageId = mUsbPermissionMessageHandler.getMessageId();
        String message = (messageId != Resources.ID_NULL)
                ? getString(messageId, mDialogHelper.getAppName(),
                mDialogHelper.getDeviceDescription()) : null;
        setAlertParams(title, message);

        // Only show the "always use" checkbox if there is no USB/Record warning
        if (!useRecordWarning && mDialogHelper.canBeDefault()) {
            addAlwaysUseCheckbox();
        }
        setupAlert();
    }

    @Override
    protected void onPause() {
        if (isFinishing()) {
            // 返回权限授权结果
            mDialogHelper.sendPermissionDialogResponse(mPermissionGranted);
        }
        super.onPause();
    }

    @Override
    void onConfirm() {
        // 同意权限处理
        mDialogHelper.grantUidAccessPermission();
        if (isAlwaysUseChecked()) {
            mDialogHelper.setDefaultPackage();
        }
        mPermissionGranted = true;
        finish();
    }
}
```

通过`mDialogHelper.grantUidAccessPermission()->UsbService.grantDevicePermission()->UsbUserPermissionManager.grantDevicePermission()`调用流程，记录下已授权的usb设备：

```java
/**
 * Grants permission for USB device without showing system dialog for package with uid.
 *
 * @param device to grant permission for
 * @param uid to grant permission for
 */
void grantDevicePermission(@NonNull UsbDevice device, int uid) {
    synchronized (mLock) {
        String deviceName = device.getDeviceName();
        SparseBooleanArray uidList = mDevicePermissionMap.get(deviceName);
        if (uidList == null) {
            uidList = new SparseBooleanArray(1);
            mDevicePermissionMap.put(deviceName, uidList);
        }
        uidList.put(uid, true);
    }
}
```

通过`mDialogHelper.sendPermissionDialogResponse(mPermissionGranted)`发送授权结果：

```java
/**
 * Sends the result of the permission dialog via the provided PendingIntent.
 *
 * @param permissionGranted True if the user pressed ok in the permission dialog.
 */
public void sendPermissionDialogResponse(boolean permissionGranted) {
    if (!mResponseSent) {
        // send response via pending intent
        Intent intent = new Intent();
        if (mIsUsbDevice) {
            // 返回device
            intent.putExtra(UsbManager.EXTRA_DEVICE, mDevice);
        } else {
            intent.putExtra(UsbManager.EXTRA_ACCESSORY, mAccessory);
        }
        // 返回权限结果
        intent.putExtra(UsbManager.EXTRA_PERMISSION_GRANTED, permissionGranted);
        try {
            // 执行发送
            mPendingIntent.send(mContext, 0, intent);
            mResponseSent = true;
        } catch (PendingIntent.CanceledException e) {
            Log.w(TAG, "PendingIntent was cancelled");
        }
    }
}
```

执行`mPendingIntent.send(mContext, 0, intent)`：

```java
public int sendAndReturnResult(Context context, int code, @Nullable Intent intent,
        @Nullable OnFinished onFinished, @Nullable Handler handler,
        @Nullable String requiredPermission, @Nullable Bundle options)
        throws CanceledException {
    try {
        //...
        // 调用AMS的sendIntentSender
        return ActivityManager.getService().sendIntentSender(
                mTarget, mWhitelistToken, code, intent, resolvedType,
                onFinished != null
                        ? new FinishedDispatcher(this, onFinished, handler)
                        : null,
                requiredPermission, options);
    } catch (RemoteException e) {
        throw new CanceledException(e);
    }
}
```

调用到`ActivityManagerService.sendIntentSender`：

```java
@Override
public int sendIntentSender(IIntentSender target, IBinder allowlistToken, int code,
        Intent intent, String resolvedType,
        IIntentReceiver finishedReceiver, String requiredPermission, Bundle options) {
    // 根据PendingIntent.getBroadcast调用分析，可以确定target为PendingIntentRecord
    if (target instanceof PendingIntentRecord) {
        return ((PendingIntentRecord)target).sendWithResult(code, intent, resolvedType,
                allowlistToken, finishedReceiver, requiredPermission, options);
    } else {
        // ...
        return 0;
    }
}
```

最终调用到`PendingIntentRecord.sendInner`：

```java
public int sendInner(int code, Intent intent, String resolvedType, IBinder allowlistToken,
        IIntentReceiver finishedReceiver, String requiredPermission, IBinder resultTo,
        String resultWho, int requestCode, int flagsMask, int flagsValues, Bundle options) {
    if (intent != null) intent.setDefusable(true);
    if (options != null) options.setDefusable(true);

    TempAllowListDuration duration = null;
    Intent finalIntent = null;
    Intent[] allIntents = null;
    String[] allResolvedTypes = null;
    SafeActivityOptions mergedOptions = null;
    synchronized (controller.mLock) {
        if (canceled) {
            return ActivityManager.START_CANCELED;
        }

        sent = true;
        if ((key.flags & PendingIntent.FLAG_ONE_SHOT) != 0) {
            controller.cancelIntentSender(this, true);
        }

        finalIntent = key.requestIntent != null ? new Intent(key.requestIntent) : new Intent();

        // 如果设置了不可变，那不执行填充intent信息到finalIntent中，对于需要通过intent返回结果的方式来说就获取不到结果了
        final boolean immutable = (key.flags & PendingIntent.FLAG_IMMUTABLE) != 0;
        if (!immutable) {
            if (intent != null) {
                // 填充intent信息到finalIntent中
                int changes = finalIntent.fillIn(intent, key.flags);
                if ((changes & Intent.FILL_IN_DATA) == 0) {
                    resolvedType = key.requestResolvedType;
                }
            } else {
                resolvedType = key.requestResolvedType;
            }
            flagsMask &= ~Intent.IMMUTABLE_FLAGS;
            flagsValues &= flagsMask;
            finalIntent.setFlags((finalIntent.getFlags() & ~flagsMask) | flagsValues);
        } else {
            resolvedType = key.requestResolvedType;
        }

        //...

        if (key.type == ActivityManager.INTENT_SENDER_ACTIVITY
                && key.allIntents != null && key.allIntents.length > 1) {
            // Copy all intents and resolved types while we have the controller lock so we can
            // use it later when the lock isn't held.
            allIntents = new Intent[key.allIntents.length];
            allResolvedTypes = new String[key.allIntents.length];
            System.arraycopy(key.allIntents, 0, allIntents, 0, key.allIntents.length);
            if (key.allResolvedTypes != null) {
                System.arraycopy(key.allResolvedTypes, 0, allResolvedTypes, 0,
                        key.allResolvedTypes.length);
            }
            allIntents[allIntents.length - 1] = finalIntent;
            allResolvedTypes[allResolvedTypes.length - 1] = resolvedType;
        }

    }
    // We don't hold the controller lock beyond this point as we will be calling into AM and WM.

    final int callingUid = Binder.getCallingUid();
    final int callingPid = Binder.getCallingPid();
    final long origId = Binder.clearCallingIdentity();

    int res = START_SUCCESS;
    try {
        //...

        boolean sendFinish = finishedReceiver != null;
        int userId = key.userId;
        if (userId == UserHandle.USER_CURRENT) {
            userId = controller.mUserController.getCurrentOrTargetUserId();
        }
        // temporarily allow receivers and services to open activities from background if the
        // PendingIntent.send() caller was foreground at the time of sendInner() call
        final boolean allowTrampoline = uid != callingUid
                && controller.mAtmInternal.isUidForeground(callingUid)
                && isPendingIntentBalAllowedByCaller(options);

        // note: we on purpose don't pass in the information about the PendingIntent's creator,
        // like pid or ProcessRecord, to the ActivityTaskManagerInternal calls below, because
        // it's not unusual for the creator's process to not be alive at this time
        switch (key.type) {
            case ActivityManager.INTENT_SENDER_ACTIVITY:
                // 打开activity处理
                try {
                    // Note when someone has a pending intent, even from different
                    // users, then there's no need to ensure the calling user matches
                    // the target user, so validateIncomingUser is always false below.

                    if (key.allIntents != null && key.allIntents.length > 1) {
                        res = controller.mAtmInternal.startActivitiesInPackage(
                                uid, callingPid, callingUid, key.packageName, key.featureId,
                                allIntents, allResolvedTypes, resultTo, mergedOptions, userId,
                                false /* validateIncomingUser */,
                                this /* originatingPendingIntent */,
                                mAllowBgActivityStartsForActivitySender.contains(
                                        allowlistToken));
                    } else {
                        res = controller.mAtmInternal.startActivityInPackage(uid, callingPid,
                                callingUid, key.packageName, key.featureId, finalIntent,
                                resolvedType, resultTo, resultWho, requestCode, 0,
                                mergedOptions, userId, null, "PendingIntentRecord",
                                false /* validateIncomingUser */,
                                this /* originatingPendingIntent */,
                                mAllowBgActivityStartsForActivitySender.contains(
                                        allowlistToken));
                    }
                } catch (RuntimeException e) {
                    Slog.w(TAG, "Unable to send startActivity intent", e);
                }
                break;
            case ActivityManager.INTENT_SENDER_ACTIVITY_RESULT:
                controller.mAtmInternal.sendActivityResult(-1, key.activity, key.who,
                            key.requestCode, code, finalIntent);
                break;
            case ActivityManager.INTENT_SENDER_BROADCAST:
                // 发送广播处理
                try {
                    final boolean allowedByToken =
                            mAllowBgActivityStartsForBroadcastSender.contains(allowlistToken);
                    final IBinder bgStartsToken = (allowedByToken) ? allowlistToken : null;

                    // 最终调用到AMS的broadcastIntentInPackage发送广播
                    // If a completion callback has been requested, require
                    // that the broadcast be delivered synchronously
                    int sent = controller.mAmInternal.broadcastIntentInPackage(key.packageName,
                            key.featureId, uid, callingUid, callingPid, finalIntent,
                            resolvedType, finishedReceiver, code, null, null,
                            requiredPermission, options, (finishedReceiver != null), false,
                            userId, allowedByToken || allowTrampoline, bgStartsToken,
                            null /* broadcastAllowList */);
                    if (sent == ActivityManager.BROADCAST_SUCCESS) {
                        sendFinish = false;
                    }
                } catch (RuntimeException e) {
                    Slog.w(TAG, "Unable to send startActivity intent", e);
                }
                break;
            case ActivityManager.INTENT_SENDER_SERVICE:
            case ActivityManager.INTENT_SENDER_FOREGROUND_SERVICE:
                try {
                    final boolean allowedByToken =
                            mAllowBgActivityStartsForServiceSender.contains(allowlistToken);
                    final IBinder bgStartsToken = (allowedByToken) ? allowlistToken : null;

                    controller.mAmInternal.startServiceInPackage(uid, finalIntent, resolvedType,
                            key.type == ActivityManager.INTENT_SENDER_FOREGROUND_SERVICE,
                            key.packageName, key.featureId, userId,
                            allowedByToken || allowTrampoline, bgStartsToken);
                } catch (RuntimeException e) {
                    Slog.w(TAG, "Unable to send startService intent", e);
                } catch (TransactionTooLargeException e) {
                    res = ActivityManager.START_CANCELED;
                }
                break;
        }

        if (sendFinish && res != ActivityManager.START_CANCELED) {
            try {
                finishedReceiver.performReceive(new Intent(finalIntent), 0,
                        null, null, false, false, key.userId);
            } catch (RemoteException e) {
            }
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }

    return res;
}
```

## 总结

`Android`系统usb权限获取设计成了通过`PendingIntent`结合广播通知的方式，来进行权限结果通知，要在广播通知中正确的获取到结果，就需要在创建`PendingIntent`的时候设置`PendingIntent.FLAG_MUTABLE`标记，以便权限结果能通过广播的`Intent`返回。

除了usb权限获取的方式，通过`PendingIntent`结合广播通知的方式，机制都是一致的，仅是细节上的差异，需要结合场景分析。