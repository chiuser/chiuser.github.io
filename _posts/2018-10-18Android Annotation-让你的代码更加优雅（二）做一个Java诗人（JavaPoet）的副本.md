---
layout: post
title: "做一个Java诗人（JavaPoet）"
subtitle: "Android Annotation-让你的代码更加优雅（二）"
author: "小铭"
header-img: "img/Annotation + Java"
header-mask: 0.4
tags:
  - Android
  - JAVA
---

# Android Annotation-让你的代码更加优雅（二）做一个Java诗人（JavaPoet）

# 上篇回顾

上一篇我们按照思维导图，介绍了注解的基础知识，如何定义一个注解，提示性注解，运行时注解的写法和用法。没有看过第一篇，又对注解知识相对陌生的同学，建议先食用第一篇。本篇将重点介绍编译期注解，自动生成Java文件相关内容。第一篇传送门：

[Android Annotation-让你的代码更加优雅（一）](https://juejin.im/post/5bc3fe59e51d451a3f4c5db8)

# 本篇食用路线

照例，这里先给出本篇的学习导图。方便大家掌握学习大纲。本章照例会先给出一些用来处理编译期注解的基础类和方法，然后通过一些具体的例子学习如何利用编译期注解来实现一些便捷功能。

![篇二](/Users/CongMing/GitHub/chiuser.github.io/img/篇二.png)

# 编译期静态处理-做一个Java诗人

## JavaPoet简介

JavaPoet是square公司的开源库，传送门见下面。从名字就可以看出，Java诗人，即JavaPoet是一个通过注解生成java文件的库。我们可以利用注解，运用JavaPoet来生成一些重复的模板代码。从而大大提高我们编程效率。像我们熟知的ButterKnife，就是通过这种方法来简化代码编写的。在JavaPoet使用过程中，也需要用到一些Java API，我们会在后文一并讲解。

[Github-JavaPoet](https://github.com/square/javapoet)

使用时，引入依赖就可以了：

```java
compile 'com.squareup:javapoet:1.7.0'
```



## “诗人”眼中的结构化Java文件

了解编译原理的同学都知道，在编译器眼中，代码文件其实就是按一定语法编写的结构化数据。编译器在处理Java文件时，也是按照既定的语法，分析Java文件的结构化数据。结构化数据就是我们日常编写Java文件时用到的基本元素。在Java中，对于编译器来说代码中的元素结构是基本不变的，例如组成代码的基本元素包括包、类、函数、字段、变量等，JDK为这些元素定义了一个基类也就是Element类，我们用Element类及其子类来表示这些基本元素，Element共用5个子类：

|         类名         | 表达的元素                                                   |
| :------------------: | :----------------------------------------------------------- |
|    PackageElement    | 表示一个包程序元素，可以获取到包名等                         |
|     TypeElement      | 表示一个类或接口程序元素                                     |
|   VariableElement    | 表示一个字段、enum 常量、方法或构造方法参数、局部变量、类成员变量或异常参数 |
|  ExecutableElement   | 表示某个类或接口的方法、构造方法或初始化程序（静态或实例），包括注解类型元素 |
| TypeParameterElement | 表示一般类、接口、方法或构造方法元素的泛型参数               |

通过一个例子来明确一下：

```java
package com.xm.test;    // 包名，PackageElement

public class Test<      // 类名，TypeElement
    T                   // 泛型参数，TypeParameterElement
    > {     

    private int a;       // 成员变量，VariableElement
    private Test other;  // 成员变量，VariableElement

    public Test () {}    // 成员方法，ExecuteableElement
    public void setA (   // 成员方法，ExecuteableElement
                     int newA       // 方法参数，VariableElement
    ) {
        String test;     // 局部变量，VariableElement
    }
}
```

当编译器操作Java文件中的元素时，就是通过上面这些类来进行操作的。即我们想通过JavaPoet来生成Java文件时，就可以使用这些子类来表达结构化程序的元素。任何一个Element类对象，都可以根据实际情况，强转成对应的子类。而Element类，实际上是一个接口，它定义了一套方法，我们来一起看一下。

```java
public interface Element extends AnnotatedConstruct {  
    /** 
     * 返回此元素定义的类型 
     * 例如，对于一般类元素 Clazz<P extends People>，返回参数化类型 Clazz<P> 
     */  
    TypeMirror asType();  
  
    /** 
     * 返回此元素的种类：包、类、接口、方法、字段...,如下枚举值 
     * PACKAGE, ENUM, CLASS, ANNOTATION_TYPE, INTERFACE, ENUM_CONSTANT, FIELD, PARAMETER, LOCAL_VARIABLE, EXCEPTION_PARAMETER, 
     * METHOD, CONSTRUCTOR, STATIC_INIT, INSTANCE_INIT, TYPE_PARAMETER, OTHER, RESOURCE_VARIABLE; 
     */  
    ElementKind getKind();  
  
    /** 
     * 返回此元素的修饰符,如下枚举值 
     * PUBLIC, PROTECTED, PRIVATE, ABSTRACT, DEFAULT, STATIC, FINAL, 
     * TRANSIENT, VOLATILE, SYNCHRONIZED, NATIVE, STRICTFP; 
     */  
    Set<Modifier> getModifiers();  
  
    /** 
     * 返回此元素的简单名称,例如 
     * 类型元素 java.util.Set<E> 的简单名称是 "Set"； 
     * 如果此元素表示一个未指定的包，则返回一个空名称； 
     * 如果它表示一个构造方法，则返回名称 "<init>"； 
     * 如果它表示一个静态初始化程序，则返回名称 "<clinit>"； 
     * 如果它表示一个匿名类或者实例初始化程序，则返回一个空名称 
     */  
    Name getSimpleName();  
  
    /** 
     * 返回封装此元素的最里层元素。 
     * 如果此元素的声明在词法上直接封装在另一个元素的声明中，则返回那个封装元素； 
     * 如果此元素是顶层类型，则返回它的包； 
     * 如果此元素是一个包，则返回 null； 
     * 如果此元素是一个泛型参数，则返回 null. 
     */  
    Element getEnclosingElement();  
  
    /** 
     * 返回此元素直接封装的子元素 
     */  
    List<? extends Element> getEnclosedElements();  
    
    boolean equals(Object var1);

    int hashCode();
  
    /** 
     * 返回直接存在于此元素上的注解 
     * 要获得继承的注解，可使用 getAllAnnotationMirrors 
     */  
    List<? extends AnnotationMirror> getAnnotationMirrors();  
  
    /** 
     * 返回此元素针对指定类型的注解（如果存在这样的注解），否则返回 null。注解可以是继承的，也可以是直接存在于此元素上的 
     */   
    <A extends Annotation> A getAnnotation(Class<A> annotationType); 
     
    <R, P> R accept(ElementVisitor<R, P> var1, P var2);
}
```

## “诗人”的大脑-APT(Annotation Processor Tool)注解处理器

APT，说其是诗人的大脑，是因为我们整个代码生成任务的核心，需要用到注解的处理器提供的方法和入口。在整个流程的核心部分，都是由APT来实现和控制的。APT是一种处理注解的工具，确切的说它是javac的一个工具，它用来在编译时扫描和处理注解，一个注解的注解处理器，以java代码(或者编译过的字节码)作为输入，生成.java文件作为输出，核心是交给自己定义的处理器去处理。实际上，APT在编译期留出了一个供我们编程的一套模板接口。我们通过实现处理器中的方法，就可以编写自己的注解处理流程了。

### APT核心-AbstractProcessor

每个自定义的注解处理器，都要继承虚处理器AbstractProcessor，来实现其几个关键方法。

虚处理器AbstractProcessor的几个关键方法：

```java
public class MyProcessor extends AbstractProcessor {
    @Override
    public synchronized void init(ProcessingEnvironment env){ }

    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

    @Override
    public Set<String> getSupportedAnnotationTypes() { }

    @Override
    public SourceVersion getSupportedSourceVersion() { }
}
```

当我们实现自定义的注解处理器时，上述的这几个方法，是必须要实现的。下面重点介绍一下这四个方法：

- `init(ProcessingEnvironment env)`：每一个注解处理器类都必须有一个空的构造函数。然而，这里有一个特殊的init()方法，它会被注解处理工具调用，并输入ProcessingEnviroment参数。ProcessingEnviroment提供很多有用的工具类，如Elements, Types和Filer。

- `process(Set<? extends TypeElement> annotations, RoundEnvironment env)`：这相当于每个处理器的主函数main()。你在这里写你的扫描、评估和处理注解的代码，以及生成Java文件。输入参数RoundEnviroment，可以让你查询出包含特定注解的被注解元素。这是一个布尔值，表明注解是否已经被处理器处理完成，官方原文whether or not the set of annotations are claimed by this processor，通常在处理出现异常直接返回false、处理完成返回true。

- `getSupportedAnnotationTypes()`：必须要实现；用来表示这个注解处理器是注册给哪个注解的。返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。

- `getSupportedSourceVersion()`：用来指定你使用的Java版本。通常这里返回SourceVersion.latestSupported()，你也可以使用SourceVersion_RELEASE_6、7、8注册处理器版本。

由于注解处理器是javac的工具，因此我们必须将自定义的处理器注册到javac中，方法是我们需要提供一个.jar文件，打包你的注解处理器到此文件中，并且在jar中，需要打包一个特定的文件 javax.annotation.processing.Processor到META-INF/services路径下 。而这一切都是极为繁琐的。**幸好**谷歌给我们开发了AutoService注解，你只需要引入这个依赖，然后在你的处理器类上加上注解：

```java
@AutoService(Processor.class)
```

然后我们就可以自动生成文件，并打包进jar中。省去了很多麻烦事儿。

那么上面我们介绍完处理器相关内容，下面我们再来看一看APT还为我们提供了哪些其它工具。

### APT提供的四个辅助工具

这四个工具，我们通过在AbstractProcessor的实现类中，通过ProcessingEnvironment即可获得：

```java
    private Filer mFiler;
    private Elements mElementUtils;
    private Messager mMessager;
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        mElementUtils = processingEnv.getElementUtils();
        mMessager = processingEnv.getMessager();
        mFiler = processingEnv.getFiler();
    }
```



#### Filer

从名字看得出来，与文件相关的操作会用到。一般配合JavaPoet来生成Java文件

#### Messager

它提供给注解处理器一个报告错误、警告信息的途径。当我们自定义的注解处理器运行时报错时，那么运行注解处理器的JVM也会崩溃，打印出一些不容易被应用开发者读懂的日志信息。这时，我们可以借助Messager输出一些调试信息，以更直观的方式提示程序运行的错误。

#### Types

Types是一个用来操作TypeMirror的工具。TypeMirror是Element中通过adType()方法得到的一个对象。它保存了元素的具体信息，比如Element是一个类，那么其成员详细信息就保存在TypeMirror中。

#### Elements

Elements是一个用来处理Element的工具。这里不详细展开了。用到的时候会提到。

## “诗人”的工具箱

JavaPoet为我们提供了编译期通过操作Java文件结构元素，依据注解生成Java文件的便捷方法。那么如何来生成呢？我们先有必要来了解一下JavaPoet为我们提供了哪些工具。

### JavaPoet为我们提供的四个表达Java文件元素的常用类

这些用来表达Java文件元素的类，其实和上面说的Element有异曲同工之妙。现在没法具体理解没关系，后面有例子。

|      类名      |              含义              |
| :------------: | :----------------------------: |
|   MethodSpec   |   代表一个构造函数或方法声明   |
|    TypeSpec    | 代表一个类，接口，或者枚举声明 |
|   FieldSpec    | 代表一个成员变量，一个字段声明 |
| ParameterSpec  |  代表一个参数，可用来生成参数  |
| AnnotationSpec |          代表一个注解          |
|    JavaFile    |    包含一个顶级类的Java文件    |

## “诗人”实战

我们先来整体看一个简单的例子，然后再拓展到各种情况。

### 一个例子

假如我们定义了如下一个注解，运用上一篇我们学过的知识：

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.TYPE)
public @interface Xnpe {
    String value();
}
```

接下来实现注解处理器：

```java
@AutoService(Processor.class)
public class XnpeProcess extends AbstractProcessor {

    private Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        filer = processingEnv.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (TypeElement element : annotations) {
            if (element.getQualifiedName().toString().equals(Xnpe.class.getCanonicalName())) {
                MethodSpec main = MethodSpec.methodBuilder("main")
                        .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
                        .returns(void.class)
                        .addParameter(String[].class, "args")
                        .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
                        .build();
                TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
                        .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                        .addMethod(main)
                        .build();

                try {
                    JavaFile javaFile = JavaFile.builder("com.xm", helloWorld)
                            .addFileComment(" This codes are generated automatically. Do not modify!")
                            .build();
                    javaFile.writeTo(filer);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return true;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(Xnpe.class.getCanonicalName());
        return annotations;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}
```

这里需要一点耐心了，乍看起来有点多，但实际上比较简单。这里我们总结出实现自定义注解处理器的几个关键步骤：

1. 为注解处理器增加@AutoService注解。即@AutoService(Processor.java)
2. 实现上文说的自定义注解常用到的四个方法，即init()、process()、getSupportedAnnotationTypes和getSupportedSourceVersion。
3. 编写处理注解的逻辑。

本例中，我们先来重点看第二条，即四个大方法的实现。重点在处理方法上，即process()方法。我们拿出其中的核心部分做一个讲解。

```java
MethodSpec main = MethodSpec.methodBuilder("main") //MethodSpec当然是methodBuilder，即创建方法。
    .addModifiers(Modifier.PUBLIC, Modifier.STATIC)//增加限定符
    .returns(void.class)                           //指定返回值类型
    .addParameter(String[].class, "args")          //指定方法的参数
    .addStatement("$T.out.println($S)", System.class, "Hello, I am Poet!")//添加逻辑代码。
    .build();                                      //创建
TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld") //TypeSpec构建Class
    .addModifiers(Modifier.PUBLIC, Modifier.FINAL) //增加限定符
    .addMethod(main)                               //将刚才创建的main方法添加进类中。
    .build();                                      //创建
```

是不是流程上很容易理解。MethodSpec是用来生成方法的，详细解释可参加代码上的注释。

细心的你也许注意到了，代码中有些$T等字样的东东，这个又是什么呢？下面我们通过几个小例子，一方面来了解一下Poet中的一些占位符，另一方面也熟悉一下常用的方法。

### 常用方法

##### `addCode`与`addStatement`用来增加代码

```
MethodSpec main = MethodSpec.methodBuilder("main")
    .addCode(""
        + "int total = 0;\n"
        + "for (int i = 0; i < 10; i++) {\n"
        + "  total += i;\n"
        + "}\n")
    .build();
```

生成的是

```
void main() {
  int total = 0;
  for (int i = 0; i < 10; i++) {
    total += i;
  }
}
```

- `addCode`用于增加极简代码。即代码中仅包含纯Java基础类型和运算。
- `addStatement`用于增加一些需要import方法的代码。如上面的`.addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")` 就需要使用`.addStatement`来声明。

##### beginControlFlow和endControlFlow，流控方法

流控方法主要用来实现一些流控代码的添加，比上面的add方法看着美观一点。比如上面的代码，可以改写为：

```java
MethodSpec main = MethodSpec.methodBuilder("main")
    .addStatement("int total = 0")
    .beginControlFlow("for (int i = 0; i < 10; i++)")
    .addStatement("total += i")
    .endControlFlow()
    .build();
```

### 占位符

#### $L字面量（Literals）

```java
private MethodSpec computeRange(String name, int from, int to, String op) {
  return MethodSpec.methodBuilder(name)
      .returns(int.class)
      .addStatement("int result = 0")
      .beginControlFlow("for (int i = $L; i < $L; i++)", from, to)
      .addStatement("result = result $L i", op)
      .endControlFlow()
      .addStatement("return result")
      .build();
}
```

当我们传参调用时，`coputeRange("test", 0, 10, "+")`它能生成的代码如下：

```java
int test(){
    int result = 0;
    for(int i = 0; i < 10; i++) {
        result = result + i;
    }
    return result;
}
```

#### $S 字符串常量（String）

这个比较容易理解，这里就不赘述了。

#### $T 类型(Types)

```java
MethodSpec today = MethodSpec.methodBuilder("today")
    .returns(Date.class)
    .addStatement("return new $T()", Date.class)
    .build(); //创建today方法
TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
    .addMethod(today)
    .build(); //创建HelloWorld类
JavaFile javaFile = JavaFile.builder("com.xm.helloworld", helloWorld).build();
javaFile.writeTo(System.out);//写java文件
```

生成的代码如下，我们看到，它会自动导入所需的包。这也是我们使用占位符的好处，也是使用JavaPoet的一大好处。

```java
package com.xm.helloworld;

import java.util.Date;

public final class HelloWorld {
  Date today() {
    return new Date();
  }
}
```

如果我们想要导入自己写的类怎么办？上面的例子是传入系统的class，这里也提供一种方式，通过ClassName.get（”类的路径”，”类名“），结合$T可以生成

```java
ClassName testClass = ClassName.get("com.xm", "TestClass");
ClassName list = ClassName.get("java.util", "List");
ClassName arrayList = ClassName.get("java.util", "ArrayList");
TypeName listOftestClasses = ParameterizedTypeName.get(list, testClass);

MethodSpec xNpe = MethodSpec.methodBuilder("xNpe")
    .returns(listOftestClasses)
    .addStatement("$T result = new $T<>()", listOftestClasses, arrayList)
    .addStatement("result.add(new $T())", testClass)
    .addStatement("result.add(new $T())", testClass)
    .addStatement("result.add(new $T())", testClass)
    .addStatement("return result")
    .build();
```

生成的代码如下：

```java
package com.xm.helloworld;

import com.xm.TestClass;
import java.util.ArrayList;
import java.util.List;

public final class HelloWorld {
  List<TestClass> xNpe() {
    List<TestClass> result = new ArrayList<>();
    result.add(new TestClass());
    result.add(new TestClass());
    result.add(new TestClass());
    return result;
  }
}
```

Javapoet 同样支持import static，通过`addStaticImport`来添加：

```java
JavaFile.builder("com.xm.helloworld", hello)
    .addStaticImport(TestClass, "START")
    .addStaticImport(TestClass2, "*")
    .addStaticImport(Collections.class, "*")
    .build();
```

#### $N 命名(Names)

通常指我们自己生成的方法名或者变量名等等。比如这样的代码：

```java
public String byteToHex(int b) {
  char[] result = new char[2];
  result[0] = hexDigit((b >>> 4) & 0xf);
  result[1] = hexDigit(b & 0xf);
  return new String(result);
}

public char hexDigit(int i) {
  return (char) (i < 10 ? i + '0' : i - 10 + 'a');
}
```

这个例子中，我们在byteToHex中需要调用到hexDigit方法，我们就可以用$N来表示这种引用。然后通过传递方法名，达到这种调用语句的生成。

```java
MethodSpec hexDigit = MethodSpec.methodBuilder("hexDigit")
    .addParameter(int.class, "i")
    .returns(char.class)
    .addStatement("return (char) (i < 10 ? i + '0' : i - 10 + 'a')")
    .build();

MethodSpec byteToHex = MethodSpec.methodBuilder("byteToHex")
    .addParameter(int.class, "b")
    .returns(String.class)
    .addStatement("char[] result = new char[2]")
    .addStatement("result[0] = $N((b >>> 4) & 0xf)", hexDigit)
    .addStatement("result[1] = $N(b & 0xf)", hexDigit)
    .addStatement("return new String(result)")
    .build();
```

### 自顶向下，构建Java类的各种元素

#### 普通方法

我们在定义方法时，也要对方法增加一些修饰符，如`Modifier.ABSTRACT`。可以通过`addModifiers()`方法：

```java
MethodSpec test = MethodSpec.methodBuilder("test")
    .addModifiers(Modifier.ABSTRACT, Modifier.PROTECTED)
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addMethod(test)
    .build();
```

将会生成如下代码：

```java
public abstract class HelloWorld {
  protected abstract void test();
}
```

#### 构造器

构造器只不过是一个特殊的方法，因此可以使用`MethodSpec`来生成构造器方法。使用`constrctorBuilder`来生成：

```java
MethodSpec flux = MethodSpec.constructorBuilder()
    .addModifiers(Modifier.PUBLIC)
    .addParameter(String.class, "greeting")
    .addStatement("this.$N = $N", "greeting", "greeting")
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC)
    .addField(String.class, "greeting", Modifier.PRIVATE, Modifier.FINAL)
    .addMethod(flux)
    .build();
```

将会生成代码：

```java
public class HelloWorld {
  private final String greeting;

  public HelloWorld(String greeting) {
    this.greeting = greeting;
  }
}
```

#### 各种参数

参数也有自己的一个专用类`ParameterSpec`，我们可以使用`ParameterSpec.builder()`来生成参数，然后MethodSpec的addParameter去使用，这样更加优雅。

```java
ParameterSpec android = ParameterSpec.builder(String.class, "android")
    .addModifiers(Modifier.FINAL)
    .build();

MethodSpec welcomeOverlords = MethodSpec.methodBuilder("test")
    .addParameter(android)
    .addParameter(String.class, "robot", Modifier.FINAL)
    .build();
```

生成的代码

```java
void test(final String android, final String robot) {
}
```

#### 字段,成员变量

可以使用FieldSpec去声明字段，然后加到类中：

```java
FieldSpec android = FieldSpec.builder(String.class, "android")
    .addModifiers(Modifier.PRIVATE, Modifier.FINAL)
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC)
    .addField(android)
    .addField(String.class, "robot", Modifier.PRIVATE, Modifier.FINAL)
    .build();
```

生成代码：

```java
public class HelloWorld {
  private final String android;
  private final String robot;
}
```

通常Builder可以更加详细的创建字段的内容，比如javadoc、annotations或者初始化字段参数等，如：

```java
FieldSpec android = FieldSpec.builder(String.class, "android")
    .addModifiers(Modifier.PRIVATE, Modifier.FINAL)
    .initializer("$S + $L", "Pie v.", 9.0)//初始化赋值
    .build();
```

对应生成的代码：

```java
private final String android = "Pie v." + 9.0;
```

#### 接口

接口方法必须是PUBLIC ABSTRACT并且接口字段必须是PUBLIC STATIC FINAL ，使用`TypeSpec.interfaceBuilder`如下：

```java
TypeSpec helloWorld = TypeSpec.interfaceBuilder("HelloWorld")
    .addModifiers(Modifier.PUBLIC)
    .addField(FieldSpec.builder(String.class, "KEY_START")
        .addModifiers(Modifier.PUBLIC, Modifier.STATIC, Modifier.FINAL)
        .initializer("$S", "start")
        .build())
    .addMethod(MethodSpec.methodBuilder("beep")
        .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
        .build())
    .build();
```

生成的代码如下：

```java
public interface HelloWorld {
  String KEY_START = "start";
  void beep();
}
```

#### 枚举类型

使用`TypeSpec.enumBuilder`来创建，使用`addEnumConstant`来添加枚举值：

```java
TypeSpec helloWorld = TypeSpec.enumBuilder("Roshambo")
    .addModifiers(Modifier.PUBLIC)
    .addEnumConstant("ROCK")
    .addEnumConstant("SCISSORS")
    .addEnumConstant("PAPER")
    .build();
```

生成的代码

```java
public enum Roshambo {
  ROCK,
  SCISSORS,
  PAPER
}
```

#### 匿名内部类

需要使用`Type.anonymousInnerClass("")`,通常可以使用$L占位符来指代：

```java
TypeSpec comparator = TypeSpec.anonymousClassBuilder("")
    .addSuperinterface(ParameterizedTypeName.get(Comparator.class, String.class))
    .addMethod(MethodSpec.methodBuilder("compare")
        .addAnnotation(Override.class)
        .addModifiers(Modifier.PUBLIC)
        .addParameter(String.class, "a")
        .addParameter(String.class, "b")
        .returns(int.class)
        .addStatement("return $N.length() - $N.length()", "a", "b")
        .build())
    .build();

TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
    .addMethod(MethodSpec.methodBuilder("sortByLength")
        .addParameter(ParameterizedTypeName.get(List.class, String.class), "strings")
        .addStatement("$T.sort($N, $L)", Collections.class, "strings", comparator)
        .build())
    .build();
```

生成代码：

```java
void sortByLength(List<String> strings) {
  Collections.sort(strings, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
      return a.length() - b.length();
    }
  });
}
```

定义匿名内部类的一个特别棘手的问题是参数的构造。在上面的代码中我们传递了不带参数的空字符串。`TypeSpec.anonymousClassBuilder("")`。

#### 注解

注解使用起来比较简单，通过addAnnotation就可以添加：

```java
MethodSpec toString = MethodSpec.methodBuilder("toString")
    .addAnnotation(Override.class)
    .returns(String.class)
    .addModifiers(Modifier.PUBLIC)
    .addStatement("return $S", "Hello XiaoMing")
    .build();
```

生成代码

```java
@Override
public String toString() {
  return "Hello XiaoMing";
}
```

通过`AnnotationSpec.builder()` 可以对注解设置属性：

```java
MethodSpec logRecord = MethodSpec.methodBuilder("recordEvent")
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addAnnotation(AnnotationSpec.builder(Headers.class)
        .addMember("accept", "$S", "application/json; charset=utf-8")
        .addMember("userAgent", "$S", "Square Cash")
        .build())
    .addParameter(LogRecord.class, "logRecord")
    .returns(LogReceipt.class)
    .build();
```

代码生成如下

```java
@Headers(
    accept = "application/json; charset=utf-8",
    userAgent = "Square Cash"
)
LogReceipt recordEvent(LogRecord logRecord);
```



```java
MethodSpec logRecord = MethodSpec.methodBuilder("recordEvent")
    .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
    .addAnnotation(AnnotationSpec.builder(HeaderList.class)
        .addMember("value", "$L", AnnotationSpec.builder(Header.class)
            .addMember("name", "$S", "Accept")
            .addMember("value", "$S", "application/json; charset=utf-8")
            .build())
        .addMember("value", "$L", AnnotationSpec.builder(Header.class)
            .addMember("name", "$S", "User-Agent")
            .addMember("value", "$S", "Square Cash")
            .build())
        .build())
    .addParameter(LogRecord.class, "logRecord")
    .returns(LogReceipt.class
    .build
```

### 生成Java文件

生成Java文件，我们需要用到上文提到的Filer和Elements。注意下面这段代码，重要的是包名，类名的指定。这里生成的文件名，一般会遵循某个约定，以便事先写好反射代码。

```java
//获取待生成文件的包名
public String getPackageName(TypeElement type) {
    return mElementUtils.getPackageOf(type).getQualifiedName().toString();
}

//获取待生成文件的类名
private static String getClassName(TypeElement type, String packageName) {
    int packageLen = packageName.length() + 1;
    return type.getQualifiedName().toString().substring(packageLen).replace('.', '$');
}

//生成文件
private void writeJavaFile() {
    String packageName = getPackageName(mClassElement);
    String className = getClassName(mClassElement, packageName);
    ClassName bindClassName = ClassName.get(packageName, className);
    TypeSpec finderClass = TypeSpec.classBuilder(bindClassName.simpleName() + "$$Injector")
        .addModifiers(Modifier.PUBLIC)
        .addSuperinterface(ParameterizedTypeName.get(TypeUtil.INJECTOR,
                                                     TypeName.get(mClassElement.asType())))
        .addMethod(methodBuilder.build())
        .build();
    //使用JavaFile的builder来生成java文件
    JavaFile.builder(packageName, finderClass).build().writeTo(mFiler);
}
```



# 总结

通过两篇的学习，我们熟悉了Java注解的用途，写法，以及如何用它为我们的编码或程序服务。本篇罗列了很多具体的例子，希望能覆盖到日常大家使用的方方面面，大家也可以收藏本文，在使用JavaPoet的时候进行参照。

**小铭出品，必属精品**

欢迎关注xNPE技术论坛，更多原创干货每日推送。

![扫码_搜索联合传播样式-微信标准绿版](/Users/CongMing/Documents/%E5%8D%9A%E6%96%87%E7%B4%A0%E6%9D%90/%E7%BA%BF%E4%B8%8B%E7%89%A9%E6%96%99%E7%B4%A0%E6%9D%90/%E6%90%9C%E4%B8%80%E6%90%9C%E5%85%AC%E4%BC%97%E5%8F%B7%E6%8E%A8%E5%B9%BF%E7%89%A9%E6%96%99%E5%9B%BE%E7%89%87-png/%E6%89%AB%E7%A0%81_%E6%90%9C%E7%B4%A2%E8%81%94%E5%90%88%E4%BC%A0%E6%92%AD%E6%A0%B7%E5%BC%8F-%E5%BE%AE%E4%BF%A1%E6%A0%87%E5%87%86%E7%BB%BF%E7%89%88.png)