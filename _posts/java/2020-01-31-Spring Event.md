---
layout: post
title: Spring Event
category: Java相关
tags: spring
---

## Java Observer
> This class represents an observable object, or "data" in the model-view paradigm. It can be subclassed to represent an object that the application wants to have observed.

在JDK1.0的时候就有这个类了，观察者模式实现的时候可以使用此类，和JDK有耦合还好。但使用的时候必须继承Observable，并且每次notifyObservers之前需要setChanged()，看了下代码，如果没有这个changed字段的状态，安全同步应该没有问题。没有很明白作者加这个change的意图。
```
public class ObserverDemo {

    public static void main(String[] args) {
        EventObservable observable = new EventObservable();
        // 添加观察者（监听者）
        observable.addObserver(new EventObserver());
        // 发布消息（事件）
        observable.notifyObservers("Hello,World");
    }

    static class EventObservable extends Observable {

        public void notifyObservers(Object arg) {
            setChanged();
            super.notifyObservers(new EventObject(arg));
        }
    }

    static class EventObserver implements Observer {

        @Override
        public void update(Observable o, Object event) {
            EventObject eventObject = (EventObject) event;
            System.out.println("收到事件 ：" + eventObject);
        }
    }
}

```

在这Observable的addObserver和deleteObserver方法都是用synchronized进行同步保证的，而在notifyObservers方法中，是通过obs.toArray（）方式，先拷贝一份在去通知，代码中的注释也给出了这样做的可能的问题，新的observer可能miss调，而是unregistered observer可能被错误的通知。

```
    public void notifyObservers(Object arg) {
        /*
         * a temporary array buffer, used as a snapshot of the state of
         * current Observers.
         */
        Object[] arrLocal;

        synchronized (this) {
            /* We don't want the Observer doing callbacks into
             * arbitrary code while holding its own Monitor.
             * The code where we extract each Observable from
             * the Vector and store the state of the Observer
             * needs synchronization, but notifying observers
             * does not (should not).  The worst result of any
             * potential race-condition here is that:
             * 1) a newly-added Observer will miss a
             *   notification in progress
             * 2) a recently unregistered Observer will be
             *   wrongly notified when it doesn't care
             */
            if (!changed)
                return;
            arrLocal = obs.toArray();
            clearChanged();
        }

        for (int i = arrLocal.length-1; i>=0; i--)
            ((Observer)arrLocal[i]).update(this, arg);
    }
```

## Java EventObject
> The root class from which all event state objects shall be derived.
> All Events are constructed with a reference to the object, the "source", that is logically deemed to be the object upon which the Event in question initially occurred upon.

- 所有事件的对象应该继承这类，spring中ApplicationEvent也是按规矩，继承了这类。
- 在java.util中1.1新增加的，这类中有个Object  source对象，这个source是The object on which the Event initially occurred.看解释，应该是发生事件源的对象，而不是事件的数据。这个有些像消息链中trackId，上下文。

虽然也提供了个EventListener接口，但是只是个tagging interface，在jdk中没有什么默认的实现。

## spring ApplicationEvent 
jdk没有实现的东西，spring当然会给出一套，首先这event继承自EventObject，然后加上了个发生时间，很简单实用，然后这个类是个abstract，因为在spring中是通过type进行事件的分发与listener选择的。虽然
```
	/** System time when the event happened. */
	private final long timestamp;

	/**
	 * Return the system time in milliseconds when the event occurred.
	 */
	public final long getTimestamp() {
		return this.timestamp;
	}
```

ApplicationListener用来处理ApplicationEvent，也可以通过EventListener注解

```
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```

## Spring 事件机制
使用上，可以直接通过ApplicationContext进行事件监听与进行事件的分发：
```
context.addApplicationListener(new MySpringEventListener());
context.publishEvent(new MySpringEvent("Hello,World"));
```

### 监听器注册
refresh()的过程中有两个步骤如下代码，初始化事件广播器，如果没有用户定义的事件广播器，使用一个默认的实现SimpleApplicationEventMulticaster。

后面registerListeners把bean（来源不仅是通过接口调用增加的，注解的有）中的ApplicationListener子类也注册到事件广播器上。
```
// Initialize event multicaster for this context.
				initApplicationEventMulticaster();
// Check for listener beans and register them.
				registerListeners();
```

下面是增加listener的过程，其中不仅applicationEventMulticaster中增加了，applicationListeners也增加了
```
	public void addApplicationListener(ApplicationListener<?> listener) {
		Assert.notNull(listener, "ApplicationListener must not be null");
		if (this.applicationEventMulticaster != null) {
			this.applicationEventMulticaster.addApplicationListener(listener);
		}
		this.applicationListeners.add(listener);
	}
```

```
	/** Helper class used in event publishing. */
	@Nullable
	private ApplicationEventMulticaster applicationEventMulticaster;

	/** Statically specified listeners. */
	private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();
```

这applicationListeners，干啥用？看代码，首先，所有的applicationListeners最终都会加入到applicationEventMulticaster中，而ApplicationEventMulticaster这个类，是加载比较晚的，因为是一个用户可以重新定义替换默认实现的类，而在initApplicationEventMulticaster之前，有些框架的listener是需要注册的，这样就需要个临时的listener的保存的地方就是这applicationListeners，从代码上看，ApplicationEventMulticaster有@Nullable标记，而ApplicationListener是直接new出来的，不可能为null。

### 事件分发、处理
调用publishEvent方法的时候，一般处理是通过上面提到的默认的事件广播器实现的。
```
getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
```

事件的class类型为EventA，能被ApplicationListener<EventB>，处理么？或者说一个EventA能被不同的ApplicationListener处理么？答案是可以的。贴段代码：

```
	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

上面getApplicationListeners的时候，是需要event和type的，通过源码看，最终其实根据type获取的，而这eventType默认是null的，就会触发resolveDefaultEventType方法，而这个方法会判断这个类判断核心逻辑为：
```
	public static ResolvableType forInstance(Object instance) {
		Assert.notNull(instance, "Instance must not be null");
		if (instance instanceof ResolvableTypeProvider) {
			ResolvableType type = ((ResolvableTypeProvider) instance).getResolvableType();
			if (type != null) {
				return type;
			}
		}
		return ResolvableType.forClass(instance.getClass());
	}
```

也就是说，如果Event如下实现，是能够通过控制返回ResolvableType能够被不同的Listener处理的。在一个类内部事件数据类型不一样的时候，但又不想增加子类，可以通过这种方式，对同一个类，对应多个事件处理器。
```
public class MySpringEventA implements ResolvableTypeProvider {

///...

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClass(MySpringEventB.class);
    }
}
```

同样，在获取对应的listener的过程中，也是通过ResolvableType是否equal来判断的，而这ResolvableType的equal方法中是同构instance of来判断的，也就是说，如果一个ApplicationListener<ApplicationEvent>，是能够监听到所有的事件的回调的。

```
	public boolean equals(@Nullable Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof ResolvableType)) {
			return false;
		}

```