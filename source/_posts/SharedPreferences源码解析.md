---
title: SharedPreferences源码分析
date: 2017-10-13 10:12:58
tags: 源码解析
---

# 简介
SharedPreferences(以下简称 SP) ，是 Android 系统提供的一个类，用来帮助开发者存储一些配置信息。使用 Key - Value 的形式存储，存储的文件格式为 XML。

<!-- more -->

# 使用方法

SP 的用法很简单，只要有 context 对象的地方，就可以通过 context 对象拿到 SP 对象。代码如下:

``` java

context.getSharedPreferencese("name",MODE);

```

当我们拿到 SP 对象后，就可以对 SP 对象进行获取操作，代码如下：

``` java

sp.getInt("key",defalutValue);
sp.getString("key",defalutValue);
sp.getXXX("key",defalutValue);

```

SP 对象仅仅只提供了获取数据的操作，如果如果我们需要对 SP 中的数据进行修改，那么需要使用 Editor 对象，代码如下：

``` java

SharedPreferences.Editor edit = sp.edit();
edit.putString("key","value");
edit.putInt("key",0);
edit.commit();

```

通过 SP 对象可以获取到 Editor 对象，然后在通过 Editor 对象我们可以对 SP 中的数据进行修改操作，修改完成之后需要调用 Editor 的 commit() 方法 或者是 apply() 方法，至于这两个方法有什么区别，我们后面在分析。

# 源码分析

接下来我们就来分析下 SP 的源码，看看它是怎么实现对文件的读写操作的。

SharedPreferences 是一个 interface，其具体实现叫做 SharedPreferencesImpl。

我们从 Context 的 getSharedPreferences() 方法作为入口，来分析 SP 的实现原理。代码如下：

``` java

//ContextImpl.java

private ArrayMap<String, File> mSharedPrefsPaths;

public SharedPreferences getSharedPreferences(String name, int mode) {
    // 如果当前 Android 版本是大于 4.4，那么允许使用 null 作为 文件名称，
    // 即最终生成的文件名称为 null.xml
    if (mPackageInfo.getApplicationInfo().targetSdkVersion <
            Build.VERSION_CODES.KITKAT) {
        if (name == null) {
            name = "null";
        }
    }

    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            file = getSharedPreferencesPath(name);
            mSharedPrefsPaths.put(name, file);
        }
    }
    return getSharedPreferences(file, mode);
}

```

可以看到，mSharedPrefsPaths 是一个 ArrayMap，其实就是一个 HashMap， Key 是我们传入的 name，而 Value 是对应的 File 对象，而这个 File 对象其实就是 name.xml。那么默认第一次调用此方法，File 肯定为空，这个时候会去调用 getSharedPreferencesPath() 方法，代码如下：

``` java

public File getSharedPreferencesPath(String name) {
    return makeFilename(getPreferencesDir(), name + ".xml");
}

//获取 data/data/包名/shared/ 目录的 File 对象。
private File getPreferencesDir() {
    synchronized (mSync) {
        if (mPreferencesDir == null) {
            mPreferencesDir = new File(getDataDir(), "shared_prefs");
        }
        return ensurePrivateDirExists(mPreferencesDir);
    }
}

//判断文件名称是否合法
private File makeFilename(File base, String name) {
    if (name.indexOf(File.separatorChar) < 0) {
        return new File(base, name);
    }
    throw new IllegalArgumentException(
            "File " + name + " contains a path separator");
}

```

从上面的 getSharedPreferencesPath() 方法可以得出结论，name.xml 文件存在的目录为 data/data/包名/shared_refs/name.xml。
重新回到 getSharedPreferences() 方法，拿到 File 对象之后，会调用 getSharedPreferences(File,Mode)。代码如下： 

``` java

//ContextImpl.java

private static ArrayMap<String, ArrayMap<File, SharedPreferencesImpl>> sSharedPrefsCache;

public SharedPreferences getSharedPreferences(File file, int mode) {
    checkMode(mode);
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
        getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
        // If somebody else (some other process) changed the prefs
        // file behind our back, we reload it.  This has been the
        // historical (if undocumented) behavior.
        sp.startReloadIfChangedUnexpectedly();
    }
    return sp;
}

private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
    if (sSharedPrefsCache == null) {
        sSharedPrefsCache = new ArrayMap<>();
    }

    final String packageName = getPackageName();
    ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
    if (packagePrefs == null) {
        packagePrefs = new ArrayMap<>();
        sSharedPrefsCache.put(packageName, packagePrefs);
    }

    return packagePrefs;
}


//对 mode 进行检查，如果 SDK 版本大于等于 7.0
//那么 MODE_WORLD_READABLE 和 MODE_WORLD_WRITEABLE 这两种模式就会不支持。
private void checkMode(int mode) {
    if (getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.N) {
        if ((mode & MODE_WORLD_READABLE) != 0) {
            throw new SecurityException("MODE_WORLD_READABLE no longer supported");
        }
        if ((mode & MODE_WORLD_WRITEABLE) != 0) {
            throw new SecurityException("MODE_WORLD_WRITEABLE no longer supported");
        }
    }
}


```

可以看到，sSharedPrefsCache 是一个静态的 ArrayMap，Key 为 包名，Value 也是一个 ArrayMap，而这个 ArrayMap 的 Key 为 传入的 File 对象，Value 为 SharedPreferencesImpl 对象。上面我们提到这个 SharedPreferencesImpl 其实就是 SharedPreferences 的实现类。

通过上面的分析，我们就获取到了 SP 对象，通过调用 SP 对象的 getXXX 方法能够获取到对应的数据，我们来看看其内部是怎么实现的，代码如下：

``` java

//SharedPreferencesImpl.java
//构造函数
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file);
    mMode = mode;
    mLoaded = false;
    mMap = null;
    startLoadFromDisk();
}

private void startLoadFromDisk() {
    synchronized (this) {
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}

static File makeBackupFile(File prefsFile) {
    return new File(prefsFile.getPath() + ".bak");
}

```

首先看看 SharedPreferencesImpl 的构造函数，makeBackupFile() 方法创建了一个备份文件的 File 对象。然后在看 startLoadFromDisk() 方法，在这里方法中创建了一个子线程，并在子线程中调用 loadFromDisk() 方法。代码如下：

``` java

//SharedPreferencesImpl.java

private void loadFromDisk() {
    synchronized (SharedPreferencesImpl.this) {
        if (mLoaded) {
            return;
        }
        //判断备份文件是否存在 如果存在直接使用备份文件
        if (mBackupFile.exists()) {
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    Map map = null;
    StructStat stat = null;
    try {
        stat = Os.stat(mFile.getPath());
        if (mFile.canRead()) {
            BufferedInputStream str = null;
            try {
                str = new BufferedInputStream(
                        new FileInputStream(mFile), 16*1024);
                //解析 xml 文件并装换成 map 对象。
                map = XmlUtils.readMapXml(str);
            } catch (XmlPullParserException | IOException e) {
                Log.w(TAG, "getSharedPreferences", e);
            } finally {
                IoUtils.closeQuietly(str);
            }
        }
    } catch (ErrnoException e) {
        /* ignore */
    }

    synchronized (SharedPreferencesImpl.this) {
        mLoaded = true;
        if (map != null) {
            mMap = map;
            mStatTimestamp = stat.st_mtime;
            mStatSize = stat.st_size;
        } else {
            mMap = new HashMap<>();
        }
        notifyAll();
    }
}

```

loadFromDisk() 方法主要逻辑就是解析 xml 并装换成 map 对象。在方法最后面，调用了 notifyAll()，为什么要调用这个方法？后面会有解释。

通过上面对 SP 构造函数的分析，我们可以得出结论，SP 在 new 的时候会使用一个子线程去加载本地的 xml 文件，然后转换成 map 存放在内存中，既然是异步获取的，那么 SP 的 getXXX 方法是怎么保证数据同步获取的呢？代码如下：

``` java

//SharedPreferencesImpl.java

@Nullable
public String getString(String key, @Nullable String defValue) {
    synchronized (this) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    if (!mLoaded) {
        // Raise an explicit StrictMode onReadFromDisk for this
        // thread, since the real read will be in a different
        // thread and otherwise ignored by StrictMode.
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

我们只看 getString() 方法，其他 getXXX() 方法大同小异，首先会调用 awaitLoadedLocked() 方法，在 awaitLoadedLocked() 会进入一个 while 循环，如果数据没有加载完毕，这里会一直调用 wait() 方法，直到 mLoaded == true 才会跳出循环，然后才会从 mMap 中获取对应的数据。

上面说到在 loadFromDisk() 方法最后调用了 notifyAll() 方法，我猜就和这里的 wait() 方法有关，用来处理多线程调用 getXXX() 方法的时候出现的问题。

看完 getXXX() 方法之后我们在来看看那 SP 的 Editor，看看它是怎么实现数据的写入的。代码如下：

``` java

//SharedPreferencesImpl.java

public final class EditorImpl implements Editor {

    private final Map<String, Object> mModified = Maps.newHashMap();

    public Editor putString(String key, @Nullable String value) {
        synchronized (this) {
            mModified.put(key, value);
            return this;
        }
     }

    public boolean commit() {
        MemoryCommitResult mcr = commitToMemory();
        SharedPreferencesImpl.this.enqueueDiskWrite(
            mcr, null /* sync write on this thread okay */);
        try {
            mcr.writtenToDiskLatch.await();
        } catch (InterruptedException e) {
            return false;
        }
        notifyListeners(mcr);
        return mcr.writeToDiskResult;
    }

    private MemoryCommitResult commitToMemory() {
        MemoryCommitResult mcr = new MemoryCommitResult();
        synchronized (SharedPreferencesImpl.this) {
            // We optimistically don't make a deep copy until
            // a memory commit comes in when we're already
            // writing to disk.
            if (mDiskWritesInFlight > 0) {
                // We can't modify our mMap as a currently
                // in-flight write owns it.  Clone it before
                // modifying it.
                // noinspection unchecked
                mMap = new HashMap<String, Object>(mMap);
            }
            mcr.mapToWriteToDisk = mMap;
            mDiskWritesInFlight++;

            boolean hasListeners = mListeners.size() > 0;
            if (hasListeners) {
                mcr.keysModified = new ArrayList<String>();
                mcr.listeners =
                        new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
            }

            synchronized (this) {
                if (mClear) {
                    if (!mMap.isEmpty()) {
                        mcr.changesMade = true;
                        mMap.clear();
                    }
                    mClear = false;
                }

                for (Map.Entry<String, Object> e : mModified.entrySet()) {
                    String k = e.getKey();
                    Object v = e.getValue();
                    // "this" is the magic value for a removal mutation. In addition,
                    // setting a value to "null" for a given key is specified to be
                    // equivalent to calling remove on that key.
                    if (v == this || v == null) {
                        if (!mMap.containsKey(k)) {
                            continue;
                        }
                        mMap.remove(k);
                    } else {
                        //判断是否有值被修改
                        if (mMap.containsKey(k)) {
                            Object existingValue = mMap.get(k);
                            if (existingValue != null && existingValue.equals(v)) {
                                continue;
                            }
                        }
                        //修改内存中的 mMap
                        mMap.put(k, v);
                    }

                    //标记有值修改过
                    mcr.changesMade = true;
                    if (hasListeners) {
                        mcr.keysModified.add(k);
                    }
                }

                mModified.clear();
            }
        }
        return mcr;
    }


}

```

在 commit() 方法中，首先会调用 commitToMemory()，这个方法看上去很长，其实就干了一件事情，就是判断我们在 Editor 调用 putXXX 的值和内存中的 mMap 中的值是否有修改，如果有修改，就写入到内存中并返回 true，如果没有修改就返回 false。

内存的数据写入完毕之后，肯定还需要把数据写入到磁盘中。代码如下：

``` java

//SharedPreferencesImpl.java

private void enqueueDiskWrite(final MemoryCommitResult mcr,
                              final Runnable postWriteRunnable) {
    //用于执行将数据写入磁盘操作的 Runnable 对象
    final Runnable writeToDiskRunnable = new Runnable() {
            public void run() {
                synchronized (mWritingToDiskLock) {
                    writeToFile(mcr);
                }
                synchronized (SharedPreferencesImpl.this) {
                    mDiskWritesInFlight--;
                }
                if (postWriteRunnable != null) {
                    postWriteRunnable.run();
                }
            }
        };

    //true 表示同步提交 false 表示异步提交
    final boolean isFromSyncCommit = (postWriteRunnable == null);

    // Typical #commit() path with fewer allocations, doing a write on
    // the current thread.
    if (isFromSyncCommit) {
        boolean wasEmpty = false;
        synchronized (SharedPreferencesImpl.this) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) {
            writeToDiskRunnable.run();
            return;
        }
    }

    QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
}

```

enqueueDiskWrite() 方法会通过判断 postWriteRunnable 参数是否为 null，来决定是否使用异步执行写入磁盘的操作。很明显，上面的 commit() 方法是同步写入磁盘。

在开始的时候，我们说到 SP 除了 commit() 方法可以将数据写入磁盘以外，还有一个叫做 apply() 的方法也有同样的作用，下面我们来看看它是怎么做的，代码如下：

``` java

//SharedPreferencesImpl.java

public void apply() {
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            public void run() {
                try {
                    mcr.writtenToDiskLatch.await();
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

    // Okay to notify the listeners before it's hit disk
    // because the listeners should always get the same
    // SharedPreferences instance back, which has the
    // changes reflected in memory.
    notifyListeners(mcr);
}

```

可以看到，apply() 方法和 commit() 方法没有什么大的区别，只不过在调用 enqueueDiskWrite() 方法的时候，传入了 postWriteRunnable 对象，这样就可以实现异步的提交。

根据官方文档上的描述，如果不关心提交数据后的返回值(即提交后数据是否有变化)，推荐使用 apply() 方法。

SP 还提供了监听数据更新的方法，代码如下：

``` java

SharedPreferences preferences = MyApplication.getInstance().getSharedPreferences("sp", MODE_PRIVATE);
preferences.registerOnSharedPreferenceChangeListener(this);
preferences.unregisterOnSharedPreferenceChangeListener(this);

@Override
public void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key) {
    //数据更新
}

```

# 总结

分析到这，SP 的源码就看的差不多了，可以发现,SP 的设计初衷解释为了实现小数据的频繁读写，而且还处理了多线程的问题，不过如果数据过大，在读取和写入的时候会降低性能。