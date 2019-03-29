---
layout: post
title: How to use AsyncTask and How it works
author: liuzhimi
---

-----
Android应用在主线程中执行耗时任务时会导致ANR（Application Not Responing），因此使用子线程异步执行耗时任务是必要的。Android SDK中给我们提供了AsyncTask这一封装好的组件。
让我们来了解一下它。

### AsyncTask的使用
以下代码是使用AsyncTask从网络异步加载一张图片的流程。
```
public class NetCacheUtil {

    private static NetCacheUtil instance;

    public static NetCacheUtil getInstance(){
        if (instance == null){
            synchronized (NetCacheUtil.class){
                if (instance == null)
                    instance = new NetCacheUtil();
            }
        }
        return instance;
    }

    public void getBitmapFromNet(String url, ImageView iv){
        MyAsyncTask asyncTask = new MyAsyncTask();
        asyncTask.execute(iv, url);
    }

    private class MyAsyncTask extends AsyncTask<Object, Void, Bitmap> {

        private ImageView ivPic;
        private String url;

        // 耗时任务执行之前 --主线程
        @Override
        protected void onPreExecute() {
            super.onPreExecute();
        }

        // 后台执行的任务
        @Override
        protected Bitmap doInBackground(Object... params) {
            // 执行异步任务的时候，将URL传过来
            ivPic = (ImageView) params[0];
            url = (String) params[1];
            Bitmap bitmap = downloadBitmap(url);
            // 为了保证ImageView控件和URL一一对应，给ImageView设定一个标记
            ivPic.setTag(url);// 关联ivPic和URL

            return bitmap;
        }

        // 更新进度 --主线程
        @Override
        protected void onProgressUpdate(Void... values) {
            super.onProgressUpdate(values);
        }

        // 耗时任务执行之后--主线程
        @Override
        protected void onPostExecute(Bitmap result) {
            String mCurrentUrl = (String) ivPic.getTag();
            if (url.equals(mCurrentUrl)) {
                ivPic.setImageBitmap(result);
                System.out.println("从网络获取图片");
                // 从网络加载完之后，将图片保存到本地SD卡一份，保存到内存中一份
                localCacheUtil.getInstance().saveBitmapToLocal(url, result);
                // 从网络加载完之后，将图片保存到本地SD卡一份，保存到内存中一份
                memoryCacheUtil.getInstance().saveToMemory(url, result);
            }
        }
    }

    private Bitmap downloadBitmap(String url) {
        HttpURLConnection conn = null;
        try {
            URL mURL = new URL(url);
            // 打开HttpURLConnection连接
            conn = (HttpURLConnection) mURL.openConnection();
            // 设置参数
            conn.setConnectTimeout(5000);
            conn.setReadTimeout(5000);
            conn.setRequestMethod("GET");
            // 开启连接
            conn.connect();

            // 获得响应码
            int code = conn.getResponseCode();
            if (code == 200) {
                // 相应成功,获得网络返回来的输入流
                InputStream is = conn.getInputStream();

                // 图片的输入流获取成功之后，设置图片的压缩参数,将图片进行压缩
                BitmapFactory.Options options = new BitmapFactory.Options();
                options.inSampleSize = 2;// 将图片的宽高都压缩为原来的一半,在开发中此参数需要根据图片展示的大小来确定,否则可能展示的不正常
                options.inPreferredConfig = Bitmap.Config.RGB_565;// 这个压缩的最小

                // Bitmap bitmap = BitmapFactory.decodeStream(is);
                Bitmap bitmap = BitmapFactory.decodeStream(is, null, options);// 经过压缩的图片

                return bitmap;
            }

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 断开连接
            conn.disconnect();
        }

        return null;
    }

}
```
要使用AsyncTask我们需要继承它，实现内部的抽象方法。
三个泛型，分别对应着传入参数类型，执行进度的类型，返回结果的类型。
-onPreExecute()：耗时任务执行之前执行此方法（主线程）。
-doInBackground()：耗时任务在此方法中执行（子线程）。
-onProgressUpdate()：更新进度（主线程）。
-onPostExecute()：耗时任务执行完成之后执行此方法（主线程）。

### 源码分析

先看一下成员变量：
```
    private static final String LOG_TAG = "AsyncTask";

    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    /**
     * 线程池核心线程数，最少2个最多4个。
     */
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    /**
     * 线程池最大线程数
     */
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    /**
     * 线程最大闲置时间
     */    
    private static final int KEEP_ALIVE_SECONDS = 30;
    /**
     * 线程工厂，用于创建新线程并计数。
     */
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
    /**
     * 容量为128的有序阻塞队列
     */
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     * 用来执行任务的线程池
     */
    public static final Executor THREAD_POOL_EXECUTOR;
    /**
     * 实例化执行任务的线程池
     */
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

    /**
     * 排队任务的线程池
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    /**
     * 发送给Handler的数据
     */
    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;
    /**
     * 默认线程池设为排队任务线程池
     */
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    /**
     * InternalHandler为内部类，sHandler为主线程的Handler。
     */
    private static InternalHandler sHandler;
    /**
     * 一个抽象内部类，用来执行任务
     */
    private final WorkerRunnable<Params, Result> mWorker;
    /**
     * 处理结果任务
     */
    private final FutureTask<Result> mFuture;
    /**
     * AsyncTask状态
     */
    private volatile Status mStatus = Status.PENDING;
    /**
     * 当前任务是否被取消
     */    
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    /**
     * 当前任务是否被调用
     */
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();

    private final Handler mHandler;


```

了解它们各自的作用后，我们再看看各个内部类，以及它们如何实现：
```
    /*
     * 排队任务线程池
     */
    private static class SerialExecutor implements Executor {
        /*
         * 双端队列，用来存放任务
         */
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        /*
         * 当前正在执行的任务
         */
        Runnable mActive;
        /*
         * 调用此任务时，将任务添加到mTask中
         */
        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }
         /*
          * 执行双端队列中的下一个任务
          */
        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

    /**
     * 枚举类型，指示AsyncTask状态
     */
    public enum Status {
        /**
         * Indicates that the task has not been executed yet.
         * 表示task还未被执行
         */
        PENDING,
        /**
         * Indicates that the task is running.
         * 表示task正在执行
         */
        RUNNING,
        /**
         * Indicates that {@link AsyncTask#onPostExecute} has finished.
         * task运行完成
         */
        FINISHED,
    }

    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }

    private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
```
在了解了AsyncTask的成员变量及内部类后我们就来看看一个AsyncTask实例化时都做了些什么：
```
    public AsyncTask(@Nullable Looper callbackLooper) {
        /*
         * 构造函数不传参数的话默认用 getMainHandler()方法获取Handler
         * 该方法的内部实现见之后第一个代码块
         */
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
        
        //实例化
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);//该任务被调用，设置成true
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //执行耗时任务，并返回结果
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    //出现异常将该任务被取消，设置成true
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //调用postResult方法，该方法内部实现见之后第二个代码块
                    postResult(result);
                }
                return result;
            }
        };

        //实例化
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    //调用postResultIfNotInvoked方法，传入get方法返回值-mWorker的call方法返回值
                    //postResultIfNotInvoked方法内部实现见之后第三个代码块
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
在调用postResultIfNotInvoked方法之后，最终会调用postResult方法，将result发送给handler进行处理。
```
    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                //利用主线程的Looper创建Handler（也是主线程）
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;
        }
    }
```
这部分代码不懂的话可以看看Handler内部的实现，建议看看其它博客，后续我自己也会补上对Handler机制的分析。
```
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();//将message发送给mHandler
        return result;
    }
```
postResult方法的作用就是获取一个msg.what为MESSAGE_POST_RESULT，msg.obj为AsyncTaskResult实例的message，然后将该message传给mHandler处理，至于怎么处理可以网上翻阅到内部类InternalHandler的实现。

```
    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }
```
获取到当前任务是否被调用，只有当前任务没有被调用时，才会调用postResult方法。

构造函数执行完成。我们再来看看调用execute方法会发生什么：
```
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
调用了executeOnExecutor方法，参数为等待任务线程池，以及传入参数params。
```
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
1.首先判断当前task的状态，如果状态是 运行中 或者 完成 都会报错。这也是execute只能调用一次的主要原因。
2.判断通过后，将状态设置为运行中。
3.调用onPreExecute方法
4.将传入参数params传给mWorker
5.调用线程池的execute方法并传入mFuture。
6.执行完成

我们看看该线程池的execute方法都干了什么：
```
 /*
     * 排队任务线程池
     */
    private static class SerialExecutor implements Executor {
        /*
         * 双端队列，用来存放任务
         */
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        /*
         * 当前正在执行的任务
         */
        Runnable mActive;
        /*
         * 调用此任务时，将任务添加到mTask中
         */
        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }
         /*
          * 执行双端队列中的下一个任务
          */
        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
执行任务，将执行流程可见mWorker和mFuture的实例化（构造函数内）。执行完成后会发送给mHandler调用finish方法或者onProgressUpdate方法。
finish方法：
```
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
有参onCancelled方法会调用无参的onCancelled方法，该方法可以重写。
如果没有被取消的话则调用onPostExecute方法
最后将状态设置为完成。




第一次认真写博客，不够好的地方望包涵，同时也欢迎提出您的意见、问题。
