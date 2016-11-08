#Android AsyncTask原理解析

##重要成员变量
###两个线程池：

- SerialExecutor  SERIAL_EXECUTOR：这个变量引用仅仅用于实例化一个线性线程池和赋值给sDefaultExecutor，在类中其他地方并没有用到

```
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```

- Executor sDefaultExecutor：用于任务排队的线程池

```
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

- Executor THREAD_POOL_EXECUTOR：执行实际操作的线程池

```
public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```

其中重要参数如下：

```
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;
    
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);
```

###一个Handler
- InternalHandler sHandler：用于子线程和主线程之间的切换，因为需要将执行的子线程切换到主线程，所以需要在主线程实例化，因此要求Handler要在主线程创建。另外因为sHandler是静态成员，在类加载的时候就进行了初始化，因此相当于变相要求AsyncTask的类必须在主线程中加载，否则同一进程的AsyncTask都无法使用。

```
private static InternalHandler sHandler;
```

###一个Callable、一个Runnable
- WorkerRunnable mWorker：AsyncTask中的抽象内部类实例，这个内部类实现了Callable<Result> 接口中的call()方法

```
private final WorkerRunnable<Params, Result> mWorker;
```

- FutureTask mFuture：mFuture实际上是java.util.concurrent.FutureTask的实例，而FutureTask实现了RunnableFuture接口，RunnableFuture接口继承了Runnable和Future接口，所以mFuture实际上是一个Runnable和Future，可用于异步计算，中途可取消。

```
private final FutureTask<Result> mFuture;
```

两者均在AsyncTask构造方法中实例化，主要的异步任务通过这两个变量完成，代码如下：

```
    /**
     * Creates a new asynchronous task. This constructor must be invoked on the UI thread.
     */
    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
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

注意mWorker和mFuture之间的联系和区别。

- 从实现上看，mWorker是Callable接口的实现，mFuture是Runnable和Future接口的实现。
- 在mFuture对象实例化的时候，传入了mWorker作为构造方法的参数，实际上是把这个Callable对象赋给了FutureTask类中的callable对象（FutureTask中的Callable对象就叫callable，这个是变量名），这个callable对象会在FutureTask中的run方法中调用call()方法，而call方法可从实例化的代码中看到，调用了doInBackground(mParams)方法，并将这个返回的结果Result对象传给了postResult方法中，通过postResult方法从子线程切换到主线程。
- 注意mParams这个参数，在调用executeOnExecutors方法中就设进了mWorker对象中，因此这里doInBackground(mParams)的mParams是在那个时候设定好的。

##重要方法实现
###AsyncTask用法

```
private class MyTask extends AsyncTask<String, Integer, String> { 
 
        //onPreExecute方法用于在执行后台任务前做一些UI操作  
        @Override  
        protected void onPreExecute() {  
        }  
  
        //doInBackground方法内部执行后台任务,不可在此方法内修改UI，必须重写这个方法
        @Override  
        protected String doInBackground(String... params) {  

            return null;  
        }  
  
        //onProgressUpdate方法用于更新进度信息  
        @Override  
        protected void onProgressUpdate(Integer... progresses) {  
        }  
  
        //onPostExecute方法用于在执行完后台任务后更新UI,显示结果  
        @Override  
        protected void onPostExecute(String result) {   
        }  
  
        //onCancelled方法用于在取消执行中的任务时更改UI  
        @Override  
        protected void onCancelled() {   
        }  
    }  
    
MyTask task = new MyTask();
task.execute();
```

###入口方法：
```
new AsyncTask<String, Integer, String>().execute();
```
以下从上面的入口方法开始分析：

####new AsyncTask()

由上可知，在实例化的时候，创建了一个Callable对象mWorker和Runnable对象mFuture

####execute()

```
    @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```

这个方法调用了executeOnExecutor方法，从传入的参数可知，这里用到了用于任务排队的线程池，以及传入的参数。以下看看executeOnExecutor方法，了解是怎么进行任务排队的。

####executeOnExecutor(Executor exec, Params... params)

```
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

可以看到，只有在状态是PENDING的时候才能对加进来的任务进行排队，执行流程如下：

- onPreExecute()：空方法，可重写该方法实现进队前需要实现的操作
- 设置mWorker的参数，由上可知，mWorker是一个Callable类型，主要用于执行
- exec.execute(mFuture)：通过sDefaultExecutor的execute方法进行排队，此时传入mFuture参数
	

sDefaultExecutor是SerialExecutor类的对象，SerialExecutor类是一个私有内部类，该类代码如下：

```
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

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

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

由代码可知，这个线程池内部实际上实例化了一个任务队列，以及一个Runnable对象，表示当前活跃的Runnable任务。执行步骤如下：

- 当执行了execute方法之后，新创建一个Runnable对象放进对象中，在新创建的Runnable对象中的run方法执行传入参数r的run方法，即执行的方法是mFuture这个变量的run方法。而这个run方法会调用mFuture实例化时传入的构造参数callable对象的call方法，而这个callable就是在构造时传入的mWorker对象。所以，最终的实现对象和方法是mWorker.call()
- 因为每个进队的任务不能保证一定能成功，因此即使执行不成功，也要执行下一个进队任务的操作。所以scheduleNext()方法就是做这样的工作。在mTasks把一个任务放进队列之后，如果mActive==null，即没有活跃的任务的时候，就会从队头中取出一个任务，赋给mActive，然后通过THREAD_POOL_EXECUTOR的execute方法执行这个任务。
- 由上可知，THREAD_POOL_EXECUTOR是一个ThreadPoolExecutor的实例，是负责真正任务的执行的。所以执行到现在，这个任务就会通过线程池执行，在执行的时候，会回调这个任务的run方法，而run方法中会调用到mWorker对象的call方法，然后就执行了doInBackground(mParams)方法，并将结果通过postResult方法从子线程切换到主线程，并将结果传到主线程做更新。
- doInBackground(mParams)方法由用户自己定义，因为执行的时候在子线程环境下，因此可在这个方法中做耗时操作

####postResult(Result result)

该方法主要用于从子线程切换到主线程，用到了InternalHandler的实例，代码如下：

```
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```

由代码可知，这个方法中执行的操作是构造了一个消息，然后把子线程执行的结果封装成一个AsyncTaskResult对象，然后通过Handler传递到主线程。Handler实现代码如下：

```
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
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
```

由代码可知，在切换到主线程后，通过AysncTaskResult对象的mTask的finish方法将执行的结果传入。在这里还得看一下这个类是如何实现的。

```
    @SuppressWarnings({"RawUseOfParameterizedType"})
    private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```

mTask对象所属的类是AsyncTask类，因此实际上调用了AsyncTask的finish方法，具体实现如下：

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

由方法可知，当切换到主线程时，执行了AsyncTaskResult对象的mTask对象，即AsyncTask对象的finish方法，在这个方法中，执行操作如下：

- 判断任务是否被取消，如果没有取消，则执行onPostExecute(result)方法，这个方法由用户重写，用于更新UI等操作。
- 如果任务被取消，则执行onCancelled(result)方法，同样可由用户重写。
- 最后设置状态mStatus为完成态。

####publishProgress(Progress... values)和onProgressUpdate(Progress... values)

在Handler的handleMessage方法中，有个参数入口可调用到由用户重写的onProgressUpdate方法，而这个消息参数是通过AsyncTask类的publishProgress方法进行发送的，这个方法的实现如下：

```
    @WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```

方法很简单，仅仅是把执行过程中的进度参数通过Handler发到主线程进行更新。由此可知这个方法应该在子线程中执行，可在doInBackground方法中调用，把后台执行过程的进度参数传给主线程。

至此，整个AsyncTask的执行流程已经解析完毕。

##总结

执行流程如下：

- 实例化AsyncTask对象 

-> 本质是实例化了一个Callable对象和一个Runnable对象，并把Callable对象mWorker封装进mFuture中 

- 执行execute()方法 

-> onPreExecute() , mWorker.mParams = params

-> exec.execute(mFuture) 

-> 即sDefaultExecutor对象调用execute()方法，执行操作是将新实例化一个Runnable对象，在这个Runnable对象的run方法中执行mFuture的run方法，并把这个任务进队，然后取队列中位于队头的任务出队，通过THREAD_POOL_EXECUTOR这个执行线程池执行对应的任务 

-> 通过线程池内部机制回调任务中的run方法，run方法中调用到mFuture的run方法，而mFuture的run方法中调用到了mWorker的call方法，因此实际执行的是mWorker的call方法 

-> 在call方法中执行了doInBackground方法，进行耗时操作 

-> 通过postResult方法从子线程切换到主线程，并把结果回传到主线程 

-> 主线程回调AsyncTask的finish方法进行更新，finish方法中调用到了onPostExecute()方法或者onCancelled()方法，这两个方法可由用户重写 

-> 到此AsyncTask整个调用过程完毕。

简单来说，当执行execute方法时，在主线程中构造任务，通过sDefaultExecutor线程池将任务进队，等待THREAD_POOL_EXECUTOR线程池执行任务（此时从主线程切换到子线程），执行时回调mFuture的run方法，run方法中回调mWorker的call方法，在call方法中调用doInBackground方法执行耗时操作，执行完毕后通过postResult方法切换到主线程，并执行onPostExecute()方法更新，调用过程结束。
