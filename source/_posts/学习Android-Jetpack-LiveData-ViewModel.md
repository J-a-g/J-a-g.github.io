---
title: '学习Android Jetpack:Lifecycle & LiveData & ViewModel'
date: 2021-05-13 21:40:48
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags: Android JetPack
---

## 地址

官方文档：[https://developer.android.com/topic/libraries/architecture/lifecycle](https://developer.android.com/topic/libraries/architecture/lifecycle)
官方项目文档：[https://developer.android.com/codelabs/android-lifecycles?index#0](https://developer.android.com/codelabs/android-lifecycles?index#0)
官方项目地址：[https://github.com/googlecodelabs/android-lifecycles](https://github.com/googlecodelabs/android-lifecycles)

下面文档中代码是根据上述官方资料，并结合自己实际调试情况的Sample

Sample地址：[https://github.com/J-a-g/GunDom/tree/LiveData](https://github.com/J-a-g/GunDom/tree/LiveData)


## Lifecycle 

生命周期感知型组件 **`Lifecycle`**  可执行操作来响应另一个组件（如 Activity 和 Fragment）的生命周期状态的变化。这些组件有助于您编写出更有条理且往往更精简的代码，此类代码更易于维护

主要作用：
* 精简 Activity和Fragment 生命周期方法中的大量代码，便于维护
* 处理可能出现竞态条件的问题，在这种条件下，onStop() 方法会在 onStart() 之前结束，这使得组件留存的时间比所需的时间要长

##### 使用

```js
class MyListener(private val lifecycleOwner: LifecycleOwner) : LifecycleObserver {
    init {
        lifecycleOwner.lifecycle.addObserver(this)
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onMyStart(){
         Log.w("scj", "onMyStart")
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onMyStop(){
        Log.w("scj", "onMyStop")
    }
}
class LifecycleProviderActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding =
            DataBindingUtil.setContentView(this, R.layout.activity_lifecycle)

        MyListener(this)
    }
}
```
主要 `AppCompatActivity` 已经实现了 `LifecycleOwner` 接口，此时 `LifecycleProviderActivity` 的 `onStart()、 onStop()` 生命周期方法就会回调到 `MyListener` 中通过 `@OnLifecycleEvent(Lifecycle.Event.ON_XXX)`注释的方法中

现在假设一个这样的场景，我们要在 `onStart()` 中先进行一个耗时的配置检测，然后注册一个监听器，然后在 `onStop()` 中反注册监听器，此时可能出现一个情况，就是 `onStart()`中还没执行完，用户就关闭了界面调用了 `onStop()`方法，此时会造成内存泄露，遇到这种情况我们该如何处理，在上面代码的基础进行修改：
```js
class MyListener(private val lifecycleOwner: LifecycleOwner) : LifecycleObserver {
    init {
        lifecycleOwner.lifecycle.addObserver(this)
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onMyStart(){
        Thread {
            SystemClock.sleep(2000)
            if (lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
                //do Something 注册监听器
            }else{
            }
        }.start()
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onMyStop(){
        Log.w("scj", "onMyStop")
         //do Something 反注册
    }
}

class LifecycleProviderActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding =
            DataBindingUtil.setContentView(this, R.layout.activity_lifecycle)

        MyListener(this)
    }
}
```

![java-javascript](img2.jpg)
<small class="img-hint">构成 Android Activity 生命周期的状态和事件</small>

##### 实现自定义 LifecycleOwner

直接看代码，实现也比较简单：
```js
//实现自定义 LifecycleOwner
class MyLifecycleActivity : Activity(), LifecycleOwner {

    private lateinit var lifecycleRegistry: LifecycleRegistry

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityLifecycleBinding =
            DataBindingUtil.setContentView(this, R.layout.activity_lifecycle)

        lifecycleRegistry = LifecycleRegistry(this)
        MyListener(this) //和上面LifecycleObserver实现类一样
    }

    override fun getLifecycle(): Lifecycle {
        return lifecycleRegistry
    }
}
```

注意：我们在 `MyListener` 中看到这样的代码 `lifecycleOwner.lifecycle.addObserver(this)`,这是我们就会思考初始化时添加了观察者，那么使用完需要调用 `removeObserver（）`吗？答案是[不需要](https://github.com/googlecodelabs/android-lifecycles/issues/5)，先附上官方的答案，等后续 `Lifecycle 解析`时说明一下
![java-javascript](img1.jpg)


## ViewModel

**`ViewModel`** 类旨在以注重生命周期的方式存储和管理界面相关的数据。`ViewModel` 类让数据可在发生屏幕旋转等配置更改后继续留存。

主要作用：
* 屏幕旋转时存储数据，可以在重新创建界面时快速显示数据
* 在Fragment之间共享数据
* `ViewModel` 与 `Room` 和 `LiveData` 一起使用可替换加载器,后续整理 `Room` 时再做说明
* 在进程被系统终止后继续保留，并通过同一对象保持可用状态

**屏幕旋转时存储数据，可以在重新创建界面时快速显示数据**，先看一下下面这张图：
![java-javascript](img3.jpg)
<small class="img-hint">Activity 经历屏幕旋转而后结束时所处的各种生命周期状态</small>
```js
class ViewModelActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityViewModelBinding =
            DataBindingUtil.setContentView(this, R.layout.activity_view_model)
        val chronometerViewModel = ViewModelProvider(this).get(ChronometerViewModel::class.java)
        //databing，自定义属性部分功能建议看源码
        binding.chronometerviewmodel = chronometerViewModel
        chronometerViewModel.startTime()
    }
}

class ChronometerViewModel : ViewModel() {
    //ViewModel的作用保持

    var startTime: ObservableLong? = null

    fun startTime(){
        val n = 0
        if(startTime?.get() == n.toLong() || startTime == null){
            startTime = ObservableLong(SystemClock.elapsedRealtime())
        }
    }

    override fun onCleared() {
        super.onCleared()
    }
}
```
重新创建的Activity它获取的 `ChronometerViewModel` 实例和一开始Activity中创建的实例相同，当Activity finish时，框架会调用 `onCleared()` 方法,以便清理资源 

> 注意：ViewModel 绝不能引用视图、Lifecycle 或可能存储对 Activity 上下文的引用的任何类。

**在Fragment之间共享数据**，在一个 Activity 中设计两个 Fragment，每个 Fragment 中有一个 SeekBar，移动其中一个 SeekBar 时，另外一个的值也会随着变化
```js
class SeekBarFragment : Fragment() {

    lateinit var binding: FragmentSeekBarBinding
    lateinit var seekBarViewModel: SeekBarViewModel

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        binding = DataBindingUtil.inflate(inflater, R.layout.fragment_seek_bar, container, false)

        seekBarViewModel = ViewModelProvider(requireActivity()).get(SeekBarViewModel::class.java)

        subscribeSeekBar()
        return binding.root
    }

    fun subscribeSeekBar() {
        binding.sb.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
            override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
                if(fromUser){
                    //修改ViewModel中LiveData的值，会通知到onChanged中
                    seekBarViewModel.seekbarValue.value = progress
                }
            }

            override fun onStartTrackingTouch(seekBar: SeekBar?) {
            }

            override fun onStopTrackingTouch(seekBar: SeekBar?) {
            }

        })

        seekBarViewModel.seekbarValue.observe(requireActivity(),object : Observer<Int>{
            override fun onChanged(t: Int?) {
                if (t != null) {
                    //更新UI
                    binding.sb.progress = t
                }
            }

        })
    }
}


class SeekBarViewModel : ViewModel() {

    val seekbarValue: MutableLiveData<Int> = MutableLiveData()
}
```
其中 `ViewModelProvider(requireActivity()).get(SeekBarViewModel::class.java)` ,`ViewModelProvider` 中传入的值是包含他们的 `Activity`，此时它们会收到相同的 `SeekBarViewModel` 实例，实现了 `Fragment` 间的数据共享

**在进程被系统终止后继续保留，并通过同一对象保持可用状态**,打开应用后设置 `ViewModel` 中的数据，然后将通过home键将应用退到后台，再使用 adb 命令：`adb shell am kill 包名` 来模拟进程被系统终止，然后再打开应用查看之前保留的数据,代码如下：

```js
class SavedStateViewModel(private val savedStateHandle: SavedStateHandle) : ViewModel() {

    val KEY_NAME: String = "name"
    val name: LiveData<String> = savedStateHandle.getLiveData(KEY_NAME)

    fun setName(name: String) {
        savedStateHandle[KEY_NAME] = name
    }
}


class SaveStateActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val binding: ActivitySaveStateBinding =
            DataBindingUtil.setContentView(this, R.layout.activity_save_state)

        val savedStateViewModel = ViewModelProvider(this).get(SavedStateViewModel::class.java)
        savedStateViewModel.name.observe(this, object : Observer<String>{
            override fun onChanged(t: String?) {
                binding.savedVmTv.text = resources.getString(R.string.saved_in_vm, t)
            }
        })

        binding.saveBt.setOnClickListener {
            savedStateViewModel.setName(binding.nameEt.text.toString())
        }
    }
}
```

## LiveData

**`LiveData`** 是一种可观察的数据存储器类。与常规的可观察类不同，`LiveData` 具有生命周期感知能力，意指它遵循其他应用组件（如 `Activity、Fragment 或 Service`）的生命周期。这种感知能力可确保 `LiveData` 仅更新处于活跃生命周期状态的应用组件观察者。

白话就是 `LiveData`数据发生变化时会更新到生命周期处于 STARTED 或 RESUMED 状态的观察者上，如一个Activity

##### 使用 LiveData 的优势

* **确保界面符合数据状态**
`LiveData` 遵循观察者模式。当底层数据发生变化时，`LiveData` 会通知 `Observer` 对象。您可以整合代码以在这些 `Observer` 对象中更新界面。这样一来，您无需在每次应用数据发生变化时更新界面，因为观察者会替您完成更新。
* **不会发生内存泄漏**
观察者会绑定到 `Lifecycle` 对象，并在其关联的生命周期遭到销毁后进行自我清理。
* **不会因 Activity 停止而导致崩溃**
如果观察者的生命周期处于非活跃状态（如返回栈中的 Activity），则它不会接收任何 `LiveData` 事件。
* **不再需要手动处理生命周期**
界面组件只是观察相关数据，不会停止或恢复观察。`LiveData` 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。
* **数据始终保持最新状态**
如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 `Activity` 会在返回前台后立即接收最新的数据。
* **适当的配置更改**
如果由于配置更改（如设备旋转）而重新创建了` Activity` 或 `Fragment`，它会立即接收最新的可用数据。
* **共享资源**
您可以使用单例模式扩展 `LiveData` 对象以封装系统服务，以便在应用中共享它们。`LiveData` 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 `LiveData` 对象。

##### 如何使用 LiveData

```js
class LiveDataTimerViewModel : ViewModel() {


    val ONE_SECOND: Long = 1000

    var mInitialTime: Long = 0
    var timer: Timer? = null

    val mElapsedTime: MutableLiveData<Long> = MutableLiveData()

    init {
        mInitialTime = SystemClock.elapsedRealtime()
        timer = Timer()

        timer?.scheduleAtFixedRate(object : TimerTask() {
            override fun run() {
                val newValue = (SystemClock.elapsedRealtime() - mInitialTime) / 1000
                mElapsedTime.postValue(newValue)
            }
        }, ONE_SECOND, ONE_SECOND)
    }

    override fun onCleared() {
        super.onCleared()
        timer?.cancel()
    }
}


class LiveDataActivity : AppCompatActivity() {

    var binding: ActivityLiveDataBinding? = null
    var liveDataTimerViewModel: LiveDataTimerViewModel? = null


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding =
            DataBindingUtil.setContentView(this, R.layout.activity_live_data)

        liveDataTimerViewModel = ViewModelProvider(this).get(LiveDataTimerViewModel::class.java)
        subscribe()
    }

    //订阅，mElapsedTime的值发生变成后，会回调到这个方法中更新UI
    fun subscribe() {
        val elapsedTimeObserver = Observer<Long> {
            val newText = it.toString() + "seconds elapsed"
            binding?.tvHelloWorld?.text = newText
        }
        liveDataTimerViewModel?.mElapsedTime?.observe(this, elapsedTimeObserver)
    }
}
```
> 注意：
> 1.您可以使用 `observeForever(Observer)` 方法在没有关联的 `LifecycleOwner` 对象的情况下注册一个观察者。在这种情况下，观察者会被视为始终处于活跃状态，因此它始终会收到关于修改的通知。您可以通过调用 `removeObserver(Observer)` 方法来移除这些观察者。
>&emsp;
> 2.您必须调用 `setValue(T)` 方法以从主线程更新 `LiveData` 对象。如果在工作器线程中执行代码，您可以改用 `postValue(T)` 方法来更新 `LiveData` 对象。
>&emsp;
> 3.请确保用于更新界面的 `LiveData` 对象存储在 `ViewModel` 对象中，而不是将其存储在 `Activity` 或 `Fragment` 中，原因如下：
> * 避免 `Activity` 和 `Fragment` 过于庞大。现在，这些界面控制器负责显示数据，但不负责存储数据状态。
> * 将 `LiveData` 实例与特定的 `Activity` 或 `Fragment` 实例分离开，并使 `LiveData` 对象在配置更改后继续存在。

##### 扩展 LiveData

```js
class MyElapsedTime : LiveData<Long>() {

    fun onPostValue(v: Long){
        postValue(v)
    }

    fun onValue(v: Long){
        value = v
    }

    override fun onActive() {
        super.onActive()
        Log.w("scj", "MyElapsedTime onActive ")
    }

    override fun onInactive() {
        super.onInactive()
        Log.w("scj", "MyElapsedTime onInactive ")
    }
}



        liveDataTimerViewModel?.myTime?.observe(this, Observer {
            Log.w("scj", "MyElapsedTime value-> " + it)
        })
```
这里主要要讲的是：
* 当 `LiveData` 对象具有活跃观察者时，会调用 `onActive()` 方法。
* 当 `LiveData` 对象没有任何活跃观察者时，会调用 `onInactive()` 方法。
* `setValue(T)` 方法将更新 `LiveData` 实例的值，并将更改告知活跃观察者。

>注意：
> * 如果 `Lifecycle` 对象未处于活跃状态，那么即使值发生更改，也不会调用观察者。
> * 销毁 `Lifecycle` 对象后，会自动移除观察者。

##### 转换 LiveData 

略，[自行查看官方文档](https://developer.android.com/topic/libraries/architecture/livedata#transform_livedata)

##### 合并多个 LiveData 源 

**`MediatorLiveData`** 是 `LiveData` 的子类，允许您合并多个 `LiveData` 源。只要任何原始的 `LiveData` 源对象发生更改，就会触发 `MediatorLiveData` 对象的观察者。
场景，如果界面中有可以从本地数据库或网络更新的 `LiveData` 对象，则可以向 `MediatorLiveData` 对象添加以下源：
* 与存储在数据库中的数据关联的 LiveData 对象。
* 与从网络访问的数据关联的 LiveData 对象。

您的 Activity 只需观察 MediatorLiveData 对象即可从这两个源接收更新。

下面设计一个类似场景的代码，两个 LiveData，一个 LiveData 通过定时器修改值，一个 LiveData 通过手动操作修改值，通过 MediatorLiveData 合并两个 LiveData ，观察 MediatorLiveData 来实现UI的更新
```js
class LiveDataTimerViewModel : ViewModel() {

    val ONE_SECOND: Long = 1000
    var mInitialTime: Long = 0
    var timer: Timer? = null

    val mElapsedTime: MutableLiveData<Long> = MutableLiveData()//LiveData
    val myTime: MyElapsedTime = MyElapsedTime() //LiveData
    val liveDataMerger = MediatorLiveData<Long>() //MediatorLiveData

    init {
        mInitialTime = SystemClock.elapsedRealtime()
        timer = Timer()

        //合并两个LiveData，并在LiveData值发生改变后回调到下面的代码中，修改 MediatorLiveData 值
        liveDataMerger.addSource(mElapsedTime, androidx.lifecycle.Observer {
            liveDataMerger.value = it
        })
        liveDataMerger.addSource(myTime, androidx.lifecycle.Observer {
            mInitialTime = SystemClock.elapsedRealtime()
            liveDataMerger.value = it
        })

        timer?.scheduleAtFixedRate(object : TimerTask() {
            override fun run() {
                val newValue = (SystemClock.elapsedRealtime() - mInitialTime) / 1000
                mElapsedTime.postValue(newValue)//通过定时器修改LiveData的值
            }
        }, ONE_SECOND, ONE_SECOND)
    }

    override fun onCleared() {
        super.onCleared()
        timer?.cancel()
    }
}




class LiveDataActivity : AppCompatActivity() {

    var binding: ActivityLiveDataBinding? = null
    var liveDataTimerViewModel: LiveDataTimerViewModel? = null


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding =
            DataBindingUtil.setContentView(this, R.layout.activity_live_data)

        liveDataTimerViewModel = ViewModelProvider(this).get(LiveDataTimerViewModel::class.java)
        subscribe()

        binding?.btnReset?.setOnClickListener {
            liveDataTimerViewModel?.myTime?.onValue(0)//通过手动操作修改LiveData的值
        }
    }

    fun subscribe() {
        //合并之后的订阅
        liveDataTimerViewModel?.liveDataMerger?.observe(this, Observer {
            val newText = it.toString() + "seconds elapsed"
            binding?.tvHelloWorld?.text = newText
        })
    }
}


```



