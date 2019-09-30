---
layout: post
title:  "Java并发系列-深入Jvm理解Thread启动流程"
date:   2019-07-29 20:06:06
categories: 并发
tags: Java 线程 Jvm
---

* content
{:toc}

## 前言
近期整理笔记想开个专题，准备从并发入手。并发这块又从哪里入手，一开始想的是AQS，偶然间看到自己之前编译调试openjdk的时候整理的一些笔记，又有了新的想法，决定先从最基础的开始，并发这块脱离不了线程，那么我们就结合jdk,hotspot探究一下线程的来龙去脉。
线程的定义：程序运行的最小单元，被包含在进程中。
## Java中的线程


```
class Thread implements Runnable {
    /* Make sure registerNatives is the first thing <clinit> does. */
    private static native void registerNatives();
    static {
        registerNatives();
    }
    ...//省略一大波代码，和一些英文注释
    public synchronized void start() {
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```
通常我们使用线程都是调用的start()方法；    
从上面的Thread的源码结构我们可以看到start()方法
先判断是否是NEW状态，如果不是，抛出线程状态异常；
接下来执行start0()





## Jvm中定义的线程
下面我们打开openjdk查看一下jdk中本地方法的定义，打开Thread.c

```
#include "jni.h"
#include "jvm.h"

#include "java_lang_Thread.h"

#define THD "Ljava/lang/Thread;"
#define OBJ "Ljava/lang/Object;"
#define STE "Ljava/lang/StackTraceElement;"
#define STR "Ljava/lang/String;"

#define ARRAY_LENGTH(a) (sizeof(a)/sizeof(a[0]))

static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread},
    {"isAlive",          "()Z",        (void *)&JVM_IsThreadAlive},
    {"suspend0",         "()V",        (void *)&JVM_SuspendThread},
    {"resume0",          "()V",        (void *)&JVM_ResumeThread},
    {"setPriority0",     "(I)V",       (void *)&JVM_SetThreadPriority},
    {"yield",            "()V",        (void *)&JVM_Yield},
    {"sleep",            "(J)V",       (void *)&JVM_Sleep},
    {"currentThread",    "()" THD,     (void *)&JVM_CurrentThread},
    {"countStackFrames", "()I",        (void *)&JVM_CountStackFrames},
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},
    {"holdsLock",        "(" OBJ ")Z", (void *)&JVM_HoldsLock},
    {"getThreads",        "()[" THD,   (void *)&JVM_GetAllThreads},
    {"dumpThreads",      "([" THD ")[[" STE, (void *)&JVM_DumpThreads},
    {"setNativeName",    "(" STR ")V", (void *)&JVM_SetNativeThreadName},
};

#undef THD
#undef OBJ
#undef STE
#undef STR

JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}
```
以上代码是与JNI注册相关的，很简单，有必要的话后面出一篇介绍一下；
下面我们找到start0对应的jvm中的方法，JVM_StartThread
在jvm.cpp中

```
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;
  // Heap_lock while we construct the exception.
  bool throw_illegal_thread_state = false;

  {
    MutexLocker mu(Threads_lock);

    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
      // We could also check the stillborn flag to see if this thread was already stopped, but
      // for historical reasons we let the thread detect that itself when it starts running

      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      size_t sz = size > 0 ? (size_t) size : 0;
      native_thread = new JavaThread(&thread_entry, sz);

      if (native_thread->osthread() != NULL) {
        // Note: the current thread is not being used within "prepare".
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  assert(native_thread != NULL, "Starting null thread?");

  if (native_thread->osthread() == NULL) {
    // No one should hold a reference to the 'native_thread'.
    delete native_thread;
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        "unable to create new native thread");
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              "unable to create new native thread");
  }

  Thread::start(native_thread);

JVM_END
```
以上是主要逻辑，我们接下来一步步分析

 首先是获取互斥锁
```
MutexLocker mu(Threads_lock);
```

 检查线程状态，确保是未启动状态

```
 if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
```

 上一步检查通过，则创建与jvm中的JavaThread，主要是通过构造函数来实现

```
native_thread = new JavaThread(&thread_entry, sz);
```

 看看这个构造函数的主要内容
 
```
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
  Thread()
  ...//省略
{
  if (TraceThreadEvents) {
    tty->print_cr("creating thread %p", this);
  }
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  set_entry_point(entry_point);
  // Create the native thread itself.
  // %note runtime_23
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  os::create_thread(this, thr_type, stack_sz);
```

 我们看到最后一行 os::create_thread(this, thr_type, stack_sz);这个才是真正映射到系统层面，创建线程。


```
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {
  assert(thread->osthread() == NULL, "caller responsible");

  // Allocate the OSThread object
  OSThread* osthread = new OSThread(NULL, NULL);
  if (osthread == NULL) {
    return false;
  }

  // set the correct thread state
  osthread->set_thread_type(thr_type);

  // Initial state is ALLOCATED but not INITIALIZED
  osthread->set_state(ALLOCATED);

  thread->set_osthread(osthread);
  ......//省略
    pthread_t tid;
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);

    pthread_attr_destroy(&attr);

    if (ret != 0) {
      if (PrintMiscellaneous && (Verbose || WizardMode)) {
        perror("pthread_create()");
      }
      // Need to clean up stuff we've allocated so far
      thread->set_osthread(NULL);
      delete osthread;
      if (lock) os::Linux::createThread_lock()->unlock();
      return false;
    }

    // Store pthread info into the OSThread
    osthread->set_pthread_id(tid);

```
 创建os级别的线程，与JavaThread进行绑定，接下来**划重点**
 
```
int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);
```
java_start就是新创建的线程启动入口，它会等待一个信号来调用Java中的run()方法，
jvm中具体的实现逻辑代码如下

```
  // wait until os::start_thread()
    while (osthread->get_state() == INITIALIZED) {
      sync->wait(Mutex::_no_safepoint_check_flag);
    }
  }

  // call one more level start routine
  thread->run()
```

 接下来回到JVM_StartThread方法，线程创建好了，调用native_thread->prepare(jthread);将Java 中的Thread和Jvm中的Thread进行绑定，此前还有一步跟系统级Thread绑定。

 **最后真正启动线程**

```
Thread::start(native_thread);
```
同样也是逐层调用到os级别线程，具体逻辑如下

```
void os::start_thread(Thread* thread) {
  // guard suspend/resume
  MutexLockerEx ml(thread->SR_lock(), Mutex::_no_safepoint_check_flag);
  OSThread* osthread = thread->osthread();
  osthread->set_state(RUNNABLE);
  pd_start_thread(thread);
}
```

至此，我们真正的逻辑就被系统线程调用了，当然其中还有些细节。通过这种方式我们能整体上把握线程的在Java层面，Jvm层面，Os层面的关系，好了本章就算给后续并发系列文章做个引子，在后续的文章中还会继续聊并发这块的知识。

---
  本文版权归作者本人拥有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。










