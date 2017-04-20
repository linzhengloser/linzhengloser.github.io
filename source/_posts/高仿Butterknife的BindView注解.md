---
title: 高仿Butterknife的BindView注解
date: 2017-04-20 11:26:12
tags: 注解
---
# 介绍
前面介绍了 ButterKnife 的基本原理，这里就分析下它是怎么实现动态生成 ViewBinding 类的。
大致原理原理在项目编译的时候，解析注解，然后使用 JavaFileObject 这个类生成对应的java文件。

<!-- more -->

# 准备
整个流程可以分为4个 Module ，分别如下：

1. viewinject-annotation (JavaLibrary) 用来定义注解
2. viewinject-compiler (JavaLibrary) 用来解析注解并生成对应的类
3. viewinject-api (AndroidLibrary) 面向Android提供调用的类
4. viewinject-sample(Android) 普通安卓项目,测试使用

注: viewinject-annotaion 和 viewinject-compiler 可以定义成一个 JavaLibrary 但是为了将模块划分的更清楚，分开定义最好。

# show me the code
## 定义注解

viewinject-annotation -> Bind.java
``` java

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface Bind {
    int value();
}

```

## 创建注解解析器
注解解析器需要用到 Google 的 AutoService，在 build.gradle 中添加如下依赖:

compile 'com.google.auto.service:auto-service:1.0-rc2'
//依赖上面的 module
compile project (':viewinject-annotation')


然后新建一个类，类名随意。然后继承 AbstractProcessor，并使用 @AutoService(Processor.class) 注解修饰，具体代码如下。

``` java

package com.example.ioc;

import com.google.auto.service.AutoService;

import com.lz.ioc.Bind;

import java.io.IOException;
import java.io.Writer;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Map;
import java.util.Set;

import javax.annotation.processing.AbstractProcessor;
import javax.annotation.processing.Messager;
import javax.annotation.processing.ProcessingEnvironment;
import javax.annotation.processing.Processor;
import javax.annotation.processing.RoundEnvironment;
import javax.lang.model.element.Element;
import javax.lang.model.element.ElementKind;
import javax.lang.model.element.TypeElement;
import javax.lang.model.element.VariableElement;
import javax.lang.model.util.Elements;
import javax.tools.Diagnostic;
import javax.tools.JavaFileObject;

@AutoService(Processor.class)
public class ViewInjectProcessor extends AbstractProcessor {

    //日志相关的工具类
    private Messager messager;
    //元素相关的工具类 可以获取指定的元素
    private Elements elementUtils;

    //用来存放代理类对象的Map  ClassName - ProxyInfo
    private Map<String, ProxyInfo> mProxyMap = new HashMap<>();

    /**
     * 初始化方法
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        messager = processingEnv.getMessager();
        elementUtils = processingEnv.getElementUtils();
    }

    /**
     * 获取支持注解的类型
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        HashSet<String> supportTypes = new LinkedHashSet<>();
        supportTypes.add(Bind.class.getCanonicalName());
        return supportTypes;
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        messager.printMessage(Diagnostic.Kind.NOTE, "process...");
        mProxyMap.clear();
        Set<? extends Element> elesWithBind = roundEnv.getElementsAnnotatedWith(Bind.class);
        for (Element element : elesWithBind) {
            //检查注解是否合法
            checkAnnotationValid(element, Bind.class);
            VariableElement variableElement = (VariableElement) element;
            TypeElement classElement = (TypeElement) variableElement.getEnclosingElement();
            String fqClassName = classElement.getQualifiedName().toString();
            ProxyInfo proxyInfo = mProxyMap.get(fqClassName);
            if (proxyInfo == null) {
                proxyInfo = new ProxyInfo(elementUtils, classElement);
                mProxyMap.put(fqClassName, proxyInfo);
            }
            Bind bindAnnotation = variableElement.getAnnotation(Bind.class);
            int id = bindAnnotation.value();
            proxyInfo.injectVariables.put(id, variableElement);
        }

        for (String key : mProxyMap.keySet()) {
            ProxyInfo proxyInfo = mProxyMap.get(key);
            try {
                JavaFileObject jfo = processingEnv.getFiler().createSourceFile(proxyInfo.getProxyClassFullName(), proxyInfo.getTypeElement());
                Writer writer = jfo.openWriter();
                writer.write(proxyInfo.generateJavaCode());
                writer.flush();
                writer.close();
            } catch (IOException e) {
                error(proxyInfo.getTypeElement(),
                        "Unable to write injector for type %s: %s",
                        proxyInfo.getTypeElement(), e.getMessage());
            }
        }
        return true;
    }

    private boolean checkAnnotationValid(Element annotatedElement, Class clazz) {
        if (annotatedElement.getKind() != ElementKind.FIELD) {
            error(annotatedElement, "%s must be declared on field.", clazz.getSimpleName());
            return false;
        }
        if (ClassValidator.isPrivate(annotatedElement)) {
            error(annotatedElement, "%s() must can not be private.", annotatedElement.getSimpleName());
            return false;
        }

        return true;
    }

    private void error(Element element, String message, Object... args) {
        if (args.length > 0) {
            message = String.format(message, args);
        }
        processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, message, element);
    }
}


```

1. 在 init 方法中初始化一些类，比如 messager(用来输出日志) 和 elementUtils(用来解析元素)。
2. 在 getSupportedAnnotationTypes 方法中是用来返回该注解解析器所支持的注解类型。
3. 具体逻辑就是process方法，下面我们具体分析这个方法

###process方法

process方法的实现主要分为2个部分，首先收集信息，并把信息封装成对象并存起来。然后巡回集合生成对应的java文件。

收集信息
``` java
//因为 Proccess 会调用多次 所以在这里先 clear
mProxyMap.clear();
//获取所有被 Bind 注解标识的元素
Set<? extends Element> elesWithBind = roundEnv.getElementsAnnotatedWith(Bind.class);
for (Element element : elesWithBind) {
    VariableElement variableElement = (VariableElement) element;
    TypeElement classElement = (TypeElement) variableElement.getEnclosingElement();
    String fqClassName = classElement.getQualifiedName().toString();
    ProxyInfo proxyInfo = mProxyMap.get(fqClassName);
    if (proxyInfo == null) {
        proxyInfo = new ProxyInfo(elementUtils, classElement);
        mProxyMap.put(fqClassName, proxyInfo);
    }
    //获取到Bind注解 并获奖里面设置的值添加到ProxyInfo中
    Bind bindAnnotation = variableElement.getAnnotation(Bind.class);
    int id = bindAnnotation.value();
    proxyInfo.injectVariables.put(id, variableElement);
}

```

从上面的代码中分析得出获取需要用到的信息，如使用 @Bind 注解的类的完全限定名和 @Bind 注解中的控件 ID 等等。然后封装成ProxyInfo并添加到 Map 中。

ProxyInfo类其实对应的其实就是我们生成的类。下面来看看里面是怎么弄的。
``` java

package com.example.ioc;

import java.util.HashMap;
import java.util.Map;

import javax.lang.model.element.PackageElement;
import javax.lang.model.element.TypeElement;
import javax.lang.model.element.VariableElement;
import javax.lang.model.util.Elements;

/**
 * <pre>
 *     author : linzheng
 *     e-mail : 1007687534@qq.com
 *     time   : 2017/04/20
 *     desc   : 用于封装代理类对象
 *     version: 1.0
 * </pre>
 */
public class ProxyInfo {

    //生成类的包名
    private String packageName;

    //生成类的类名
    private String proxyClassName;

    //使用 @Bind 类的元素对象
    private TypeElement typeElement;

    //viewid - View变量
    //R.id.tv_content -  TextView tv_content
    public Map<Integer, VariableElement> injectVariables = new HashMap<>();

    public static final String PROXY = "ViewInject";

    public ProxyInfo(Elements elementUtils, TypeElement classElement) {
        this.typeElement = classElement;
        PackageElement packageElement = elementUtils.getPackageOf(classElement);
        //获取包名 如:ioc_apt_demo
        String packageName = packageElement.getQualifiedName().toString();
        String className = ClassValidator.getClassName(classElement, packageName);
        this.packageName = packageName;
        this.proxyClassName = className + "$$" + PROXY;
    }

    /**
     * 生成Java代码
     * @return
     */
    public String generateJavaCode() {
        StringBuilder builder = new StringBuilder();
        builder.append("// Generated code. Do not modify!\n");
        builder.append("package ").append(packageName).append(";\n\n");
        builder.append("import com.lz.viewinject_api.*;\n");
        builder.append('\n');

        builder.append("public class ").append(proxyClassName).append(" implements " + ProxyInfo.PROXY + "<" + typeElement.getQualifiedName() + ">");
        builder.append(" {\n");
        generateMethods(builder);
        builder.append('\n');
        builder.append("}\n");
        return builder.toString();
    }

    /**
     * 生成方法代码
     * @param builder
     */
    private void generateMethods(StringBuilder builder) {
        builder.append("@Override\n ");
        builder.append("public void inject(" + typeElement.getQualifiedName() + " host, Object source ) {\n");
        for (int id : injectVariables.keySet()) {
            VariableElement element = injectVariables.get(id);
            //View 的变量名称
            String name = element.getSimpleName().toString();
            //View 的 类型
            String type = element.asType().toString();
            //下面对上面的 source 的类型进行判断 ,来进行不同的类型强转
            builder.append(" if(source instanceof android.app.Activity){\n");
            builder.append("host." + name).append(" = ");
            builder.append("(" + type + ")(((android.app.Activity)source).findViewById( " + id + "));\n");
            builder.append("\n}else{\n");
            builder.append("host." + name).append(" = ");
            builder.append("(" + type + ")(((android.view.View)source).findViewById( " + id + "));\n");
            builder.append("\n};");
        }
        builder.append("  }\n");


    }

    public String getProxyClassFullName() {
        return packageName + "." + proxyClassName;
    }

    public TypeElement getTypeElement() {
        return typeElement;
    }

}


```

上面是就是ProxyInfo对象的具体实现，中而言之就是把生成的代理类生成与之对应的对象，后面就直接操作ProxyInfo对象来生成代理类。具体代码如下

```java
//ViewInjectProcessor.java -> process()
for (String key : mProxyMap.keySet()) {
        ProxyInfo proxyInfo = mProxyMap.get(key);
        try {
            //这和JavaFileObject类是系统提供的方便我们创建.java的文件
            JavaFileObject jfo = processingEnv.getFiler().createSourceFile(proxyInfo.getProxyClassFullName(), proxyInfo.getTypeElement());
            Writer writer = jfo.openWriter();
            //将生成的 javacode write到Writer对象中
            writer.write(proxyInfo.generateJavaCode()+"//key = "+key);
            writer.flush();
            writer.close();
        } catch (IOException e) {
            error(proxyInfo.getTypeElement(),
                    "Unable to write injector for type %s: %s",
                    proxyInfo.getTypeElement(), e.getMessage());
        }
}

```

到此生成的代理类的代码就分析完毕了，但是到这个还没完，因为我们Android界面里面不知道怎么调用，因为是在编译的时候生成的，这里就使用了一个接口在做连接。具体看下面代码。

``` java

package com.lz.viewinject_api;

/**
 * <pre>
 *     author : linzheng
 *     e-mail : 1007687534@qq.com
 *     time   : 2017/04/20
 *     desc   : 提供给Proxy类实现的接口
 *     version: 1.0
 * </pre>
 */
public interface ViewInject<T> {
    void inject(T t,Object source);
}

package com.lz.viewinject_api;

import android.app.Activity;
import android.view.View;

/**
 * <pre>
 *     author : linzheng
 *     e-mail : 1007687534@qq.com
 *     time   : 2017/04/20
 *     desc   : View注入
 *     version: 1.0
 * </pre>
 */
public class ViewInjector{

    private static final String SUFFIX = "$$ViewInject";

    public static void injectView(Activity activity)
    {
        ViewInject proxyActivity = findProxyActivity(activity);
        //调用 inject 方法
        proxyActivity.inject(activity, activity);
    }

    public static void injectView(Object object,View view)
    {
        ViewInject proxyActivity = findProxyActivity(object);
        //调用 inject 方法
        proxyActivity.inject(object, view);
    }

    private static ViewInject findProxyActivity(Object activity)
    {
        try
        {
            //因为我们生成的代理实现了 ViewInject 接口
            //所以这里可以直接用反射获取并强转成 ViewInject 类型
            Class clazz = activity.getClass();
            Class injectorClazz = Class.forName(clazz.getName() + SUFFIX);
            return (ViewInject) injectorClazz.newInstance();
        } catch (ClassNotFoundException e)
        {
            e.printStackTrace();
        } catch (InstantiationException e)
        {
            e.printStackTrace();
        } catch (IllegalAccessException e)
        {
            e.printStackTrace();
        }
        throw new RuntimeException(String.format("can not find %s , something when compiler.", activity.getClass().getSimpleName() + SUFFIX));
    }

}


```

因为代码是动态生成的,所以我们在项目里面并不知道类名，所以我们需要写一个接口，然后我们调用接口，代理类实现这个接口。具体代码如下:

```java

// Generated code. Do not modify!
package com.lz.ioc_apt_demo;

import com.lz.viewinject_api.ViewInject;

//这就是我们通过上面的注解解析器生成的代理列 可以看到 实现了 ViewInject 接口
public class MainActivity$$ViewInject implements ViewInject<com.lz.ioc_apt_demo.MainActivity> {
    @Override
    public void inject(com.lz.ioc_apt_demo.MainActivity host, Object source) {
        if (source instanceof android.app.Activity) {
            host.tv_textview = (android.widget.TextView) (((android.app.Activity) source).findViewById(2131427415));

        } else {
            host.tv_textview = (android.widget.TextView) (((android.view.View) source).findViewById(2131427415));

        };
    }

}

```

#总结

到此为止，整个代码逻辑就分析完了。总的来说还是很简单，而且这里只实现了最基本的功能，其实还可以对反射生成的类做一个缓冲，避免反复使用反射带来的性能开销，相比较 ButterKnife 大体的实现逻辑就这样的，只不过 ButterKnife 还实现了很多别的注解，有但是实现的原理都是差不多的。