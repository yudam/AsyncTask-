# AsyncTask-
Android开发艺术探索阅读笔记之：AsyncTask源码阅读
Time：2017年11月7日
context：
AsyncTask异步任务执行类，在线程池中执行异步任务，返回结果到主线程更新UI，AsyncTask属于轻量级的异步任务类，不适合执行长时间耗时任务，建议使用线程池。
AsyncTask是一个抽象类，使用时需要我们去实现它的子类，默认需要三个参数：public abstract class AsyncTask<Params, Progress, Result> ，其中Params代表传入的参数，Progress代表进度，Result代表返回结果。
AsyncTask执行需要注意的几点：
1，AsyncTask的类必须在主线程中加载，这一点会在后面说明
2，AsyncTask对象必须在主线中创建
3，AsyncTask的execute必须在主线程中执行
4，AsyncTask对象职能执行一次。否则会报异常
接下来是对源码的分析，
  通过调用execute方法来执行异步任务，当然我们也可以通过executeOnExecutor方法，来执行，至于两者区别下边会介绍。
  new LoadAsynvTask("0001").execute();
  进入AsyncTask方法可以看到execute中也是默认调用executeOnExecutor方法，sDefaultExecutor为默认的线程池
   @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
           ...
        }
       
        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
通过这sDefaultExecutor是一种串行线程池，后面会介绍，可以看到调用executeOnExecutor方法后，会首先设置当前状态为运行时，然后执行onPreExecute方法，继续执行线程池，接下来看一下线程池的运行
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;
        //通过该方法，可以看出在4.0以上AsyncTask使用串行执行
        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        /**
                         * 实际调用mFuture的run方法
                         * run方法中又调用mWorker的call方法
                         * 所以线程池最后的执行是在mWorker的call中
                         */
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

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {

            }
        }
    }
    
    在SerialExecutor中系统会首先将Params封装为FutureTask对象mWorker.mParams = params;exec.execute(mFuture);FutureTask是一个并发类，在这里的作用相当于Runnable，SerialExecutor会调用线程池的execute方法，来执行FutureTask，可以看到execute方法时加入了synchronized，从而可以看出SerialExecutor属于串行执行，线程池会首先将FutureTask插入到任务队列mTasks中，若没有正在活动任务则调用scheduleNext获取下一个任务，直到所有任务完成。由于FutureTask的run方法会调用mWorker的call方法，所以后续的执行是在mWorker的call当中。
    
    public AsyncTask(@Nullable Looper callbackLooper) {
        //获取当前主线程的Handler
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                //设为true表示当前应用已经被调用
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //执行AsyncTask的doInBackground方法
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //，调用返回值result，最后执行该方法
                    postResult(result);
                }
                return result;
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
              ...
            }
        };
    }
    首先调用doInBackground方法，获取返回值执行切换线程造作，在这里我们可以看到当前mHandler即为主线程的Handler，这就解释了为什么AsyncTask必须在主线程执行的原因。
    
    private Result postResult(Result result) {

        /**
         * 最后是通过主线程的handler来切换线程发送消息
         * 将结果发送出去,设置标志MESSAGE_POST_RESULT
         */
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
    
    在postResult中通过Handler发送结果到InternalHandler的handleMessage中进行处理，可以看到有两种状态，分别调用的时返回值和加载进度
    
    private static class InternalHandler extends Handler {
        //初始化主线程的Handler
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
                    //处理返回结果
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:

                    //处理加载进度
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

    这里看一下finish方法，比较简单了
    /**
     * 处理返回结果：
     * 如果AsyncTask被取消了，调用onCancelled方法
     * 否则调用onPostExecute返回结果主线程，
     * 设置当前状态，结束，完整流程
     */

    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
    
    
    通过以上扥分析也可以看出AsyncTask的执行顺序：onPreExecute——doInBackground——onProgressUpdate——onPostExecute，若是任务取消的话则会调用
    onCancelled方法。
    
