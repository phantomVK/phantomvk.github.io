---
layout:     post
title:      ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---



```groovy
implementation 'androidx.appcompat:appcompat:1.2.0'
implementation "androidx.lifecycle:lifecycle-common-java8:2.2.0"
```



```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        getLifecycle().addObserver(new Observer());
    }

    static class Observer implements DefaultLifecycleObserver {
        @Override
        public void onCreate(@NonNull LifecycleOwner owner) {
            // LifecycleOwner就是这个Observer注册所在的Activity，即MainActivity
            Log.i("Observer", "doOnCreate()");
        }
    }
}
```



AppCompatActivity -> FragmentActivity -> ComponentActivity



代码裁剪后

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {

    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }

    @CallSuper
    @Override
    protected void onSaveInstanceState(@NonNull Bundle outState) {
        Lifecycle lifecycle = getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).setCurrentState(Lifecycle.State.CREATED);
        }
        super.onSaveInstanceState(outState);
        mSavedStateRegistryController.performSave(outState);
    } 

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```



里面订阅了生命周期，调用dispatch()，dispatch里面获取Activity实现接口LifecycleRegistryOwner的getLifecycle()，然后在返回的Lifecycle上调用handleLifecycleEvent

```java
@SuppressWarnings("UnknownNullness") // TODO https://issuetracker.google.com/issues/112197238
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class ReportFragment extends Fragment {
    private static final String REPORT_FRAGMENT_TAG = "androidx.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";

    public static void injectIfNeededIn(Activity activity) {
        // ProcessLifecycleOwner should always correctly work and some activities may not extend
        // FragmentActivity from support lib, so we use framework fragments for activities
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }

    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }

    private ActivityInitializationListener mProcessListener;

    private void dispatchCreate(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onCreate();
        }
    }

    private void dispatchStart(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onStart();
        }
    }

    private void dispatchResume(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onResume();
        }
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    void setProcessListener(ActivityInitializationListener processListener) {
        mProcessListener = processListener;
    }

    interface ActivityInitializationListener {
        void onCreate();

        void onStart();

        void onResume();
    }
}
```



下面 handleLifecycleEvent -> sync -> forwardPass()，从FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap获取value，即ObserverWithState，进行生命周期派发

```java
/**
 * An implementation of {@link Lifecycle} that can handle multiple observers.
 * <p>
 * It is used by Fragments and Support Library Activities. You can also directly use it if you have
 * a custom LifecycleOwner.
 */
public class LifecycleRegistry extends Lifecycle {

    /**
     * Custom list that keeps observers and can handle removals / additions during traversal.
     *
     * Invariant: at any moment of time for observer1 & observer2:
     * if addition_order(observer1) < addition_order(observer2), then
     * state(observer1) >= state(observer2),
     */
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
    /**
     * Current state
     */
    private State mState;
    /**
     * The provider that owns this Lifecycle.
     * Only WeakReference on LifecycleOwner is kept, so if somebody leaks Lifecycle, they won't leak
     * the whole Fragment / Activity. However, to leak Lifecycle object isn't great idea neither,
     * because it keeps strong references on all other listeners, so you'll leak all of them as
     * well.
     */
    private final WeakReference<LifecycleOwner> mLifecycleOwner;

    private int mAddingObserverCounter = 0;

    private boolean mHandlingEvent = false;
    private boolean mNewEventOccurred = false;

    // we have to keep it for cases:
    // void onStart() {
    //     mRegistry.removeObserver(this);
    //     mRegistry.add(newObserver);
    // }
    // newObserver should be brought only to CREATED state during the execution of
    // this onStart method. our invariant with mObserverMap doesn't help, because parent observer
    // is no longer in the map.
    private ArrayList<State> mParentStates = new ArrayList<>();

    /**
     * Creates a new LifecycleRegistry for the given provider.
     * <p>
     * You should usually create this inside your LifecycleOwner class's constructor and hold
     * onto the same instance.
     *
     * @param provider The owner LifecycleOwner
     */
    public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
    }

    /**
     * Moves the Lifecycle to the given state and dispatches necessary events to the observers.
     *
     * @param state new state
     * @deprecated Use {@link #setCurrentState(State)}.
     */
    @Deprecated
    @MainThread
    public void markState(@NonNull State state) {
        setCurrentState(state);
    }

    /**
     * Moves the Lifecycle to the given state and dispatches necessary events to the observers.
     *
     * @param state new state
     */
    @MainThread
    public void setCurrentState(@NonNull State state) {
        moveToState(state);
    }

    /**
     * Sets the current state and notifies the observers.
     * <p>
     * Note that if the {@code currentState} is the same state as the last call to this method,
     * calling this method has no effect.
     *
     * @param event The event that was received
     */
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            // we will figure out what to do on upper level.
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }

    private boolean isSynced() {
        if (mObserverMap.size() == 0) {
            return true;
        }
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }

    private State calculateTargetState(LifecycleObserver observer) {
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

        State siblingState = previous != null ? previous.getValue().mState : null;
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        return min(min(mState, siblingState), parentState);
    }

    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }

        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }

    private void popParentState() {
        mParentStates.remove(mParentStates.size() - 1);
    }

    private void pushParentState(State state) {
        mParentStates.add(state);
    }

    @Override
    public void removeObserver(@NonNull LifecycleObserver observer) {
        // we consciously decided not to send destruction events here in opposition to addObserver.
        // Our reasons for that:
        // 1. These events haven't yet happened at all. In contrast to events in addObservers, that
        // actually occurred but earlier.
        // 2. There are cases when removeObserver happens as a consequence of some kind of fatal
        // event. If removeObserver method sends destruction events, then a clean up routine becomes
        // more cumbersome. More specific example of that is: your LifecycleObserver listens for
        // a web connection, in the usual routine in OnStop method you report to a server that a
        // session has just ended and you close the connection. Now let's assume now that you
        // lost an internet and as a result you removed this observer. If you get destruction
        // events in removeObserver, you should have a special case in your onStop method that
        // checks if your web connection died and you shouldn't try to report anything to a server.
        mObserverMap.remove(observer);
    }

    /**
     * The number of observers.
     *
     * @return The number of observers.
     */
    @SuppressWarnings("WeakerAccess")
    public int getObserverCount() {
        return mObserverMap.size();
    }

    @NonNull
    @Override
    public State getCurrentState() {
        return mState;
    }

    static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }

    private static Event downEvent(State state) {
        switch (state) {
            case INITIALIZED:
                throw new IllegalArgumentException();
            case CREATED:
                return ON_DESTROY;
            case STARTED:
                return ON_STOP;
            case RESUMED:
                return ON_PAUSE;
            case DESTROYED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }

    private static Event upEvent(State state) {
        switch (state) {
            case INITIALIZED:
            case DESTROYED:
                return ON_CREATE;
            case CREATED:
                return ON_START;
            case STARTED:
                return ON_RESUME;
            case RESUMED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }

    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }

    // happens only on the top of stack (never in reentrance),
    // so it doesn't have to take in account parents
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }

    static State min(@NonNull State state1, @Nullable State state2) {
        return state2 != null && state2.compareTo(state1) < 0 ? state2 : state1;
    }

    static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            // 我们的Observer被封装为FullLifecycleObserverAdapter
            // 由FullLifecycleObserverAdapter的onStateChanged()给我们的Observer派发生命周期状态
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
}
```



上面的静态内部类ObserverWithState，调用Lifecycling.lifecycleEventObserver(observer);







```java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class Lifecycling {

    private static final int REFLECTIVE_CALLBACK = 1;
    private static final int GENERATED_CALLBACK = 2;

    private static Map<Class, Integer> sCallbackCache = new HashMap<>();
    private static Map<Class, List<Constructor<? extends GeneratedAdapter>>> sClassToAdapters =
            new HashMap<>();

    // Left for binary compatibility when lifecycle-common goes up 2.1 as transitive dep
    // but lifecycle-runtime stays 2.0

    /**
     * @deprecated Left for compatibility with lifecycle-runtime:2.0
     */
    @Deprecated
    @NonNull
    static GenericLifecycleObserver getCallback(final Object object) {
        final LifecycleEventObserver observer = lifecycleEventObserver(object);
        return new GenericLifecycleObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                observer.onStateChanged(source, event);
            }
        };
    }

    @NonNull
    static LifecycleEventObserver lifecycleEventObserver(Object object) {
        boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
        boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
        if (isLifecycleEventObserver && isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                    (LifecycleEventObserver) object);
        }
        
        // 创建的Observer在这里被封装为FullLifecycleObserverAdapter
        if (isFullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
        }

        if (isLifecycleEventObserver) {
            return (LifecycleEventObserver) object;
        }

        final Class<?> klass = object.getClass();
        int type = getObserverConstructorType(klass);
        if (type == GENERATED_CALLBACK) {
            List<Constructor<? extends GeneratedAdapter>> constructors =
                    sClassToAdapters.get(klass);
            if (constructors.size() == 1) {
                GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                        constructors.get(0), object);
                return new SingleGeneratedAdapterObserver(generatedAdapter);
            }
            GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
            for (int i = 0; i < constructors.size(); i++) {
                adapters[i] = createGeneratedAdapter(constructors.get(i), object);
            }
            return new CompositeGeneratedAdaptersObserver(adapters);
        }
        return new ReflectiveGenericLifecycleObserver(object);
    }

    private static GeneratedAdapter createGeneratedAdapter(
            Constructor<? extends GeneratedAdapter> constructor, Object object) {
        //noinspection TryWithIdenticalCatches
        try {
            return constructor.newInstance(object);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        } catch (InstantiationException e) {
            throw new RuntimeException(e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException(e);
        }
    }

    @Nullable
    private static Constructor<? extends GeneratedAdapter> generatedConstructor(Class<?> klass) {
        try {
            Package aPackage = klass.getPackage();
            String name = klass.getCanonicalName();
            final String fullPackage = aPackage != null ? aPackage.getName() : "";
            final String adapterName = getAdapterName(fullPackage.isEmpty() ? name :
                    name.substring(fullPackage.length() + 1));

            @SuppressWarnings("unchecked") final Class<? extends GeneratedAdapter> aClass =
                    (Class<? extends GeneratedAdapter>) Class.forName(
                            fullPackage.isEmpty() ? adapterName : fullPackage + "." + adapterName);
            Constructor<? extends GeneratedAdapter> constructor =
                    aClass.getDeclaredConstructor(klass);
            if (!constructor.isAccessible()) {
                constructor.setAccessible(true);
            }
            return constructor;
        } catch (ClassNotFoundException e) {
            return null;
        } catch (NoSuchMethodException e) {
            // this should not happen
            throw new RuntimeException(e);
        }
    }

    private static int getObserverConstructorType(Class<?> klass) {
        Integer callbackCache = sCallbackCache.get(klass);
        if (callbackCache != null) {
            return callbackCache;
        }
        int type = resolveObserverCallbackType(klass);
        sCallbackCache.put(klass, type);
        return type;
    }

    private static int resolveObserverCallbackType(Class<?> klass) {
        // anonymous class bug:35073837
        if (klass.getCanonicalName() == null) {
            return REFLECTIVE_CALLBACK;
        }

        Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
        if (constructor != null) {
            sClassToAdapters.put(klass, Collections
                    .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
            return GENERATED_CALLBACK;
        }

        boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
        if (hasLifecycleMethods) {
            return REFLECTIVE_CALLBACK;
        }

        Class<?> superclass = klass.getSuperclass();
        List<Constructor<? extends GeneratedAdapter>> adapterConstructors = null;
        if (isLifecycleParent(superclass)) {
            if (getObserverConstructorType(superclass) == REFLECTIVE_CALLBACK) {
                return REFLECTIVE_CALLBACK;
            }
            adapterConstructors = new ArrayList<>(sClassToAdapters.get(superclass));
        }

        for (Class<?> intrface : klass.getInterfaces()) {
            if (!isLifecycleParent(intrface)) {
                continue;
            }
            if (getObserverConstructorType(intrface) == REFLECTIVE_CALLBACK) {
                return REFLECTIVE_CALLBACK;
            }
            if (adapterConstructors == null) {
                adapterConstructors = new ArrayList<>();
            }
            adapterConstructors.addAll(sClassToAdapters.get(intrface));
        }
        if (adapterConstructors != null) {
            sClassToAdapters.put(klass, adapterConstructors);
            return GENERATED_CALLBACK;
        }

        return REFLECTIVE_CALLBACK;
    }

    private static boolean isLifecycleParent(Class<?> klass) {
        return klass != null && LifecycleObserver.class.isAssignableFrom(klass);
    }

    /**
     * Create a name for an adapter class.
     */
    public static String getAdapterName(String className) {
        return className.replace(".", "_") + "_LifecycleAdapter";
    }

    private Lifecycling() {
    }
}
```



我们的Observer是DefaultLifecycleObserver，DefaultLifecycleObserver有继承自FullLifecycleObserver，下面可见，FullLifecycleObserver被封装到FullLifecycleObserverAdapter。



在生命周期发生变化时，FullLifecycleObserverAdapter.onStateChanged会switch case所在生命周期并回调我们的Observer



```java
class FullLifecycleObserverAdapter implements LifecycleEventObserver {

    private final FullLifecycleObserver mFullLifecycleObserver;
    private final LifecycleEventObserver mLifecycleEventObserver;

    FullLifecycleObserverAdapter(FullLifecycleObserver fullLifecycleObserver,
            LifecycleEventObserver lifecycleEventObserver) {
        mFullLifecycleObserver = fullLifecycleObserver;
        mLifecycleEventObserver = lifecycleEventObserver;
    }
    
    // 根据状态派发
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
        switch (event) {
            case ON_CREATE:
                mFullLifecycleObserver.onCreate(source);
                break;
            case ON_START:
                mFullLifecycleObserver.onStart(source);
                break;
            case ON_RESUME:
                mFullLifecycleObserver.onResume(source);
                break;
            case ON_PAUSE:
                mFullLifecycleObserver.onPause(source);
                break;
            case ON_STOP:
                mFullLifecycleObserver.onStop(source);
                break;
            case ON_DESTROY:
                mFullLifecycleObserver.onDestroy(source);
                break;
            case ON_ANY:
                throw new IllegalArgumentException("ON_ANY must not been send by anybody");
        }
        if (mLifecycleEventObserver != null) {
            mLifecycleEventObserver.onStateChanged(source, event);
        }
    }
}
```



