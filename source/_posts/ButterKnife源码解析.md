---
title: ButterKnife源码解析
date: 2017-03-10 14:37:05
tags: 源码解析
---

# ButterKnife介绍
ButterKnife是Github著名Android大神JakeWharton开源的,主要用于解决Android平台上findViewById,setOnClick等重复代码,使用非常方便.

<!-- more -->

## 几年前的View注入框架
即的刚开始学Android的时候,最喜欢的框架就是xUtils,其中也有一个类似ButterKnife中的@BindView的注解,叫做@ViewInject,当时用的很爽,结果后来得知其内部使用的是反射实现的,要知道反射在Android上会大大降低APP的性能,所以果断弃之,后来听别人说ButterKnife不是用反射实现View注入的,当时好奇,决定试试这个库,然后就没有然后了.

## 反射实现的View注入

``` java

//下面代码来住xUtils中 ViewInject.java
private static void injectObject(Object handler, ViewFinder finder) {

        Class<?> handlerType = handler.getClass();

        // inject ContentView
        ContentView contentView = handlerType.getAnnotation(ContentView.class);
        if (contentView != null) {
            try {
                Method setContentViewMethod = handlerType.getMethod("setContentView", int.class);
                //反射调用setContentView
                setContentViewMethod.invoke(handler, contentView.value());
            } catch (Throwable e) {
                LogUtils.e(e.getMessage(), e);
            }
        }

        // inject view
        //获取Activity 所用公共字段
        Field[] fields = handlerType.getDeclaredFields();
        if (fields != null && fields.length > 0) {
            for (Field field : fields) {
                //处理@ViewInject注解
                ViewInject viewInject = field.getAnnotation(ViewInject.class);
                if (viewInject != null) {
                    try {
                        //
                        View view = finder.findViewById(viewInject.value(), viewInject.parentId());
                        if (view != null) {
                            field.setAccessible(true);
                            //反射设置View
                            field.set(handler, view);
                        }
                    } catch (Throwable e) {
                        LogUtils.e(e.getMessage(), e);
                    }
                } else {
                    //@ResInject 注解的相关处理
                    ResInject resInject = field.getAnnotation(ResInject.class);
                    if (resInject != null) {
                        try {
                            Object res = ResLoader.loadRes(
                                    resInject.type(), finder.getContext(), resInject.id());
                            if (res != null) {
                                field.setAccessible(true);
                                field.set(handler, res);
                            }
                        } catch (Throwable e) {
                            LogUtils.e(e.getMessage(), e);
                        }
                    } else {
                        PreferenceInject preferenceInject = field.getAnnotation(PreferenceInject.class);
                        if (preferenceInject != null) {
                            try {
                                Preference preference = finder.findPreference(preferenceInject.value());
                                if (preference != null) {
                                    field.setAccessible(true);
                                    field.set(handler, preference);
                                }
                            } catch (Throwable e) {
                                LogUtils.e(e.getMessage(), e);
                            }
                        }
                    }
                }
            }
        }

        //省略部分代码......
    }

```

>上面的代码就是xUtils中ViewInject这块代码的实现,不仅使用了反射,而且还没有做缓存处理,控件少还好,如控件多,会明显感觉卡顿,那有没有好的办法,答案肯定是有的,ButterKnife就很好的实现了.下面我们就好好分析下ButterKnife是怎么实现的.

# ButterKnife 的基本使用

``` java

class ExampleActivity extends Activity {

  @BindView(R.id.user) EditText username;
  @BindView(R.id.pass) EditText password;

  //String资源的绑定
  @BindString(R.string.login_error) String loginErrorMessage;

  //OnClick事件的绑定
  @OnClick(R.id.submit) void submit() {
    // TODO call server...
  }

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    //必须在setContentView之后调用
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}

```

使用分为两步:

1. 在需要注入的View上添加注解@BindView
2. 在onCreate的setContentView后面调用ButterKnife.bind(this);

# 源码分析

``` java

  //代码目录 ButterKnife.java
  //首先调用的就是该方法
  public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    //获取DecorView 并传入createbinding方法中
    return createBinding(target, sourceView);
  }

  //代码目录同上
  private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());

    //constructor 意为构造器对象,主要用于创建对象用,这里虽然使用了反射,但是在findBindingConstructorForClass方法中做了缓存处理,反复获取同一个只会执行一次反射.
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }

  //代码目录 同上
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {

    //先从缓存中获取
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    //如果是android开头 或者 java 开头的类 就直接返回null
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }
    try {
      // 加载类ActivityName_ViewBinding类是在编译的时候生成的
      Class<?> bindingClass = Class.forName(clsName + "_ViewBinding");
      //noinspection unchecked
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }

```

>ButterKnife的源码的核心部分大部分就是上面这些,大致步骤如下

1. 基于编译时注解解析,来生成ActivityName_ViewBinding类,类在 build/generaated/source/apt/packageName/ActivityName_ViewBinding.java 目录下.
2. 在程序运行的时候,用反射调用生成的ActivityName_ViewBinding.java,在类中调用findViewById方法.


# 总结
ButterKnife的核心是使用apt在代码编译的时候,自动生成代码,这一块涉及的内容比较多,以后在单独分析,总之只需要知道ButterKnife的实现方式就是帮你生成了findViewById方法并没有使用反射强行赋值.
