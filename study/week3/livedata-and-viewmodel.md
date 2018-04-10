# about-livedata-and-viewmodel
> Handler의 관한 것을 정리
>
> present time : 2018-04-01-SUN
> last updated : 2018-04-05-03:21-seohyun99

----------------

## 0. 공식 문서
### 0-1. 레퍼런스
* [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html)
* [LiveData](https://developer.android.com/reference/android/arch/lifecycle/LiveData.html)

### 0-2. 가이드
* [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html)
* [LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html)
----------------

## 1. ViewModel
> ViewModel is a class that is responsible for preparing and managing the data for an Activity or a Fragment. It also handles the communication of the Activity / Fragment with the rest of the application

사용하기 위해서는 ViewModel이 Abstract class라 별도의 클래스를 하나 만들고 ViewModel을 상속받는다.

#### 사용시 이점
: 앱이 회전되거나 다시 그려져야하는 경우, UI에 세팅되어 있던 데이터들을 onSaveInstance() 메서드를 통해 원복할 수 있지만 Serializable이 되는 데이터, 소량의 데이터에만 이용하기에 적합하다. 하지만 ViewModel을 이용하면, UI와 내부 로직을 분리할 수 있고, 리소스 관리가 용이해져 메모리를 관리하는데 간편하다는 장점이 있다.


## 1-1. ViewModel 객체 가져오기

ViewModel은 일반적으로 Object를 생성하는 **new** 키워드로 생성하지 않고 **ViewModelProvider** 를 통해서만 가능하다.

#### CustomViewModel
```java
public class ExampleViewModel extends ViewModel {
    private Integer num;

    public Integer getNum() {
        return num;
    }
    public void setNum(int i) {
        this.num = i;
    }
}
```

### 1-1-1. DefaultFactory를 이용해 가져오기
```java
// ExampleViewModel extends ViewModel
ExampleViewModel viewmodel = ViewModelProviders.of(this).get(ExampleViewModel.class);
```

### 1-1-2. CustomFactory를 이용해 가져오기
```java
// CustomViewModelFactory
public class ViewModelFactory implements ViewModelProvider.Factory {
    private final TempData viewModelData;
    public ViewModelFactory(TempData viewModelData) {
        this.viewModelData = viewModelData;
    }

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (modelClass.isAssignableFrom(ExampleViewModel.class)) {
            return (T) new CustomViewModel(viewModelData);
        }
        throw new IllegalArgumentException("Unknown ViewModel class");
    }
}
```

## 1-2. 두 개의 Fragment에서 ViewModel 공유하기

```java
//ViewModel
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}
```
```java
//Fragment
public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends Fragment {
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, { item ->
           // Update the UI.
        });
    }
}
```

getActivity()를 통해 같은 ViewModelProviders에 같은 Activity를 전달하여, 동일한 ViewModelProvider를 가져온다.  
가져온 ViewModelProvider에서 get(Class class)메소드를 이용하여 같은 ViewModel을 가져온다.



## 1-3. Related Class
### 1-3-1. ViewModelProviders
> Utilities methods for **ViewModelStore** class.

#### important methods
1. static // **of**(Activity(or Fragment) view, ViewModelProvider.Factory Factory)  
: 주어진 Activity 또는 Fragment가 살아있는 동안 ViewModel을 유지하는 ViewModelProvider를 만든다.  
: ViewModelProvider를 리턴한다.

```java
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}
```

### 1-3-2. ViewModelProvider
> An utility class that provides ViewModels for a scope.

#### important methods
1. static // **get**(Class<T﻿﻿ 'extends ViewModel> modelClass)  
```java
private static final String DEFAULT_KEY = "android.arch.lifecycle.ViewModelProvider.DefaultKey";
```
```java
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}
```
```java
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);
    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```
: get(Class modelClass) -> get(String key, Class modelClass)를 참조한다.  
: ViewModelStore 내부 HashMap<String, ViewModel>의 값을 get해서 ViewModel을 가져온다.  

### 1-3-3. ViewModelStores
> Factory methods for ViewModelStore class.

#### important methods
1. static // **of**(Activity(or Fragment) view)  
: 주어진 fragment 또는 Activity의 ViewModelStroe를 반환한다.

```java
@NonNull
@MainThread
public static ViewModelStore of(@NonNull FragmentActivity activity) {
    if (activity instanceof ViewModelStoreOwner) {
        return ((ViewModelStoreOwner) activity).getViewModelStore();
    }
    return holderFragmentFor(activity).getViewModelStore();
}
```

holderFragmentFor(activity) 메소드를 통해 HolderFragment를 반환받고 반환받은 HolderFragment를 통해 ViewModelStore를 가져온다.

### 1-3-4. ViewModelStore
> Class to store ViewModels.

```java
private final HashMap<String, ViewModel> mMap = new HashMap<>();
```

ViewModel을 value로 하는 HashMap을 갖고 있으며, 해당 Map의 데이터를 조작할 수 있는 **put(String key, ViewModel viewModel)**, **get(String key)**, **clear()** 메서드를 갖고 있다.

```java
final void put(String key, ViewModel viewModel) {
    ViewModel oldViewModel = mMap.put(key, viewModel);
    if (oldViewModel != null) {
        oldViewModel.onCleared();
    }
}
final ViewModel get(String key) {
    return mMap.get(key);
}

//Clears internal storage and notifies ViewModels that they are no longer used.
public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.onCleared();
    }
    mMap.clear();
}
```

## 1-4. 사용시 주의점
: ViewModel은 View의 Lifecycle과 관련이 없기 때문에 ViewModel에서 Activity 등 UI의 Context를 참조하면 MemoryLeaks를 발생시키는 원인이 될 수 있다.  
: Context를 참조해야되는 상황에서는 안드로이드에서 제공해주는 **AndroidViewModel** 클래스를 이용하면 된다.

```java
public class AndroidViewModel extends ViewModel {
    private Application mApplication;
    public AndroidViewModel(Application application) {
        mApplication = application;
    }
    // Return the application.
    public <T extends Application> T getApplication() {
        return (T) mApplication;
    }
}
```

----------------

## 2. LiveData
> LiveData is a data holder class that can be observed within a given lifecycle. This means that an Observer can be added in a pair with a LifecycleOwner, and this observer will be notified about modifications of the wrapped data only if the paired LifecycleOwner is in active state.

#### 사용시 이점
1. Obserevers를 통해 데이터가 변경되는 경우 즉각적으로 수행할 수 있어 UI가 데이터 상태와 일치하는 것을 보장한다.
2. Observers는 Lifecycle Object에 바인딩 되어 있고, destroyed 되는 경우에 알아서 clean up하기 때문에 memory leaks를 피할 수 있다.
3. Observers는 lifecycle이 inactive인 경우 event를 수신하지 않기때문에 에러가 발생할 염려가 없다.
4. lifecycle이 inactive 상태였다 active 상태로 변한 경우 최신 데이터를 수신해 항상 최신의 데이터를 보장한다.

→ LifecycleOwner의 getLifeCycle() 메서드를 통해 현재의 lifecycle을 가져올 수 있어 상태 체크가 가능하다.


### 2-1. Observer
**observeForever(Observer observer)**와 **observe(LifecycleOwner owner, Observer<T> observer)** 를 통해 복수 개의 observer를 추가할 수 있다.

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}

@MainThread
public void observeForever(@NonNull Observer<T> observer) {
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```
두 메서드의 파라미터를 보면 LifeCycleOwner를 갖고 있냐 없냐의 차이가 있다.

또한 내부를 보면 wrapper의 클래스가 다른 것을 볼 수 있다.  
observe 메서드를 통해 생성된 wrapper는 **LifecycleBoundObserver** 이고,  
observeForever 메서드를 통해 생성도니 wrapper의 클래스는 **AlwaysActiveObserver** 이다.


두 메서드는 LiveData에 Observer를 추가한다는 공통점을 갖고 있지만,   
observeForever는 항상 activity 상태인 것으로 간주되므로, 이러한 Observer는 removeObserver(Observer)를 수동으로 호출해주어야 한다.
observe를 통해 추가된 Observer는 해당 lifecycle이 STARTED와 RESUME 상태일 때 active 상태로 간주하며,DESTROYED 상태로 전환될 때 자동으로 제거된다.

#### 사용 예제
```java
ViewModel mModel = ViewModelProviders.of(this).get(ExampleViewModel.class);
final Observer<String> nameObserver = newName -> {
    //todo:
};
mModel.getCurrentName().observe(this, nameObserver);
```

## 2-2. data를 set시키는 방법
```java
private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```
```java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```

메인 스레드에서 데이터를 set시킬 때에는 setValue(value)를 이용하고 작업 스레드에서 value를 세팅해야 될 때에는 postValue(value)를 이용해야 한다.

## 2-3. LifeCycleOwner
현재의 Lifecycle을 반환하는 getLifecycle() 메서드를 포함한 인터페이스이다.
> 2018년 1월 22일부터 FragmentActivity와 AppCompatActivity에서 getLifeCycle() 메서드를 지원한다.  
> aac 1.0에서 지원하던 LifecycleActivity 와 LifecycleFragment은 depreaced 되었다.

```java
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

Lifecycle의 내부는 public enum으로 Event와 State가 있으며,  
Event에는 ON_CREATE, ON_START, ...(resume, pause, stop), ON_DESTORYED와 모든 이벤트인 ON_ANY가 있다.  
STATE에는 DESTROYED, INITIALIZED, CREATED, STARTED, RESUMED의 다섯 가지 STATE가 있다.
