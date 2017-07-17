---
title: SheredPreference的合理使用
date: 2017-07-13 21:54:47
tags:
---
SharedPreference（下文成sp）是Android上一种轻量级存储方式。这里针对sp所引起的严苛模式问题给出解决方法并展开一些可能引发的问题。

### 严苛模式问题 ###

严苛模式主要会造成的问题有：
- 在主线程通过commit()写入时
- 通过getXXX()的方法读取值时

解决方法
- 对于第一个问题是由于sp本质是在应用自身的data文件夹下的一个XML文件，对于commit()方法实际为IO操作，在主线程操作会卡住主线程的执行。可以使用apply()方法替换commit()方法解决。
- 对于第二个问题是由于sp还未将内容读取到缓存中，虽然其加载本身是在子线程中进行，但getXXX()方法需要等待加载sp的线程加载完成，因此会出现严苛模式问题。可以在需要使用sp的地方提前加载。之后再去调用getXXX()方法时就不会再卡住主线程。如下：

``` java
// 首先加载sp（加载本身会在子线程完成，所以可以直接将getSharedPreferences放在主线程中）
SharedPreferences sp = getSharedPreferences("XXX", MODE_PRIVETE);
//...做一堆别的事情
...
//OK，估计sp已经被加载完毕了
String testValue = sp.getString("testKey", null);
```

解决sp的严苛模式问题至此就可以结束了，下附干货。

--- 

### sp加载卡住主线程的原因 ###
下面是SharedPreferenceImpl.java中getString()方法的源码。
``` java
public String getString(String key, @Nullable String defValue) {
    synchronized (this) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```
我们继续看下awaitLoadedLocked()方法。
``` java
private void awaitLoadedLocked() {
    if (!mLoaded) {
        BlockGuard.getThreadPolicy().onReadFromDisk();
    }
    while (!mLoaded) {
        try {
            wait();
        } catch (InterruptedException unused) {
        }
    }
}
```
明显看出在sp在未被加载完成之前需要一直等待其加载。虽然主线程中没有IO操作，但由于其会卡住主线程的运行，严苛模式会主动打出onReadFromDisk的异常Log。

我们再看下getSharedPreferences()的源码会发现在该方法中会调用getSharedPreferencesCacheLocked()方法。该方法中会将整个应用中的sp全部加载到一个静态变量中缓存。即应用中所用到的sp就会一直在内存中存在。所以在使用sp的过程中，切勿将较大数据内容放入sp中存储。

### 切勿多次执行apply()方法 ###
在解决主线程写入sp时可以使用apply()方法取代commit()方法。但在执行apply()方法时还需要注意一些问题。多次执行apply()方法时也有可能会卡住主线程。
``` java
public void apply() {
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
        public void run() {
            try {
                mcr.writtenToDiskLatch().await();
            } catch (InterruptedException ignored) {
            }
        }
    };

    QueuedWork.add(awaitCommit);

    Runnable postWriteRunnable = new Runnable() {
        public void run() {
            awaitCommit.run();
            QueuedWork.remove(awaitCommit);
        }
    };

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

    notifyListeners(mcr);
}
```
在apply()方法中会将awitCommit的任务加入到一个队列之中，再将IO操作加入到单线程的线程池中执行。正常情况下该操作一切正常，不会卡住主线程。但倘若再此时Activity走到onPause、onStop，或者Service走到onStop时。ActivityThread.java执行对应的方法时会执行`QueuedWork.waitToFinish();`，会执行上面awaitCommit的内容，即等待写入线程。在这种情况下则有可能会卡住主线程。