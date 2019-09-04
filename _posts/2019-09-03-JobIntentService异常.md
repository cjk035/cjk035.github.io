---
layout:     post
title:      JobIntentService中异常
subtitle:   java.lang.SecurityException: Caller no longer running, last stopped +2m10s967ms because: cancelled due to doze
date:       2019-09-03
author:     keier
# header-img: img/post-bg-ios9-web.jpg
catalog: true
# tags:
#     - iOS
#     - ReactiveCocoa
#     - 函数式编程
#     - 开源框架
---



##### 异常如下：

java.lang.RuntimeException: An error occurred while executing doInBackground()
at android.os.AsyncTask$3.done(AsyncTask.java:355)
at java.util.concurrent.FutureTask.finishCompletion(FutureTask.java:383)
at java.util.concurrent.FutureTask.setException(FutureTask.java:252)
at java.util.concurrent.FutureTask.run(FutureTask.java:271)
at java.util.concurrent.ThreadPoolExecutor.processTask(ThreadPoolExecutor.java:1187)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
at java.lang.Thread.run(Thread.java:784)
Caused by: java.lang.SecurityException: Caller no longer running, last stopped +2m10s967ms because: cancelled due to doze
at android.os.Parcel.createException(Parcel.java:1953)
at android.os.Parcel.readException(Parcel.java:1921)
at android.os.Parcel.readException(Parcel.java:1871)
at android.app.job.IJobCallback$Stub$Proxy.completeWork(IJobCallback.java:222)
at android.app.job.JobParameters.completeWork(JobParameters.java:267)
at androidx.core.app.JobIntentService$JobServiceEngineImpl$WrapperWorkItem.complete(JobIntentService.java:3)
at androidx.core.app.JobIntentService$CommandProcessor.doInBackground(JobIntentService.java:4)
at androidx.core.app.JobIntentService$CommandProcessor.doInBackground(JobIntentService.java:1)
at android.os.AsyncTask$2.call(AsyncTask.java:334)
at java.util.concurrent.FutureTask.run(FutureTask.java:266)
... 4 more
Caused by: android.os.RemoteException: Remote stack trace:
at android.app.job.IJobCallback$Stub.onTransact(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z(libmapleframework.so:4940213)
at android.os.Binder.execTransact(IJJI)Z(libmapleframework.so:6087961)

java.lang.SecurityException: Caller no longer running, last stopped +2m10s967ms because: cancelled due to doze
at android.os.Parcel.createException(Parcel.java:1953)
at android.os.Parcel.readException(Parcel.java:1921)
at android.os.Parcel.readException(Parcel.java:1871)
at android.app.job.IJobCallback$Stub$Proxy.completeWork(IJobCallback.java:222)
at android.app.job.JobParameters.completeWork(JobParameters.java:267)
at androidx.core.app.JobIntentService$JobServiceEngineImpl$WrapperWorkItem.complete(JobIntentService.java:3)
at androidx.core.app.JobIntentService$CommandProcessor.doInBackground(JobIntentService.java:4)
at androidx.core.app.JobIntentService$CommandProcessor.doInBackground(JobIntentService.java:1)
at android.os.AsyncTask$2.call(AsyncTask.java:334)
at java.util.concurrent.FutureTask.run(FutureTask.java:266)
at java.util.concurrent.ThreadPoolExecutor.processTask(ThreadPoolExecutor.java:1187)
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
at java.lang.Thread.run(Thread.java:784)
Caused by: android.os.RemoteException: Remote stack trace:
at android.app.job.IJobCallback$Stub.onTransact(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z(libmapleframework.so:4940213)
at android.os.Binder.execTransact(IJJI)Z(libmapleframework.so:6087961)

android.os.RemoteException: Remote stack trace:
at android.app.job.IJobCallback$Stub.onTransact(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z(libmapleframework.so:4940213)
at android.os.Binder.execTransact(IJJI)Z(libmapleframework.so:6087961)



##### 解决方法：

```java
public abstract class MyJobIntentService extends JobIntentService {

    @Override
    GenericWorkItem dequeueWork() {
        try {
            return super.dequeueWork();
        } catch (SecurityException ignored) {
            ignored.printStackTrace();
        }    
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        // override mJobImpl with safe class to ignore SecurityException
        if (Build.VERSION.SDK_INT >= 26) {
            mJobImpl = new SafeJobServiceEngineImpl(this);
        } else {
            mJobImpl = null;
        }
    }
}
```

```java
/**
 * Implementation of a JobServiceEngine for interaction with JobIntentService.
 */
@RequiresApi(26)
public class SafeJobServiceEngineImpl extends JobServiceEngine
        implements JobIntentService.CompatJobEngine {
    static final String TAG = "JobServiceEngineImpl";

    static final boolean DEBUG = false;

    final JobIntentService mService;
    final Object mLock = new Object();
    JobParameters mParams;

    final class WrapperWorkItem implements JobIntentService.GenericWorkItem {
        final JobWorkItem mJobWork;

        WrapperWorkItem(JobWorkItem jobWork) {
            mJobWork = jobWork;
        }

        @Override
        public Intent getIntent() {
            return mJobWork.getIntent();
        }

        @Override
        public void complete() {
            synchronized (mLock) {
                if (mParams != null) {
                    try {
                        mParams.completeWork(mJobWork);
                    } catch (SecurityException se) {
                        // ignore
                        se.printStackTrace();
                    }
                }
            }
        }
    }

    SafeJobServiceEngineImpl(JobIntentService service) {
        super(service);
        mService = service;
    }

    @Override
    public IBinder compatGetBinder() {
        return getBinder();
    }

    @Override
    public boolean onStartJob(JobParameters params) {
        if (DEBUG) Log.d(TAG, "onStartJob: " + params);
        mParams = params;
        // We can now start dequeuing work!
        mService.ensureProcessorRunningLocked(false);
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        if (DEBUG) Log.d(TAG, "onStartJob: " + params);
        boolean result = mService.doStopCurrentWork();
        synchronized (mLock) {
            // Once we return, the job is stopped, so its JobParameters are no
            // longer valid and we should not be doing anything with them.
            mParams = null;
        }
        return result;
    }

    /**
     * Dequeue some work.
     */
    @Override
    public JobIntentService.GenericWorkItem dequeueWork() {
        JobWorkItem work = null;
        synchronized (mLock) {
            if (mParams == null) {
                return null;
            }
            try {
                work = mParams.dequeueWork();
            } catch (SecurityException se) {
                //ignore
                se.printStackTrace();
            }
        }
        if (work != null) {
            work.getIntent().setExtrasClassLoader(mService.getClassLoader());
            return new WrapperWorkItem(work);
        } else {
            return null;
        }
    }
}
```