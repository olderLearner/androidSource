Launcher程序就是我们平时看到的桌面程序，它其实也是一个android应用程序，只不过这个应用程序是系统默认第一个启动的应用程序，
这里我们就简单的分析一下Launcher应用的启动流程。

不同的手机厂商定制android操作系统的时候都会更改Launcher的源代码，我们这里以android23的源码为例大致的分析一下Launcher的启动流程。

通过上一篇文章，我们知道SystemServer进程主要用于启动系统的各种服务，二者其中就包含了负责启动Launcher的服务，LauncherAppService。
具体关于SystenServer的启动流程可以参见：<a href="http://blog.csdn.net/qq_23547831/article/details/51105171"> 
android源码解析之（九）-->SystemServer进程启动流程</a>

在SystemServer进程的启动过程中会调用其main静态方法，开始执行整个SystemServer的启动流程，在其中通过调用三个内部方法分别启动
boot service、core service和other service。在调用startOtherService方法中就会通过调用mActivityManagerService.systemReady()方法，那么我们看一下其具体实现：

```
// We now tell the activity manager it is okay to run third party
        // code.  It will call back into us once it has gotten to the state
        // where third party code can really run (but before it has actually
        // started launching the initial applications), for us to complete our
        // initialization.
        mActivityManagerService.systemReady(new Runnable() {
            @Override
            public void run() {
                Slog.i(TAG, "Making services ready");
                mSystemServiceManager.startBootPhase(
                        SystemService.PHASE_ACTIVITY_MANAGER_READY);
                ...
                try {
                    startSystemUi(context);
                } catch (Throwable e) {
                    reportWtf("starting System UI", e);
                }
                ...
            }
        });
```
可以发现这个方法传递了一个Runnable参数，里面执行了各种其他服务的systemReady方法，这里不是我们关注的重点，我们看一下在
ActivityManagerService中systemReady方法的具体实现，方法体比较长，我就不在这里贴出代码了，主要的逻辑就是做一些
ActivityManagerService的ready操作

```
public void systemReady(final Runnable goingCallback) {
        ...
        // Start up initial activity.
        mBooting = true;
        startHomeActivityLocked(mCurrentUserId, "systemReady");
		...
    }
```
重点是在这个方法体中调用了startHomeActivityLocked方法，看其名字就是说开始执行启动homeActivity的操作，好了，既然如此，
我们再看一下startHomeActivityLocked的具体实现：

```
    boolean startHomeActivityLocked(int userId, String reason) {
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                && mTopAction == null) {
            // We are running in factory test mode, but unable to find
            // the factory test app, so just sit around displaying the
            // error message and don't try to start anything.
            return false;
        }
        Intent intent = getHomeIntent();
        ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
        if (aInfo != null) {
            intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
            // Don't do this if the home app is currently being
            // instrumented.
            aInfo = new ActivityInfo(aInfo);
            aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
            ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                    aInfo.applicationInfo.uid, true);
            if (app == null || app.instr == null) {
                intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
                final int resolvedUserId = UserHandle.getUserId(aInfo.applicationInfo.uid);
                // For ANR debugging to verify if the user activity is the one that actually
                // launched.
                final String myReason = reason + ":" + userId + ":" + resolvedUserId;
                mActivityStarter.startHomeActivityLocked(intent, aInfo, myReason);    <------------ ****
            }
        } else {
            Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
        }

        return true;
    }
```
首先是调用getHomeIntent()方法，看一下getHomeIntent是如何实现构造Intent对象的：

```
    Intent getHomeIntent() {
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
    }
```
可以发现，启动Launcher的Intent对象中添加了Intent.CATEGORY_HOME常量，这个其实是一个launcher的标志，一般系统的启动页面
Activity都会在androidmanifest.xml中配置这个标志。比如我们在github中的android launcher源码中查看其androidmanifest.xml文件：
![这里写图片描述](http://img.blog.csdn.net/20160410171414711)
可以发现其Activity的定义intentfilter中就是定义了这样的category。不同的手机厂商可能会修改Launcher的源码，
但是这个category一般是不会更改的。

继续回到我们的startHomeActivityLocked方法，我们发现经过一系列的判断逻辑之后最后调用了
mActivityStarter.startHomeActivityLocked.
...
    void startHomeActivityLocked(Intent intent, ActivityInfo aInfo, String reason) {
        // 调整launcher的stack
        mSupervisor.moveHomeStackTaskToTop(reason);
        mLastHomeActivityStartResult = startActivityLocked(null /*caller*/, intent,  <-----------
                null /*ephemeralIntent*/, null /*resolvedType*/, aInfo, null /*rInfo*/,
                null /*voiceSession*/, null /*voiceInteractor*/, null /*resultTo*/,
                null /*resultWho*/, 0 /*requestCode*/, 0 /*callingPid*/, 0 /*callingUid*/,
                null /*callingPackage*/, 0 /*realCallingPid*/, 0 /*realCallingUid*/,
                0 /*startFlags*/, null /*options*/, false /*ignoreTargetSecurity*/,
                false /*componentSpecified*/, mLastHomeActivityStartRecord /*outActivity*/,
                null /*inTask*/, "startHomeActivity: " + reason);
        if (mSupervisor.inResumeTopActivity) {                                       <-----------
            // If we are in resume section already, home activity will be initialized, but not
            // resumed (to avoid recursive resume) and will stay that way until something pokes it
            // again. We need to schedule another resume.
            mSupervisor.scheduleResumeTopActivities();                               <--------------
        }
    }
...
       // return : mLastHomeActivityStartResult
-----> int startActivityLocked() {
        ...
        mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask);
        ...
        return mLastStartActivityResult != START_ABORTED ? mLastStartActivityResult : START_SUCCESS;
}
-----> startActivity() {
        int err = ActivityManager.START_SUCCESS;
        ...
        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid);
        }
        ...        
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, options, sourceRecord);
        ...
        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
            options, inTask, outActivity);    
}

    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {
        int result = START_CANCELED;
        try {
            mService.mWindowManager.deferSurfaceLayout();
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,  <-----
                    startFlags, doResume, options, inTask, outActivity);
        } finally {
            // If we are not able to proceed, disassociate the activity from the task. Leaving an
            // activity in an incomplete state can lead to issues, such as performing operations
            // without a window container.
            if (!ActivityManager.isStartResultSuccessful(result)
                    && mStartActivity.getTask() != null) {
                mStartActivity.getTask().removeActivity(mStartActivity);
            }
            mService.mWindowManager.continueSurfaceLayout();
        }

        postStartActivityProcessing(r, result, mSupervisor.getLastStack().mStackId,  mSourceRecord,
                mTargetStack);

        return result;
    }

    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);

        computeLaunchingTaskFlags();

        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);

        ActivityRecord reusedActivity = getReusableIntentActivity();

        final int preferredLaunchStackId =
                (mOptions != null) ? mOptions.getLaunchStackId() : INVALID_STACK_ID;
        final int preferredLaunchDisplayId =
                (mOptions != null) ? mOptions.getLaunchDisplayId() : DEFAULT_DISPLAY;

        if (reusedActivity != null) {
            ...
        }

        if (mStartActivity.packageName == null) {
            ...
            return START_CLASS_NOT_FOUND;
        }

        // If the activity being launched is the same as the one currently at the top, then
        // we need to check if it should only be launched once.
        final ActivityStack topStack = mSupervisor.mFocusedStack;
        final ActivityRecord topFocused = topStack.topActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.realActivity.equals(mStartActivity.realActivity)
                && top.userId == mStartActivity.userId
                && top.app != null && top.app.thread != null
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || mLaunchSingleTop || mLaunchSingleTask);
        if (dontStart) {
            ...
            return START_DELIVERED_TO_TOP;
        }

        boolean newTask = false;
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;

        // Should this be considered a new task?
        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {

        } else if (mSourceRecord != null) {
            result = setTaskFromSourceRecord();
        } else if (mInTask != null) {
            result = setTaskFromInTask();
        } else {
            setTaskToCurrentTopOrCreateNewTask();
        }
        if (result != START_SUCCESS) {
            return result;
        }

        mService.grantUriPermissionFromIntentLocked(mCallingUid, mStartActivity.packageName,
                mIntent, mStartActivity.getUriPermissionsLocked(), mStartActivity.userId);
        mService.grantEphemeralAccessLocked(mStartActivity.userId, mIntent,
                mStartActivity.appInfo.uid, UserHandle.getAppId(mCallingUid));
        if (mSourceRecord != null) {
            mStartActivity.getTask().setTaskToReturnTo(mSourceRecord);
        }
        if (newTask) {
            EventLog.writeEvent(
                    EventLogTags.AM_CREATE_TASK, mStartActivity.userId,
                    mStartActivity.getTask().taskId);
        }
        ActivityStack.logStartActivity(
                EventLogTags.AM_CREATE_ACTIVITY, mStartActivity, mStartActivity.getTask());
        mTargetStack.mLastPausedActivity = null;

        sendPowerHintForLaunchStartIfNeeded(false /* forceSend */, mStartActivity);
        // check this
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition, <-----
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                mTargetStack.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
                mWindowManager.executeAppTransition();
            } else {
                if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else {
            mTargetStack.addRecentActivityLocked(mStartActivity);
        }
        mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);

        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredLaunchStackId,
                preferredLaunchDisplayId, mTargetStack.mStackId);

        return START_SUCCESS;
    }


ActivityStack::startActivityLocked()

    final void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
            boolean newTask, boolean keepCurTransition, ActivityOptions options) {
        TaskRecord rTask = r.getTask();
        final int taskId = rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
            // Last activity in task had been removed or ActivityManagerService is reusing task.
            // Insert or replace.
            // Might not even be in.
            insertTaskAtTop(rTask, r);
        }
        TaskRecord task = null;
        if (!newTask) {
            // If starting in an existing task, find where that is...
            boolean startIt = true;
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    // All activities in task are finishing.
                    continue;
                }
                if (task == rTask) {
                    // Here it is!  Now, if this is not yet visible to the
                    // user, then just add it without starting; it will
                    // get started when the user navigates back to it.
                    if (!startIt) {
                        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                                + task, new RuntimeException("here").fillInStackTrace());
                        r.createWindowContainer();
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }

        // Place a new activity at top of stack, so it is next to interact with the user.

        // If we are not placing the new activity frontmost, we do not want to deliver the
        // onUserLeaving callback to the actual frontmost activity
        final TaskRecord activityTask = r.getTask();
        if (task == activityTask && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
            mStackSupervisor.mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                    "startActivity() behind front, mUserLeaving=false");
        }

        task = activityTask;

        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());
        // TODO: Need to investigate if it is okay for the controller to already be created by the
        // time we get to this point. I think it is, but need to double check.
        // Use test in b/34179495 to trace the call path.
        if (r.getWindowContainerController() == null) {
            r.createWindowContainer();
        }
        task.setFrontOfTask();

        if (mActivityTrigger != null) {
            mActivityTrigger.activityStartTrigger(r.intent, r.info, r.appInfo, r.getTask().mFullscreen);
        }
        if (!isHomeStack() || numActivities() > 0) {
            // We want to show the starting preview window if we are
            // switching to a new task, or the next activity's process is
            // not currently running.
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else {
                int transit = TRANSIT_ACTIVITY_OPEN;
                if (newTask) {
                    if (r.mLaunchTaskBehind) {
                        transit = TRANSIT_TASK_OPEN_BEHIND;
                    } else {
                        // If a new task is being launched, then mark the existing top activity as
                        // supporting picture-in-picture while pausing only if the starting activity
                        // would not be considered an overlay on top of the current activity
                        // (eg. not fullscreen, or the assistant)
                        if (canEnterPipOnTaskSwitch(focusedTopActivity,
                                null /* toFrontTask */, r, options)) {
                            focusedTopActivity.supportsEnterPipOnTaskSwitch = true;
                        }
                        transit = TRANSIT_TASK_OPEN;
                    }
                }
                mWindowManager.prepareAppTransition(transit, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            boolean doShow = true;
            if (newTask) {
                // Even though this activity is starting fresh, we still need
                // to reset it to make sure we apply affinities to move any
                // existing activities from other tasks in to it.
                // If the caller has requested that the target task be
                // reset, then do so.
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && options.getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {
                // Don't do a starting window for mLaunchTaskBehind. More importantly make sure we
                // tell WindowManager that r is visible even though it is at the back of the stack.
                r.setVisibility(true);
                ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                // Figure out if we are transitioning from another activity that is
                // "has the same starting icon" as the next one.  This allows the
                // window manager to keep the previous window it had previously
                // created, if it still had one.
                TaskRecord prevTask = r.getTask();
                ActivityRecord prev = prevTask.topRunningActivityWithStartingWindowLocked();
                if (prev != null) {
                    // We don't want to reuse the previous starting preview if:
                    // (1) The current activity is in a different task.
                    if (prev.getTask() != prevTask) {
                        prev = null;
                    }
                    // (2) The current activity is already displayed.
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                r.showStartingWindow(prev, newTask, isTaskSwitch(r, focusedTopActivity));
            }
        } else {
            // If this is the first activity, don't do any fancy animations,
            // because there is nothing for it to animate on top of.
            ActivityOptions.abort(options);
        }
    }



startHomeActivity方法，然后我们可以查看一下该方法的具体实现逻辑：

```
void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason) {
        moveHomeStackTaskToTop(HOME_ACTIVITY_TYPE, reason);
        startActivityLocked(null /* caller */, intent, null /* resolvedType */, aInfo,
                null /* voiceSession */, null /* voiceInteractor */, null /* resultTo */,
                null /* resultWho */, 0 /* requestCode */, 0 /* callingPid */, 0 /* callingUid */,
                null /* callingPackage */, 0 /* realCallingPid */, 0 /* realCallingUid */,
                0 /* startFlags */, null /* options */, false /* ignoreTargetSecurity */,
                false /* componentSpecified */,
                null /* outActivity */, null /* container */,  null /* inTask */);
        if (inResumeTopActivity) {
            // If we are in resume section already, home activity will be initialized, but not
            // resumed (to avoid recursive resume) and will stay that way until something pokes it
            // again. We need to schedule another resume.
            scheduleResumeTopActivities();
        }
    }
```
发现其调用的是scheduleResumeTopActivities()方法，这个方法其实是关于Activity的启动流程的逻辑了，这里我们不在详细的说明，关于Activity的启动流程我们在下面的文章中会介绍。

因为我们的Launcher启动的Intent是一个隐士的Intent，所以我们会启动在androidmanifest.xml中配置了相同catogory的activity，android M中配置的这个catogory就是LauncherActivity。

LauncherActivity继承与ListActivity，我们看一下其Layout布局文件：

```
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    >

    <ListView
        android:id="@android:id/list"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        />

    <TextView
        android:id="@android:id/empty"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:text="@string/activity_list_empty"
        android:visibility="gone"
        android:textAppearance="?android:attr/textAppearanceMedium"
        />

</FrameLayout>
```
可以看到我们现实的桌面其实就是一个ListView控件，然后看一下其onCreate方法：

```
@Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        
        mPackageManager = getPackageManager();

        if (!mPackageManager.hasSystemFeature(PackageManager.FEATURE_WATCH)) {
            requestWindowFeature(Window.FEATURE_INDETERMINATE_PROGRESS);
            setProgressBarIndeterminateVisibility(true);
        }
        onSetContentView();

        mIconResizer = new IconResizer();
        
        mIntent = new Intent(getTargetIntent());
        mIntent.setComponent(null);
        mAdapter = new ActivityAdapter(mIconResizer);

        setListAdapter(mAdapter);
        getListView().setTextFilterEnabled(true);

        updateAlertTitle();
        updateButtonText();

        if (!mPackageManager.hasSystemFeature(PackageManager.FEATURE_WATCH)) {
            setProgressBarIndeterminateVisibility(false);
        }
    }
```
可以看到在LauncherActivity的onCreate方法中初始化了一个PackageManager，其主要作用就是从中查询出系统所有已经安装的应用列表，应用包名，应用图标等信息。然后将这些信息注入到Adapter中，这样就可以将系统应用图标和名称显示出来了。
在系统的回调方法onListItemClick中

```
@Override
    protected void onListItemClick(ListView l, View v, int position, long id) {
        Intent intent = intentForPosition(position);
        startActivity(intent);
    }
```
这也就是为什么我们点击了某一个应用图标之后可以启动某一项应用的原因了，我们看一下这里的intentForPosition是如何实现的。

```
protected Intent intentForPosition(int position) {
        ActivityAdapter adapter = (ActivityAdapter) mAdapter;
        return adapter.intentForPosition(position);
    }
```
这里又调用了adapter的intentForPosition方法：

```
public Intent intentForPosition(int position) {
            if (mActivitiesList == null) {
                return null;
            }

            Intent intent = new Intent(mIntent);
            ListItem item = mActivitiesList.get(position);
            intent.setClassName(item.packageName, item.className);
            if (item.extras != null) {
                intent.putExtras(item.extras);
            }
            return intent;
        }
```
可以看到由于adapter的每一项中都保存了应用的包名可启动Activity名称，所以这里在初始化Intent的时候，直接将这些信息注入到Intent中，然后调用startActivity，就将这些应用启动了（关于startActivity是如何启动的下面的文章中我将介绍）。

总结：

Launcher的启动流程

- Zygote进程 --> SystemServer进程 --> startOtherService方法 --> ActivityManagerService的systemReady方法 --> startHomeActivityLocked方法 --> ActivityStackSupervisor的startHomeActivity方法 --> 执行Activity的启动逻辑，执行scheduleResumeTopActivities()方法。。。。

- 因为是隐士的启动Activity，所以启动的Activity就是在AndroidManifest.xml中配置catogery的值为：

```
public static final String CATEGORY_HOME = "android.intent.category.HOME";
```
可以发现android M中在androidManifest.xml中配置了这个catogory的activity是LauncherActivity，所以我们就可以将这个Launcher启动起来了

- LauncherActivity中是以ListView来显示我们的应用图标列表的，并且为每个Item保存了应用的包名和启动Activity类名，这样点击某一项应用图标的时候就可以根据应用包名和启动Activity名称启动我们的App了。

另外对android源码解析方法感兴趣的可参考我的：
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50634435"> android源码解析之（一）-->android项目构建过程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50751687">android源码解析之（二）-->异步消息机制</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50803849">android源码解析之（三）-->异步任务AsyncTask</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50936584">android源码解析之（四）-->HandlerThread</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50958757">android源码解析之（五）-->IntentService</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50963006">android源码解析之（六）-->Log</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/50971968">android源码解析之（七）-->LruCache</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51104873">android源码解析之（八）-->Zygote进程启动流程</a>
<br><a href="http://blog.csdn.net/qq_23547831/article/details/51105171">android源码解析之（九）-->SystemServer进程启动流程</a>
