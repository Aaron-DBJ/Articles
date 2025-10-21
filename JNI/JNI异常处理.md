# 基础逻辑

在JNI的C++代码中，调用Java层方法时，如果Java层方法发生异常，不会立刻退出C++层后续代码的执行，而是等C++层方法执行结束回到Java层时，抛出异常。

例如以下代码

1.Java层业务代码：

```java
fun testException(): Int {
        val ret = 10/0
        return ret
    }
```

2.JNI C++代码

```cpp
extern "C"
JNIEXPORT jint JNICALL
Java_com_example_kotlindemo_jni_JniApi_testException(JNIEnv *env, jobject thiz,
                                                     jobject user_handler) {
    LOGD("Java_com_example_kotlindemo_jni_JniApi_testException 开始执行");
    jclass userHandlerClass = env->GetObjectClass(user_handler);
    jmethodID testExceptionMethod = env->GetMethodID(userHandlerClass, "testException", "()I");
    // 省略不必要代码
    ...
    // testExceptionMethod内部会发送异常
    ret = env->CallIntMethod(user_handler, testExceptionMethod);
    // 省略不必要代码
    ...    
    LOGD("Java_com_example_kotlindemo_jni_JniApi_testException 执行完毕");
    return ret;
}
```

3.最终的输出

```
Java_com_example_kotlindemo_jni_JniApi_testException 开始执行
Java_com_example_kotlindemo_jni_JniApi_testException 执行完毕
FATAL EXCEPTION: main
Process: com.example.kotlindemo, PID: 16120
java.lang.ArithmeticException: divide by zero
at com.example.kotlindemo.UserHandler.testException(UserHandler.kt:32)
at com.example.kotlindemo.jni.JniApi.testException(Native Method)
at com.example.kotlindemo.jni.JniManager.testException(JniManager.kt:38)
at com.example.kotlindemo.MainActivity.jniTest$lambda$4(MainActivity.kt:120)
at com.example.kotlindemo.MainActivity.$r8$lambda$FnfWElkiQGm5YAF5dYSzCJz6YI8(Unknown Source:0)
at com.example.kotlindemo.MainActivity$$ExternalSyntheticLambda1.run(D8$$SyntheticClass:0)
at android.os.Handler.handleCallback(Handler.java:996)
at android.os.Handler.dispatchMessage(Handler.java:110)
at android.os.Looper.loopOnce(Looper.java:210)
at android.os.Looper.loop(Looper.java:302)
at android.app.ActivityThread.main(ActivityThread.java:9652)
at java.lang.reflect.Method.invoke(Native Method)
at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:601)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1062)
```

可以发现，C++方法中虽然执行了先发生异常的Java层方法，但还是把后续C++代码执行完毕后才崩溃的。

ps：如果是JNI接口的错误，比如env->GetMethodID传入了一个不存在的函数名，那么会直接崩溃，不会执行后续C++代码。

# 异常处理

JNI开发中主要存在2类异常，一种是Java层异常（是被C++层调用的Java方法内部发生的异常），另一种则是C++异常，纯粹是因为C++代码逻辑导致的异常，对于二者，有各自的处理方式。。

## Java层异常

Java层异常处理主要是使用JNI提供的异常处理相关方法，其实不止Java层异常，前文所说的JNI接口的错误也可以被处理。

常用的JNI异常处理方法如下：

| 方法                      | 说明                                              | 备注                                                                       |
| ----------------------- | ----------------------------------------------- | ------------------------------------------------------------------------ |
| **ExceptionCheck()**    | 快速检查当前线程是否有挂起的异常，返回JNI_TRUE(有异常)或JNI_FALSE（无异常） | 性能：比 ExceptionOccurred() 更快<br/>✅推荐：大多数情况下首选此方法                          |
| **ExceptionOccurred()** | 获取当前挂起的异常对象，返回异常引用。                             | 资源管理：返回局部引用，需要手动调用 DeleteLocalRef()<br/>使用场景：需要详细处理异常对象时                 |
| **ExceptionClear()**    | 清除当前线程挂起的异常。                                    | 如果发生异常                                                                   |
| **ThrowNew()**          | 创建新的异常对象并抛出。返回值jint                             | 异常替换：如果已有挂起异常，需要先调用 ExceptionClear()<br/><br/>立即返回：调用后应立即返回，让 Java 层处理异常 |
| **Throw()**             | 抛出已有异常。返回值jint                                  | 需手动释放                                                                    |

### 检测异常并清除

如果不想处理或者不关心异常，那检测到异常直接清除并返回了，不再继续执行C++代码。例如

```cpp
extern "C"
JNIEXPORT jint JNICALL
Java_com_example_kotlindemo_jni_JniApi_testException(JNIEnv *env, jobject thiz,
                                                     jobject user_handler) {
    
    // 省略无关代码
    ...
    jint ret = 0;

    ret = env->CallIntMethod(user_handler, testExceptionMethod);
    if (env->ExceptionCheck()) {
        LOGE("=========> UserHandler testException check exception");
        env->ExceptionDescribe();
        env->ExceptionClear();
        return -1;
    }

    // 省略无关代码
    ...
    LOGD("Java_com_example_kotlindemo_jni_JniApi_testException 执行完毕");
    return ret;
}
```

通过`env->ExceptionCheck()`检测是否有异常发送，有的话通过`env->ExceptionDescribe()`打印异常信息， 最后通过`env->ExceptionClear()`清除异常，Java层就不会有异常发生了。

检测异常也可以使用`env->ExceptionOccurred()`，他返回的是异常引用，可以获取异常的详细信息。不过注意使用完后需要手动删除，不然会导致内存泄露。

```cpp
...
    jthrowable exception = env->ExceptionOccurred();
    if (exception) {
        LOGE("=========> UserHandler testException check exception");
        env->ExceptionDescribe();
        env->ExceptionClear();
        
        ...
        env->DeleteLocalRef(exception) // 手动删除
        return -1;
    }
...
```

### 转换异常

有的时候想自定义异常，可以在「检测异常并清除」这一小节的示例代码中，通过`throwNew`抛出异常，注意这种方式必须要先清除已有异常（调用`env->ExceptionClear()`方法），再抛出。示例代码

```cpp
...
    if (env->ExceptionCheck()) {
        LOGE("=========> UserHandler testException check exception");
        env->ExceptionDescribe();
        env->ExceptionClear();
        env->ThrowNew(env->FindClass("java/lang/NullPointerException"),
                      "crash cause by divide by zero");
        return -1;
    }
...
```

### JNI接口错误

如果是JNI接口的错误，比如`env->GetMethodID()`传入了错误函数名，导致找不到方法，也可以用上述方式检测处理异常。

# C++层异常

如果是C++层的异常，跟Java类似，C++也提供了try-catch的机制来捕获处理异常。注意，C++处理的异常范围或种类跟Java有较大不同，不要带着Java经验认为C++会处理这些异常（比如）
