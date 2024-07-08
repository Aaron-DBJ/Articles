# ViewModel

## 简介

**`ViewModel`**

类是一种[业务逻辑或屏幕级状态容器](https://developer.android.com/topic/architecture/ui-layer/stateholders?hl=zh-cn)。它用于将状态公开给界面，以及封装相关的业务逻辑。 它的主要优点是，它可以缓存状态，并可在配置更改后持久保留相应状态。这意味着在 activity 之间导航时或进行配置更改后（例如旋转屏幕时），界面将无需重新提取数据。

**`ViewModelStore`**

用于存放*ViewModel*实例的类。如果这个*ViewModelStore*的*Owner*由于配置更改（如横竖屏切换、跳转到其他页面）而被销毁并重新创建，则*Owner*的新实例应该和*ViewModelStore*旧实例相同。

如果这个*ViewModelStore*的所有者被销毁并且不打算被重新创建，那么它应该在这个*ViewModelStore*上调用clear()，这样*ViewModels*就会收到不再使用的通知。

**`ViewModelStoreOwner`**

拥有*ViewModelStore*的作用域。该接口实现的职责是在配置更改期间保留拥有的*ViewModelStore*。当这个作用域要被销毁时，调用*ViewModelStore.clear()*。

## ViewModel 的生命周期

[`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 的生命周期与其作用域直接关联。`ViewModel` 会一直保留在内存中，直到其作用域 [`ViewModelStoreOwner`](https://developer.android.com/reference/kotlin/androidx/lifecycle/ViewModelStoreOwner?hl=zh-cn) 消失。以下上下文中可能会发生这种情况：

- 对于 activity，是在 activity 完成时。
- 对于 fragment，是在 fragment 分离时。
- 对于 Navigation 条目，是在 Navigation 条目从返回堆栈中移除时。

这使得 ViewModels 成为了存储在配置更改后仍然存在的数据的绝佳解决方案。

图 1 说明了 activity 经历屏幕旋转而后结束时所处的各种生命周期状态。该图还在关联的 activity 生命周期的旁边显示了 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 的生命周期。此图表说明了 activity 的各种状态。这些基本状态同样适用于 fragment 的生命周期

<img src="https://raw.githubusercontent.com/Aaron-DBJ/ImageRepo/img/%E6%88%AA%E5%B1%8F2024-07-08%2009.53.08.png" title="" alt="" width="398">

通常在系统首次调用 activity 对象的 [`onCreate()`](https://developer.android.com/reference/android/app/Activity?hl=zh-cn#onCreate(android.os.Bundle)) 方法时请求 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn)。系统可能会在 activity 的整个生命周期内多次调用 [`onCreate()`](https://developer.android.com/reference/android/app/Activity?hl=zh-cn#onCreate(android.os.Bundle))，如在旋转设备屏幕时。[`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 存在的时间范围是从您首次请求 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 直到 activity 完成并销毁。

## 清除ViewModel

当 `ViewModelStoreOwner` 在 ViewModel 的生命周期内销毁 ViewModel 时，ViewModel 会调用 [`onCleared`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn#onCleared()) 方法。这样就可以清理遵循 ViewModel 生命周期的任何工作或依赖项。

## 创建具有依赖项的ViewModel

ViewModel 可以在其构造函数中将依赖项作为参数，通过 `ViewModelProvider.Factory` 接口。**此接口的实现才能在适当的作用域内实例化 ViewModel**。

## ViewModel 作用域 API

作用域是有效使用 ViewModel 的关键。每个 ViewModel 的作用域都**限定为一个实现 [`ViewModelStoreOwner`](https://developer.android.com/reference/androidx/lifecycle/ViewModelStoreOwner?hl=zh-cn) 接口的对象。**

**作用域内获取ViewModel实例的方法**：借助 `ViewModelProvider.get()` 方法，可以获取作用域限定为任何 `ViewModelStoreOwner` 的 ViewModel 实例

例如：

```java
import androidx.lifecycle.ViewModelProvider;

public class MyActivity extends AppCompatActivity {
    // The ViewModel is scoped to `this` Activity
    MyViewModel viewModel = new ViewModelProvider(this).get(MyViewModel.class);
}

public class MyFragment extends Fragment {
    // The ViewModel is scoped to `this` Fragment
    MyViewModel viewModel = new ViewModelProvider(this).get(MyViewModel.class);
}
```

如果要获取任意作用域（ViewModelStoreOwner）的ViewModel实例，通过View 系统中的 `ComponentActivity.viewModels()` 函数和 `Fragment.viewModels()` 函数以及 Compose 中的 `viewModel()` 函数接受可选的 `ownerProducer` 参数，可用于指定 ViewModel 实例的作用域限定为哪个 `ViewModelStoreOwner`。

以下示例展示了如何获取作用域限定为父 fragment 的 ViewModel 实例：

```java
import androidx.lifecycle.ViewModelProvider;

public class MyFragment extends Fragment {
    SharedViewModel viewModel;

    @Override
    public void onViewCreated(View view, Bundle savedInstanceState) {
        // The ViewModel is scoped to the parent of `this` Fragment
        viewModel = new ViewModelProvider(requireParentFragment())
            .get(SharedViewModel.class);
    }
}
```



## 最佳实践

以下是实现 ViewModel 时应遵循的一些重要的最佳实践：

- 由于 [ViewModel 的作用域](https://developer.android.com/topic/libraries/architecture/viewmodel?hl=zh-cn#lifecycle)，请使用 ViewModel 作为屏幕级状态容器的实现细节。请勿将它们用条状标签组或表单等可重复使用的界面组件的状态容器。否则，除非您针对每个芯片使用显式视图模型键，否则，对于同一个界面组件在同一 ViewModelStoreOwner 下的不同用法，您将会获得相同的 ViewModel 实例。
- **ViewModel 不应该知道界面实现细节。请尽可能对 ViewModel API 公开的方法和界面状态字段使用通用名称。**这样一来，ViewModel 便可以适应任何类型的界面：手机、可折叠设备、平板电脑甚至 Chromebook！
- **由于 ViewModel 的生命周期可能比 `ViewModelStoreOwner` 更长，因此 ViewModel 不应保留任何对与生命周期相关的 API（例如 `Context` 或 `Resources`）的引用，以免发生内存泄漏。**
- **请勿将 ViewModel 传递给其他类、函数或其他界面组件**。由于平台会管理它们，因此您应该使其尽可能靠近平台。应该靠近您的 activity、fragment 或屏幕级可组合函数。这样可以防止较低级别的组件访问超出其需求的数据和逻辑。





---

# LiveData

## 简介

[`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) **是一种可观察的数据存储器类**。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 activity、fragment 或 service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。

如果观察者（由 [`Observer`](https://developer.android.com/reference/androidx/lifecycle/Observer?hl=zh-cn) 类表示）的生命周期处于 [`STARTED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh-cn#STARTED) 或 [`RESUMED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh-cn#RESUMED) 状态，则 LiveData 会认为该观察者处于活跃状态。LiveData 只会将更新通知给活跃的观察者。为观察 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 对象而注册的非活跃观察者不会收到更改通知。

可以注册与实现了 [`LifecycleOwner`](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner?hl=zh-cn) 接口的对象配对的观察者。有了这种关系，当相应的 [`Lifecycle`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle?hl=zh-cn) 对象的状态变为 [`DESTROYED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh-cn#DESTROYED) 时，便可移除此观察者。这对于 activity 和 fragment 特别有用，因为它们可以放心地观察 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 对象，而不必担心泄露（当 activity 和 fragment 的生命周期被销毁时，系统会立即退订它们）。

`LiveData` 的部分特性如下：

- `LiveData` 可存储数据；`LiveData` 是一种可与任何类型数据搭配使用的封装容器。
- `LiveData` 是可观察的，这意味着当 `LiveData` 对象存储的数据发生更改时，观察器会收到通知。
- `LiveData` 具有生命周期感知能力。当您将观察器附加到 `LiveData` 后，观察器就会与 [`LifecycleOwner`](https://developer.android.com/topic/libraries/architecture/lifecycle?hl=zh-cn#lco)（通常是 activity 或 fragment）相关联。`LiveData` 仅更新处于活跃生命周期状态（例如 [`STARTED`](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.State.html?hl=zh-cn#STARTED) 或 [`RESUMED`](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.State.html?hl=zh-cn#RESUMED)）的观察器。

`MutableLiveData` 是 `LiveData` 的可变版本，也就是说，其中存储的数据的值是可以更改的。

## 优点

**确保界面符合数据状态**

LiveData 遵循观察者模式。当底层数据发生变化时，LiveData 会通知 [`Observer`](https://developer.android.com/reference/androidx/lifecycle/Observer?hl=zh-cn) 对象。您可以整合代码以在这些 `Observer` 对象中更新界面。这样一来，您无需在每次应用数据发生变化时更新界面，因为观察者会替您完成更新。

**不会发生内存泄漏**

观察者会绑定到 [`Lifecycle`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle?hl=zh-cn) 对象，并在其关联的生命周期遭到销毁后进行自我清理。

**不会因 Activity 停止而导致崩溃**

如果观察者的生命周期处于非活跃状态（如返回堆栈中的 activity），它便不会接收任何 LiveData 事件。

**不再需要手动处理生命周期**

界面组件只是观察相关数据，不会停止或恢复观察。LiveData 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。

**数据始终保持最新状态**

如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 Activity 会在返回前台后立即接收最新的数据。

**适当的配置更改**

如果由于配置更改（如设备旋转）而重新创建了 activity 或 fragment，它会立即接收最新的可用数据。

**共享资源**

您可以使用单例模式扩展 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 对象以封装系统服务，以便在应用中共享它们。`LiveData` 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 `LiveData` 对象。如需了解详情，请参阅[扩展 LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html?hl=zh-cn#extend_livedata)。

## 使用LiveData

请按照以下步骤使用 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 对象：

1. 创建 `LiveData` 的实例以存储某种类型的数据。这通常在 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 类中完成。
2. 创建可定义 [`onChanged()`](https://developer.android.com/reference/androidx/lifecycle/Observer?hl=zh-cn#onChanged(T)) 方法的 [`Observer`](https://developer.android.com/reference/androidx/lifecycle/Observer?hl=zh-cn) 对象，该方法可以控制当 `LiveData` 对象存储的数据更改时会发生什么。通常情况下，您可以在界面控制器（如 activity 或 fragment）中创建 `Observer` 对象。
3. 使用 [`observe()`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn#observe(androidx.lifecycle.LifecycleOwner,%0Aandroidx.lifecycle.Observer%3CT%3E)) 方法将 `Observer` 对象附加到 `LiveData` 对象。`observe()` 方法会采用 [`LifecycleOwner`](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner?hl=zh-cn) 对象。这样会使 `Observer` 对象订阅 `LiveData` 对象，以使其收到有关更改的通知。通常情况下，您可以在界面控制器（如 activity 或 fragment）中附加 `Observer` 对象。

> **注意**：您可以使用 [`observeForever(Observer)`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn#observeForever(androidx.lifecycle.Observer%3CT%3E)) 方法在没有关联的 [`LifecycleOwner`](https://developer.android.com/reference/androidx/lifecycle/LifecycleOwner?hl=zh-cn) 对象的情况下注册一个观察者。在这种情况下，观察者会被视为始终处于活跃状态，因此它始终会收到关于修改的通知。您可以通过调用 [`removeObserver(Observer)`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn#removeObserver(androidx.lifecycle.Observer%3CT%3E)) 方法来移除这些观察者。

当您更新存储在 `LiveData` 对象中的值时，它会触发所有已注册的观察者（只要附加的 `LifecycleOwner` 处于活跃状态）。

LiveData 允许界面控制器观察者订阅更新。当 `LiveData` 对象存储的数据发生更改时，界面会自动更新以做出响应。

## 创建 LiveData 对象

LiveData 是一种可用于任何数据的封装容器，其中包括可实现 `Collections` 的对象，如 `List`。`LiveData` 对象通常存储在 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 对象中，并可通过 getter 方法进行访问

```java
public class NameViewModel extends ViewModel {

    // Create a LiveData with a String
    private MutableLiveData<String> currentName;
    public MutableLiveData<String> getCurrentName() {
        if (currentName == null) {
            currentName = new MutableLiveData<String>();
        }
        return currentName;
    }

    // Rest of the ViewModel...
}
```

**注意**：请确保用于更新界面的 `LiveData` 对象存储在 `ViewModel` 对象中，而不是将其存储在 activity 或 fragment 中，原因如下：

- 避免 Activity 和 Fragment 过于庞大。现在，这些界面控制器负责显示数据，但不负责存储数据状态。
- 将 `LiveData` 实例与特定的 Activity 或 Fragment 实例分离开，并使 `LiveData` 对象在配置更改后继续存在。

## 观察 LiveData 对象

在大多数情况下，应用组件的 `onCreate()` 方法是开始观察 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 对象的正确着手点，原因如下：

- 确保系统不会从 Activity 或 Fragment 的 `onResume()` 方法进行冗余调用。
- 确保 activity 或 fragment 变为活跃状态后具有可以立即显示的数据。一旦应用组件处于 [`STARTED`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle.State?hl=zh-cn#STARTED) 状态，就会从它正在观察的 `LiveData` 对象接收最新值。只有在设置了要观察的 `LiveData` 对象时，才会发生这种情况。

通常，**LiveData 仅在数据发生更改时才发送更新，并且仅发送给活跃观察者。此行为的一种例外情况是，观察者从非活跃状态更改为活跃状态时也会收到更新**。此外，如果观察者第二次从非活跃状态更改为活跃状态，则只有在自上次变为活跃状态以来值发生了更改时，它才会收到更新。

```java
public class NameActivity extends AppCompatActivity {
    private NameViewModel model;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // Other code to setup the activity...

        // Get the ViewModel.
        model = new ViewModelProvider(this).get(NameViewModel.class);

        // Create the observer which updates the UI.
        final Observer<String> nameObserver = new Observer<String>() {
            @Override
            public void onChanged(@Nullable final String newName) {
                // Update the UI, in this case, a TextView.
                nameTextView.setText(newName);
            }
        };

        // Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        model.getCurrentName().observe(this, nameObserver);
    }
}
```

## 更新 LiveData 对象

LiveData 没有公开可用的方法来更新存储的数据。[`MutableLiveData`](https://developer.android.com/reference/androidx/lifecycle/MutableLiveData?hl=zh-cn) 类将公开 [`setValue(T)`](https://developer.android.com/reference/androidx/lifecycle/MutableLiveData?hl=zh-cn#setValue(T)) 和 [`postValue(T)`](https://developer.android.com/reference/androidx/lifecycle/MutableLiveData?hl=zh-cn#postValue(T)) 方法，如果您需要修改存储在 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=zh-cn) 对象中的值，则必须使用这些方法。通常情况下会在 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=zh-cn) 中使用 `MutableLiveData`，然后 `ViewModel` 只会向观察者公开不可变的 `LiveData` 对象。

**注意**：您必须调用 [`setValue(T)`](https://developer.android.com/reference/androidx/lifecycle/MutableLiveData?hl=zh-cn#setValue(T)) 方法以从主线程更新 `LiveData` 对象。如果在工作器线程中执行代码，您可以改用 [`postValue(T)`](https://developer.android.com/reference/androidx/lifecycle/MutableLiveData?hl=zh-cn#postValue(T)) 方法来更新 `LiveData` 对象。


