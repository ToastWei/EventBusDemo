# EventBus使用

开源地址：[https://github.com/greenrobot/EventBus](https://github.com/greenrobot/EventBus)


## 使用步骤

### 1. 在Module的build.gradle添加依赖

	compile 'org.greenrobot:eventbus:3.0.0'


### 2. 创建事件
	public class MyEvent {
	    public String msg;
	    public MyEvent(String msg) {
	        this.msg = msg;
	    }
	}

### 3. 注册和反注册EventBus
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //注册事件总线
        EventBus.getDefault().register(this);
    }


    @Override
    protected void onDestroy() {
        super.onDestroy();
        //反注册事件总线
        EventBus.getDefault().unregister(this);
    }


### 4. 监听事件
	
	/**
     * POSTING线程模型：在哪个线程发布事件，就在哪个线程执行onPostingEvent方法
     */
    @Subscribe(threadMode = ThreadMode.POSTING)
    public void onPostingEvent(MyEvent event) {
        Log.d(TAG, "onPostingEvent: " + Thread.currentThread().getName());
    }


    /**
     * MAIN线程模型：不管是哪个线程发布事件，都在主线程执行onMainEvent方法
     */
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMainEvent(MyEvent event) {
        Log.d(TAG, "onMainEvent: " + Thread.currentThread().getName());
    }

    /**
     * BACKGROUND线程模型：事件如果是在子线程发布，onBackgroundEvent方法就在该子线程执行，事件如果是在
     * 主线程中发布，onBackgroundEvent方法就在EventBus内部的线程池中执行
     */
    @Subscribe(threadMode = ThreadMode.BACKGROUND)
    public void onBackgroundEvent(MyEvent event) {
        Log.d(TAG, "onBackgroundEvent: " + Thread.currentThread().getName());
    }

    /**
     * ASYNC线程模型：不管事件在哪个线程发布，onAsyncEvent方法都在EventBus内部的线程池中执行
     */
    @Subscribe(threadMode = ThreadMode.ASYNC)
    public void onAsyncEvent(MyEvent event) {
        Log.d(TAG, "onAsyncEvent: " + Thread.currentThread().getName());
    }

### 5. 发布事件
    /**
     * 在主线程中发布事件
     * @param view
     */
    public void onPublishEventOnMainThread(View view) {
        MyEvent event = new MyEvent("msg from publisher main thread");
        EventBus.getDefault().post(event);
    }

    /**
     * 在子线程中发送事件
     * @param view
     */
    public void onPublishEventOnBGThread(View view) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                MyEvent event = new MyEvent("msg from publisher bg thread");
                EventBus.getDefault().post(event);
            }
        }).start();
    }


## EventBus源码分析 ##

### 单例模式 ###

	//简单单例，当多线程时，还是会创建多个示例
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            defaultInstance = new EventBus();
        }
        return defaultInstance;
    }

	//加锁单例，每次调用都检查是否加锁
    public static synchronized EventBus getDefault() {
        if (defaultInstance == null) {
            defaultInstance = new EventBus();
        }
        return defaultInstance;
    }

	//两个非空，一个加锁
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }


### 注册 ###

    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
		//找到订阅者中所有的订阅方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
				//将订阅方法记录下来保存到对应事件的订阅列表中
				//Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType中
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

### 发布事件 ###

    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);.//添加到事件队列

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
					//发布的单个事件
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }


    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
			........
        } else {
			//发布对应事件类型的事件
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
		.........
    }

    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);//获取对应事件的订阅列表
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
			//遍历订阅列表
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
					//发布事件到一个订阅
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                }
				........
            }
            return true;
        }
		......
    }

	private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
		//判断订阅方法的线程模型
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
				//POSTING线程模型下，直接反射调用订阅方法
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
				//MAIN线程模型
                if (isMainThread) {
					//如果当前是主线程，则直接反射调用订阅方法
                    invokeSubscriber(subscription, event);
                } else {
					//如果当前不是主线程，则使用绑定主线程Looper的Handler在主线程调用订阅方法
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
				//BACKGROUND线程模型
                if (isMainThread) {
					//如果当前是主线程，则在EventBus内部的线程池中执行
                    backgroundPoster.enqueue(subscription, event);
                } else {
					//如果当前是子线程，则直接在该子线程反射调用订阅方法
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
				//ASYNC线程模型
				//直接在EventBus的线程池中执行
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }



