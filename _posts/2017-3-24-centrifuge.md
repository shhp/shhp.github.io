---
layout: post
title: 利用annotation提取代码
---

### 背景介绍

我们团队在对app进行启动性能优化后一直在思考一个问题：有没有办法在开发的时候就能知道某次代码修改有可能影响启动性能。这样就不会导致在发现启动变慢后，再去花费大量时间排查问题。

### 解决方案

受[dagger](https://github.com/square/dagger)的启发，我有了一个想法：可以利用Java的`Annotation`把跟启动有关的代码在编译时提取出来，放在一个文件中，同时将这个文件加入git版本管理。如此一来，只要查看这个文件的提交历史记录，我们就可以知道某一次的commit是否包含可能影响启动性能的代码了。这篇文章就是对我实现这个功能的概括总结。

<!-- more -->

#### 1. 定义`Annotation`

首先定义一个`Annotation`如下：

```java
@Target({ElementType.METHOD, ElementType.CONSTRUCTOR, ElementType.TYPE})
public @interface LaunchPerf {
}
```
    
在`@Target`中指定`@LaunchPerf`可以用来标注类、构造函数以及函数。

#### 2. 实现Annotation Processor

接下来实现我们自己的Annotation Processor. 我们需要创建一个`AbstractProcessor`的子类：

```java
@SupportedAnnotationTypes("com.cootek.dialer.annotation.LaunchPerf")
@SupportedSourceVersion(SourceVersion.RELEASE_7)
public class LaunchPerfAnnotationProcessor extends AbstractProcessor {...}
```

通过`@SupportedAnnotationTypes`指定了`LaunchPerfAnnotationProcessor`是用来处理之前创建的annotation `LaunchPerf`.

为了在编译时得到源代码，需要借助Compiler Tree API. 相关的代码包含在java安装目录下的`lib/tools.jar`里，因此我把`tools.jar`引入到工程里。其中的核心思想是遍历经编译得到的抽象语法树（Abstract Syntax Tree），当访问到我们需要的节点时，通过Compiler Tree API拿到相关的源代码。

在`LaunchPerfAnnotationProcessor`初始化时构造一个`Tree`节点：

```java
private Trees mTrees;
@Override
public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    mTrees = Trees.instance(processingEnv);
}
```

后面会通过`TreePathScanner`的`scan`来遍历`mTrees`.

接下来就是重写`AbstractProcessor`的`process`方法进行真正的处理了：

```java
private FileObject outputFile;
private Writer outputWriter;
private Messager mMessager;
    
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    BufferedWriter bufferedWriter = null;
    mMessager = processingEnv.getMessager();
    try {
        if (outputFile == null) {
            outputFile = processingEnv.getFiler().createResource(StandardLocation.SOURCE_OUTPUT, "launchPerf", "related_code");
            outputWriter = outputFile.openWriter();
        }
        mMessager.printMessage(Diagnostic.Kind.NOTE, "file location:"+outputFile.toUri());
        bufferedWriter = new BufferedWriter(outputWriter);
    } catch (IOException e) {
        e.printStackTrace();
    }

    StringBuilder stringBuilder = new StringBuilder();
    for (Element element : roundEnv.getElementsAnnotatedWith(LaunchPerf.class)) {
        String identifier = "// " + getElementId(element);
        stringBuilder.append(identifier).append("\n").append(getElementSourceCode(element)).append("\n\n");
        processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, identifier);
    }

    if (bufferedWriter != null) {
        try {
            bufferedWriter.append(stringBuilder.toString());
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                bufferedWriter.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    return true;
}
```
    
我们的需求是把所有相关的代码提取出来放到一个文件中，所以通过`processingEnv.getFiler().createResource`创建一个文件用以写入被`@LaunchPerf`标注的源代码。

下一步通过`roundEnv.getElementsAnnotatedWith(LaunchPerf.class)`拿到被标注了`@LaunchPerf`的`Element`. 我们关心的`Element`有三种类别：`ElementKind.CLASS`, `ElementKind.CONSTRUCTOR`和`ElementKind.METHOD`. 为了能够区分这些`Element`，我写了一个函数`getElementId`来给每一个`Element`生成唯一的标识。

```java
private String getElementId(Element element) {
    ElementKind elementKind = element.getKind();
    if (elementKind == ElementKind.CLASS) {
        return ((TypeElement)element).getQualifiedName().toString();
    } else if (elementKind == ElementKind.CONSTRUCTOR
            || elementKind == ElementKind.METHOD) {
        StringBuilder builder = new StringBuilder();
        TypeElement parent = (TypeElement) element.getEnclosingElement();
        ExecutableElement thisElement = (ExecutableElement) element;
        builder.append(parent.getQualifiedName()).append("#").append(thisElement.getSimpleName()).append("(");
        for (VariableElement parameter : thisElement.getParameters()) {
            builder.append(parameter.asType()).append(" ").append(parameter.getSimpleName()).append(",");
        }
        builder.append(")");
        return builder.toString();
    }
    return element.getSimpleName().toString();
}
```

一个类的标识就是包名+类名；构造函数和函数的标识是对应的类名+函数签名。

生成`Element`的标识之后就是要获取相关代码了。函数`getElementSourceCode`实现了此功能。

```java
private String getElementSourceCode(Element element) {
    if (element.getKind() == ElementKind.CLASS) {
        ClassScanner scanner = new ClassScanner();
        return scanner.scan(element, mTrees);
    } else if (element.getKind() == ElementKind.CONSTRUCTOR
            || element.getKind() == ElementKind.METHOD) {
        MethodScanner scanner = new MethodScanner();
        return scanner.scan(element, mTrees);
    }
    return element.getSimpleName().toString();
}
```

仍然是对类和函数进行区别对待。这里的`ClassScanner`以及`MethodScanner`都是继承自`TreePathScanner`. 先看一下`ClassScanner`的实现：

```java
public class ClassScanner extends TreePathScanner<String, Trees> {

    private List<BlockTree> mBlockTreeList = new ArrayList<>();

    public String scan(Element classElement, Trees trees) {
        if (classElement != null
                && classElement.getKind() == ElementKind.CLASS) {
            scan(trees.getPath(classElement), trees);
            StringBuilder builder = new StringBuilder();
            for (BlockTree blockTree : mBlockTreeList) {
                builder.append(blockTree).append("\n\n");
            }
            return builder.toString();
        }
        return "";
    }

    @Override
    public String visitBlock(BlockTree node, Trees trees) {
        if (node.isStatic()) {
            mBlockTreeList.add(node);
        }
        return super.visitBlock(node, trees);
    }
}
```

对于被标注了`@LaunchPerf`的类，我们关心的是它包含的`static`代码段。因此用一个`List<BlockTree>`来保存当前类中的所有`static`代码段。
在重写的`visitBlock`方法中，如果发现当前访问的`BlockTree`节点是`static`的，则加入`mBlockTreeList`. scan完成之后，遍历拿到的`BlockTree`，
再通过`BlockTree.toString()`得到对应的源码。

下面是`MethodScanner`的实现：

```java
public class MethodScanner extends TreePathScanner<String, Trees> {

    private MethodTree mMethodTree;
    private String mMethodName;

    public String scan(Element methodElement, Trees trees) {
        if (methodElement != null
                && (methodElement.getKind() == ElementKind.METHOD || methodElement.getKind() == ElementKind.CONSTRUCTOR)) {
            mMethodName = methodElement.getSimpleName().toString();
            scan(trees.getPath(methodElement), trees);
            return mMethodTree != null ? mMethodTree.getBody().toString() : "";
        }
        return "";
    }

    @Override
    public String visitMethod(MethodTree methodTree, Trees trees) {
        if (mMethodTree == null && mMethodName.equals(methodTree.getName().toString())) {
            mMethodTree = methodTree;
        }
        return super.visitMethod(methodTree, trees);
    }
}
```

在`visitMethod`中记下当前函数对应的`MethodTree`. 然后调用`MethodTree.getBody().toString()`得到函数源码。

最后把得到的所有源码写入之前创建的文件就算大功告成了！

### 更进一步

到此我们团队的需求基本得到了满足。但是是否能更进一步呢？能不能做到通过自定义的annotation实现分门别类地提取代码？比如一些核心代码可以标注`@Core`，跟UI
相关的代码用`@UI`标注......然后将这些标注过的代码提取到不同的文件中。为了实现这个想法，我将上面的工程进行了扩展。新的工程被命名为*Centrifuge*, 中文
意思是离心机。我觉得这个工具就好比代码的离心机，将不同性质的代码分离提取。源代码放在了[github](https://github.com/shhp/Centrifuge)上。接下来我就说明如何扩展
上面的工程实现*Centrifuge*.

#### 1. 允许自定义annotation

为了让使用者可以自定义annotation，我需要定义一个用来标注annotation的annotation。

```java
@Documented
@Target({ElementType.ANNOTATION_TYPE})
public @interface CodeExtractor {
}
```

通过`@Target({ElementType.ANNOTATION_TYPE})`，限制了`@CodeExtractor`只能用来标注annotation.

现在我们可以自定义一个annotation：

```java
@Target({ElementType.METHOD, ElementType.CONSTRUCTOR, ElementType.TYPE, ElementType.LOCAL_VARIABLE})
@CodeExtractor
public @interface Core {
}
```

后面就可以使用`@Core`去标注我们感兴趣的类和方法了。

#### 2. 修改annotation processor

下一步就是要修改annotation processor了。之前通过`@SupportedAnnotationTypes("com.cootek.dialer.annotation.LaunchPerf")`限定了我们的annotation processor
只是用来处理`@LaunchPerf`. 现在为了能够处理使用者自定义的annotation，需要将其修改为`@SupportedAnnotationTypes({"*"})`，这意味着新的annotation processor
接收所有的annotation.

修改后的`process`函数如下：

```java
private Map<String, FileObject> outputFiles = new HashMap<>();
private Map<String, Writer> outputWriters = new HashMap<>();
private Set<TypeElement> annotationsForExtraction = new HashSet<>();
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    mMessager = processingEnv.getMessager();
    for (TypeElement annotation : annotations) {
        if (annotation.getAnnotation(CodeExtractor.class) != null) {
            String annotationQualifiedName = String.valueOf(annotation.getQualifiedName());
            mMessager.printMessage(Diagnostic.Kind.NOTE, "annotation:"+annotationQualifiedName);
            try {
                if (!outputFiles.containsKey(annotationQualifiedName)) {
                    annotationsForExtraction.add(annotation);

                    FileObject fileObject = processingEnv.getFiler().createResource(StandardLocation.SOURCE_OUTPUT, "centrifuge", annotation.getSimpleName());
                    outputFiles.put(annotationQualifiedName, fileObject);
                    outputWriters.put(annotationQualifiedName, fileObject.openWriter());
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    for (TypeElement annotationType : annotationsForExtraction) {
        StringBuilder stringBuilder = new StringBuilder();
        for (Element element : roundEnv.getElementsAnnotatedWith(annotationType)) {
            String identifier = "// " + getElementId(element);
            stringBuilder.append(identifier).append("\n").append(getElementSourceCode(element)).append("\n\n");
            processingEnv.getMessager().printMessage(Diagnostic.Kind.NOTE, identifier);
        }
        BufferedWriter bufferedWriter = new BufferedWriter(outputWriters.get(String.valueOf(annotationType.getQualifiedName())));
        try {
            bufferedWriter.append(stringBuilder.toString());
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                bufferedWriter.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    return true;
}
```

现在需要写多个文件，所以用一个`Map`来保存。另外用一个`Set`来保存所有被标注了`@CodeExtractor`的annotation. 判定一个annotation是否被标注了`@CodeExtractor`
的方式就是`annotation.getAnnotation(CodeExtractor.class) != null`. 拿到了所有被标注了`@CodeExtractor`的annotation之后，只需遍历这些annotation，对每一个
annotation利用之前的方法进行代码提取即可。

### 来点结语

Annotation其实是很强大的。现在有不少开源库（如*Dagger*, *Retrofit*）都利用了annotation，让开发者可以快速高效地写出简洁的代码。大胆地发挥想象，说不定
annotation就能成为帮助你编程的利器！
