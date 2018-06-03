---
layout: post
title: Android Hack(2)：快速找到view所在的xml文件
---

### 前记

对于一个Android开发，时不时会有这样的需求：想知道一个页面上的某些view元素是由哪些xml布局资源文件加载而来。如果这个页面是你开发的，那你应该很熟悉这其中涉及到的xml文件，你可以快速准确地找到它们。如果页面不是你开发的呢？幸运的话，你刚好认识相关的开发，而且TA的记性比较好，你可以直接询问TA资源文件名。然而现实是大多数情况下，你需要自己动手。寻找相关xml文件的过程并不总是简单省时的，于是我想能不能找到方法解决这个小小的痛点。

<!-- more -->

 一开始我尝试使用`AnnotationProcessor`来做文本解析，但会遗漏很多情况（因为我只解析标注了`@Override`的函数）。也想过是否可以通过AOP或者Android Studio插件的方式来实现，但这些方法太复杂了，性价比不高。

后续的调研的过程中，我在StackOverflow上搜到了这样一个[问题](https://stackoverflow.com/questions/32870969/is-there-an-android-tool-to-find-the-name-of-a-layout-for-a-running-app)。
![]({{ site.url }}/lib/images/layout_indicator_stackoverflow_question.png)

看来这个问题对这位朋友是一个很大的痛点。其中的一个回答给了我很大启发。
![]({{ site.url }}/lib/images/layout_indicator_stackoverflow_answer.png)

该回答提到的[ResourceInspector](https://github.com/nekocode/ResourceInspector)采用的方法是替换`Activity`本身的`LayoutInflater`，并利用了Facebook开源的调试神器Stetho来展示当前`Activity`涉及的xml布局资源文件。
![]({{ site.url }}/lib/images/layout_indicator_resourceinspector.png)

但是这样一来就得引入一个新的库。有没有更加简便优雅的方式？经过探索，我找到了方法可以在Layout Inspector的截屏里直接查看view是从哪个xml加载而来的。
![]({{ site.url }}/lib/images/layout_indicator_layoutinspector.png)



### 实现

实现这个功能的核心是要用一个代理`LayoutInflater`替换`Activity`本身的`LayoutInflater`. 首先创建一个代理`LayoutInflater`的类命名为`LayoutIndicatorInflater`.

```
public class LayoutIndicatorInflater extends LayoutInflater {

    private LayoutInflater mOriginalInflater;
    private String mAppPackageName;

    protected LayoutIndicatorInflater(LayoutInflater original, Context newContext) {
        super(original, newContext);
        mOriginalInflater = original;
        mAppPackageName = getContext().getPackageName();
    }

    @Override
    public LayoutInflater cloneInContext(Context newContext) {
        return new LayoutIndicatorInflater(mOriginalInflater.cloneInContext(newContext), newContext);
    }

    @Override
    public void setFactory(Factory factory) {
        super.setFactory(factory);
        mOriginalInflater.setFactory(factory);
    }

    @Override
    public void setFactory2(Factory2 factory) {
        super.setFactory2(factory);
        mOriginalInflater.setFactory2(factory);
    }

    @Override
    public View inflate(int resourceId, ViewGroup root, boolean attachToRoot) {
        Resources res = getContext().getResources();

        String packageName = "";
        try {
            packageName = res.getResourcePackageName(resourceId);
        } catch (Exception e) {}


        String resName = "";
        try {
            resName = res.getResourceEntryName(resourceId);
        } catch (Exception e) {}

        View view = mOriginalInflater.inflate(resourceId, root, attachToRoot);

        if (!mAppPackageName.equals(packageName)) {
            return view;
        }

        View targetView = view;
        if (root != null && attachToRoot) {
            targetView = root.getChildAt(root.getChildCount() - 1);
        }

        targetView.setContentDescription("资源文件名:" + resName);

        if (targetView instanceof ViewGroup) {
            ViewGroup viewGroup = (ViewGroup) targetView;
            for (int i = 0; i < viewGroup.getChildCount(); i++) {
                View child = viewGroup.getChildAt(i);
                if (TextUtils.isEmpty(child.getContentDescription())) {
                    child.setContentDescription("资源文件名:" + resName);
                }
            }
        }

        return view;
    }
}
```

`LayoutIndicatorInflater`的构造函数需要两个参数`LayoutInflater original, Context newContext`，其中`original`就是`Activity`本身的`LayoutInflater`，我们要用它来做实际的加载xml的工作。

主要来看看关键的`inflate`函数。

```
Resources res = getContext().getResources();

String packageName = "";
try {
    packageName = res.getResourcePackageName(resourceId);
} catch (Exception e) {}


String resName = "";
try {
    resName = res.getResourceEntryName(resourceId);
} catch (Exception e) {}
```

这一段做了两件事：第一拿到参数`resourceId`所在的包名，第二拿到`resourceId`对应的资源文件名。取包名是因为我只关心自己应用的xml，后面会根据这个包名做一个过滤处理。

```
View view = mOriginalInflater.inflate(resourceId, root, attachToRoot);
```

接着直接调用`mOriginalInflater`的`inflate`函数来加载xml。到这里就可以明白为什么`LayoutIndicatorInflater`只是一个代理了。`LayoutIndicatorInflater`只是拦截了页面里的`inflate`函数调用，记录下我们关心的xml资源文件名。真正加载xml的工作还是交给`Activity`本身的`LayoutInflater`.

```
if (!mAppPackageName.equals(packageName)) {
    return view;
}
```
这个`if`语句就是前面所说用来过滤包名的。

```
View targetView = view;
if (root != null && attachToRoot) {
    targetView = root.getChildAt(root.getChildCount() - 1);
}

targetView.setContentDescription("资源文件名:" + resName);
```

这里的`targetView`就是xml里的根元素。现在面临的问题是：把资源文件名这个信息记录在哪里？又要如何呈现？当然这里可以直接输出一条log。但是当页面比较复杂时，log就会令人眼花缭乱。经过尝试我发现view的`ContentDescription`属性可以直接在Layout Inspector的截屏里面展示，而且把资源文件名设置到`ContentDescription`也不会影响程序的逻辑。

```
if (targetView instanceof ViewGroup) {
    ViewGroup viewGroup = (ViewGroup) targetView;
    for (int i = 0; i < viewGroup.getChildCount(); i++) {
        View child = viewGroup.getChildAt(i);
        if (TextUtils.isEmpty(child.getContentDescription())) {
            child.setContentDescription("资源文件名:" + resName);
        }
    }
}
```

最后如果`targetView`是一个`ViewGroup`，那么将资源文件名也设置到`targetView`所有第一级子view的`ContentDescription`上。

创建了代理`LayoutInflater`之后，还要解决另一个关键问题：怎么用代理`LayoutInflater`替换`Activity`本身的`LayoutInflater`.  要解决这个问题需要先弄明白`Activity`本身的`LayoutInflater`从何而来。一般而言加载xml有以下几种方法：

1. `Activity.setContentView(...)`
2. `LayoutInflater.from(context).inflate(...)`
3. `Activity.getLayoutInflater().inflate(...)`

先看第一种情况。`Activity.setContentView(...)`会调用`PhoneWindow.setContentView(...)`，最后会调用`PhoneWindow`中的成员`mLayoutInflater`的`inflate`方法。

对于第二种情况，假定参数`context`是一个`Activity`. `LayoutInflater.from(context)`返回的是`context.getSystemService(Context.LAYOUT_INFLATER_SERVICE)`拿到的`LayoutInflater`对象。当这里的`context`是一个`Activity`时，`getSystemService(Context.LAYOUT_INFLATER_SERVICE)`返回的是`Activity`继承自父类`ContextThemeWrapper`的成员`mInflater`. 

最后一种情况，`Activity.getLayoutInflater()`直接返回对应`PhoneWindow`中的成员`mLayoutInflater`.

由此可以得出结论：接下来需要做两件事，第一替换`Activity`继承自父类`ContextThemeWrapper`的成员`mInflater`；第二替换`Activity`对应`PhoneWindow`中的成员`mLayoutInflater`.

替换的时机当然是越早越好，而且需要对每一个创建的`Activity`进行替换。这里`Application`的`ActivityLifecycleCallbacks`就派上了用场。这个工作交给一个工具类来做。

```
public class LayoutIndicatorHelper {

    public static void init(Application application) {
        if (BuildConfig.DEBUG) {
            application.registerActivityLifecycleCallbacks(new Application.ActivityLifecycleCallbacks() {
                @Override
                public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
                    try {
                        // Replace Activity's LayoutInflater
                        Field inflaterField = ContextThemeWrapper.class.getDeclaredField("mInflater");
                        inflaterField.setAccessible(true);
                        LayoutInflater inflater = (LayoutInflater) inflaterField.get(activity);
                        LayoutInflater proxyInflater = null;
                        if (inflater != null) {
                            proxyInflater = new LayoutIndicatorInflater(inflater, activity);
                            inflaterField.set(activity, proxyInflater);
                        }
                        
                        // Replace the LayoutInflater of Activity's Window
                        Class phoneWindowClass = Class.forName("com.android.internal.policy.PhoneWindow");
                        Field phoneWindowInflater = phoneWindowClass.getDeclaredField("mLayoutInflater");
                        phoneWindowInflater.setAccessible(true);
                        inflater = (LayoutInflater) phoneWindowInflater.get(activity.getWindow());
                        if (inflater != null && proxyInflater != null) {
                            phoneWindowInflater.set(activity.getWindow(), proxyInflater);
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

                @Override
                public void onActivityStarted(Activity activity) {

                }

                @Override
                public void onActivityResumed(Activity activity) {

                }

                @Override
                public void onActivityPaused(Activity activity) {

                }

                @Override
                public void onActivityStopped(Activity activity) {

                }

                @Override
                public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

                }

                @Override
                public void onActivityDestroyed(Activity activity) {

                }
            });
        }
    }

}
```

最后在`Application`的`onCreate`里调用一下`LayoutIndicatorHelper.init(this);`. 

大功告成！

实现原理就是这样，有几个注意事项需要说明一下。

1. 如果`Activity.setContentView`在`super.onCreate`之前调用，那该`Activity`对应的xml文件名就拿不到了。原因就是xml的加载发生在`Activity.setContentView`里，而`LayoutInflater`的替换发生在`super.onCreate`里。
2. xml里面包含的`<include>`标签指向的资源文件名此方法是拿不到的。这是因为`LayoutInflater`在加载`<include>`标签指向的资源文件时并不会递归调用`inflate`方法，也就意味着我们的代理监听不到`<include>`资源的加载。
3. xml里的根元素是`<merge>`的时候，文件名只会被记录到该xml包含的最后一个view上，如下图所示。
![](https://upload-images.jianshu.io/upload_images/12444548-1f0ed91d2253da3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
4. 调用`LayoutInflater.from(context)`时传入的`context`是非`Activity`对象，那么相应的xml是拿不到的。考虑到绝大多数情况下`context`都是`Activity`对象，这个case基本可以忽略不计了。

### 后记

有意思的是，调研过程中我在Google Groups上搜到了这么一篇帖子：
![]({{ site.url }}/lib/images/layout_indicator_googlegroup_question.png)

下面有一个疑似Google工程师给出了一个答复：
![]({{ site.url }}/lib/images/layout_indicator_googlegroup_reply.png)


现在Android Studio 3.1正式版已经发布，然而并没有包含该功能（看来if possible没有成立）。如果能做到点击view直接跳转相关的xml，那就真的完美了。

