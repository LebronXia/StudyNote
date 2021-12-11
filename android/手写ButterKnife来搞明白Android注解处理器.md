Butterknife现在在项目中基本没用到了，逐渐被ViewBinding所代替，而我们所熟知它的内部原理是通过自定义注解+自定义注解解析器来动态生成代码并为我们的view绑定id的。今天就通过重新手写ButterKinife来搞明白我们今天的主角-Anotation Processing(注解处理器)。

## 运行时注解

在写注解处理器之前，先用运行时注解来操作下。这里我们先新建一个library取名`lib-reflection`

然后自定义注解，我们只实现了View与id的绑定功能，所以我们这里定义：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface BindView {
    int value();
}
```

Target的Type说明这是一个用于修饰类和接口的注解，它也就是成员变量，即需要绑定资源id的view成员。

同时这个注解Retention是RUNTIME，表示注解不仅被保存到class文件中，jvm加载class文件之后，仍然存在。

注解定义好后就可以直接在项目中使用了;

```java
public class MainActivity extends AppCompatActivity {
 
    @BindView(R.id.textView)
    TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      	Binding.bind(this);
       textView.setText("哈哈哈哈");
    }
}
```

注意这里我们加了个`Binding.bind(this)`，就是它来帮助我们做注解的解析，之后在内部调用`MainActivity.textView=(TextView)MainActivity.findViewById()`来实现县为view绑定id的。还是在刚才的目录`lib-reflect`下创建了`Binding`类，具体代码如下：

```java
public class Binding {
    public static void bind(Activity activity){
        //反射获取注解注释
        for (Field field: activity.getClass().getDeclaredFields()){
            BindView bindView = field.getAnnotation(BindView.class);
            if (bindView != null){
                try {
                    //扩大范围
                    field.setAccessible(true);
                    field.set(activity, activity.findViewById(bindView.value()));
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

再运行就可以绑定了view的id了，不再需要我们手动的绑定id了。但是依靠反射始终是消耗性能的，这时候就用到我们的注解处理器了。

## 编译时注解

这里，我们新创建个`Java Library`的项目，就叫`lib-processor`吧。然后再这个目录下创建`BindingProcessor`，继承自`AbstractProcessor`。这个就是用于解析自定义注解的解析器了。不过要想让它生效还必须在`resource`下新建如下目录结构：

![image-20211120221109140](../../../../Library/Application Support/typora-user-images/image-20211120221109140.png)

`javax.annotation.processing.Processor`的文本文件里面内容就一行：

```java
com.pince.lib_processor.BindingProcessor
```

首先将之前定义的`BindView`注解改为编译时注解，放进新创建的`lin-annotations`目录下：

```java
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.FIELD)
public @interface BindView {
  int value();
}
```

还需要修改`build.gradle`文件，加入：

app依赖：

```java
implementation project(':lib')
annotationProcessor project(':lib-processor')
```

lib-processor依赖：

```java
implementation project(':lib-annotations')
```

lib依赖：

```java
//主项目需要的依赖
api project(':lib-annotations')
```

这么做就是为了让编译器使用我们的解析器用于解析注解。

后面的的工作只要都是在`BindingProcessor`操作了。通过读取类中的自定义注解，生成相应的绑定视图的代码，还需要引入一个库。

```java
compile 'com.squareup:javapoet:1.9.0'
```

先展示下最后自动生成的类：

```java
public class MainActivityBinding {
  public MainActivityBinding(MainActivity activity) {
    activity.textView = activity.findViewById(2131231093);
  }
}
```

上面的内容就是由`javapoet`生成的，下面就按照上面这个最终效果来一步一步分析要怎么生成我们的代理类。

看下面我们需要创建一个构造函数<? extend Activity>作为参数传入：

```java
String packageStr = element.getEnclosingElement().toString();
MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(ClassName.get(packageStr, classStr), "activity");
```

然后查找所有标注了BiidView注解的成员变量：

```java
 for (Element enclosedElement: element.getEnclosedElements()){
                if (enclosedElement.getKind() == ElementKind.FIELD){
                    //寻找BindView注释
                    BindView bindView = enclosedElement.getAnnotation(BindView.class);
                    if (bindView != null){
                        ....
                    }
                }
            }
```

然后再是生成findViewById的具体语句代码：

```java
for (Element enclosedElement: element.getEnclosedElements()){
                if (enclosedElement.getKind() == ElementKind.FIELD){
                    //寻找BindView注释
                    BindView bindView = enclosedElement.getAnnotation(BindView.class);
                    if (bindView != null){
                        hasBinding = true;
                        constructorBuilder.addStatement("activity.$N = activity.findViewById($L)",
                                enclosedElement.getSimpleName(), bindView.value());
                    }
                }
            }
```

做好了上面步骤，主要的代码也写完了，最后就是生成这个`MainActivityBinding `这个类：

```java
String packageStr = element.getEnclosingElement().toString();
ClassName className = ClassName.get(packageStr, classStr + "Binding");
TypeSpec builtClass = TypeSpec.classBuilder(className)
                    .addModifiers(Modifier.PUBLIC)
                    .addMethod(constructorBuilder.build())
                    .build();
try {
                    JavaFile.builder(packageStr, builtClass)
                            .build().writeTo(filer);
                } catch (IOException e) {
                    e.printStackTrace();
                }
```

注意这里的包名，生成的类的包名尽量与需要绑定的Activity所在的包名一致，这样BindView修饰的成员变量只需是包内可见就行。到了这里我们再执行`./gradlew :app:compileDebugJava `编译就可以自动生成我们所需要的类了。
到了这一步还没完，我们还需要在`lib moodule`目录下创建新的`Binding `帮助类：

```java
public class Binding {
    public static void bind(Activity activity){
        try {
            Class bindingClass = Class.forName(activity.getClass().getCanonicalName() + "Binding");
            Constructor constructor = bindingClass.getDeclaredConstructor(activity.getClass());
            constructor.newInstance(activity);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```

在这里利用了一点反射new出了MainActivityBinding实体，传入相应的`activity`在内部进行绑定操作。到这里简单的`ButterKnife`版本就实现了。`下面`BindingProcessor`给出完整代码：

```java
public class BindingProcessor extends AbstractProcessor {

    Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
      //使用Filer你可以创建文件
        filer = processingEnv.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
				//获取到全部的类
      	//Element代表的是源代码
   			// Element可以是类、方法、变量等
        for (Element element: roundEnv.getRootElements()){
            String packageStr = element.getEnclosingElement().toString();
            String classStr = element.getSimpleName().toString();
            
          //构建新的类的名字：原类名 + Binding
            ClassName className = ClassName.get(packageStr, classStr + "Binding");
          ////构建新的类的构造方法
            MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(ClassName.get(packageStr, classStr), "activity");
            boolean hasBinding = false;
  				//还有个getEnclosingElement  单数 被包住的外层
            //子类里的元素  字段 方法 内部类
            for (Element enclosedElement: element.getEnclosedElements()){
              //仅获取成员变量
                if (enclosedElement.getKind() == ElementKind.FIELD){
                    //寻找BindView注解
                    BindView bindView = enclosedElement.getAnnotation(BindView.class);
                    if (bindView != null){
                        hasBinding = true;
                      //在构造方法中加入代码
                        constructorBuilder.addStatement("activity.$N = activity.findViewById($L)",
                                enclosedElement.getSimpleName(), bindView.value());
                    }
                }
            }

            TypeSpec builtClass = TypeSpec.classBuilder(className)
                    .addModifiers(Modifier.PUBLIC)
                    .addMethod(constructorBuilder.build())
                    .build();

            if (hasBinding){
                try {
                  //生成 Java 文件
                    JavaFile.builder(packageStr, builtClass)
                            .build().writeTo(filer);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

        return false;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        //对这个注解进行注解处理
        return Collections.singleton(BindView.class.getCanonicalName());
    }
}
```

> **源码地址**：[APTDemo](https://github.com/LebronXia/APTDemo)

## 注解后话

元注解是用来定义其他注解的注解(在自定义注解的时候，需要使用到元注解来定义我们的注解)。java.lang.annotation提供了四种元注解：`@Retention`、 `@Target`、`@Inherited`、`@Documented`。

|   元注解   |                       说明                        |
| :--------: | :-----------------------------------------------: |
|  @Target   | 表明我们注解可以出现的地方。是一个ElementType枚举 |
| @Retention |               这个注解的的存活时间                |
| @Document  |       表明注解可以被javadoc此类的工具文档化       |
| @Inherited |        是否允许子类继承该注解，默认为false        |

### @Target

|   @Target-ElementType类型   |         说明         |
| :-------------------------: | :------------------: |
|      ElementType.TYPE       | 接口、类、枚举、注解 |
|      ElementType.FIELD      |   字段、枚举的常量   |
|     ElementType.METHOD      |         方法         |
|    ElementType.PARAMETER    |       方法参数       |
|   ElementType.CONSTRUCTOR   |       构造函数       |
| ElementType.LOCAL_VARIABLE  |       局部变量       |
| ElementType.ANNOTATION_TYPE |         注解         |
|     ElementType.PACKAGE     |          包          |

### @Retention

表示需要在什么几倍保存该注释信息，用于描述注解的生命周期。

| @Retention-RetentionPolicy类型 |                             说明                             |
| :----------------------------: | :----------------------------------------------------------: |
|     RetentionPolicy.SOURCE     | 注解只保留在源文件，当Java文件编译成class文件的时候，注解被遗弃 |
|     RetentionPolicy.CLASS      | 注解被保留到class文件，但jvm加载class文件时候被遗弃，这是默认的生命周期 |
|    RetentionPolicy.RUNTIME     |  解不仅被保存到class文件中，jvm加载class文件之后，仍然存在   |

### @Document

@Document表明我们标记的注解可以被javadoc此类的工具文档化

### @Inherited

@Inherited表明我们标记的注解是被继承的。比如，如果一个父类使用了@Inherited修饰的注解，则允许子类继承该父类的注解

### 注解解析

Class类里面常用方法如下：

```java
/**
     * 包名加类名
     */
    public String getName();

    /**
     * 类名
     */
    public String getSimpleName();

    /**
     * 返回当前类和父类层次的public构造方法
     */
    public Constructor<?>[] getConstructors();

    /**
     * 返回当前类所有的构造方法(public、private和protected)
     * 不包括父类
     */
    public Constructor<?>[] getDeclaredConstructors();

    /**
     * 返回当前类所有public的字段，包括父类
     */
    public Field[] getFields();

    /**
     * 返回当前类所有申明的字段，即包括public、private和protected，
     * 不包括父类
     */
    public native Field[] getDeclaredFields();

    /**
     * 返回当前类所有public的方法，包括父类
     */
    public Method[] getMethods();

    /**
     * 返回当前类所有的方法，即包括public、private和protected，
     * 不包括父类
     */
    public Method[] getDeclaredMethods();

    /**
     * 获取局部或匿名内部类在定义时所在的方法
     */
    public Method getEnclosingMethod();

    /**
     * 获取当前类的包
     */
    public Package getPackage();

    /**
     * 获取当前类的包名
     */
    public String getPackageName$();

    /**
     * 获取当前类的直接超类的 Type
     */
    public Type getGenericSuperclass();

    /**
     * 返回当前类直接实现的接口.不包含泛型参数信息
     */
    public Class<?>[] getInterfaces();

    /**
     * 返回当前类的修饰符，public,private,protected
     */
    public int getModifiers();
```

Field和Method都实现了AnnotatedElement接口，常用方法如下

```java
/**
     * 指定类型的注释是否存在于此元素上
     */
    default boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {
        return getAnnotation(annotationClass) != null;
    }

    /**
     * 返回该元素上存在的指定类型的注解
     */
    <T extends Annotation> T getAnnotation(Class<T> annotationClass);

    /**
     * 返回该元素上存在的所有注解
     */
    Annotation[] getAnnotations();

    /**
     * 返回该元素指定类型的注解
     */
    default <T extends Annotation> T[] getAnnotationsByType(Class<T> annotationClass) {
        return AnnotatedElements.getDirectOrIndirectAnnotationsByType(this, annotationClass);
    }

    /**
     * 返回直接存在与该元素上的所有注释(父类里面的不算)
     */
    default <T extends Annotation> T getDeclaredAnnotation(Class<T> annotationClass) {
        Objects.requireNonNull(annotationClass);
        // Loop over all directly-present annotations looking for a matching one
        for (Annotation annotation : getDeclaredAnnotations()) {
            if (annotationClass.equals(annotation.annotationType())) {
                // More robust to do a dynamic cast at runtime instead
                // of compile-time only.
                return annotationClass.cast(annotation);
            }
        }
        return null;
    }

    /**
     * 返回直接存在该元素岸上某类型的注释
     */
    default <T extends Annotation> T[] getDeclaredAnnotationsByType(Class<T> annotationClass) {
        return AnnotatedElements.getDirectOrIndirectAnnotationsByType(this, annotationClass);
    }

    /**
     * 返回直接存在与该元素上的所有注释
     */
    Annotation[] getDeclaredAnnotations();
```

## 注解处理器后话

注解处理器（Annotation Processor）是javac的一个工具，它用来在编译时扫描和处理注解（Annotation）。你可以自定义注解，并注册相应的注解处理器（自定义的注解处理器需继承自AbstractProcessor）。

定义一个注解处理器，需要继承自AbstractProcessor。如下所示：

```java
public class MyProcessor extends AbstractProcessor {

  /**
     * 每个Annotation Processor必须有一个空的构造函数。
     * 编译期间，init()会自动被注解处理工具调用，并传入ProcessingEnvironment参数，
     * 通过该参数可以获取到很多有用的工具类（Element，Filer，Messager等）
     */
    @Override
    public synchronized void init(ProcessingEnvironment env){ }

  /**
     * 用于指定自定义注解处理器(Annotation Processor)是注册给哪些注解的(Annotation),
     * 注解(Annotation)指定必须是完整的包名+类名
     */
    @Override
    public boolean process(Set<? extends TypeElement> annoations, RoundEnvironment env) { }

  /**
     * 用于指定你的java版本，一般返回：SourceVersion.latestSupported()
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() { }

  /**
     * Annotation Processor扫描出的结果会存储进roundEnvironment中，可以在这里获取到注解内容，编写你的操作逻辑。
     * 注意:process()函数中不能直接进行异常抛出,否则程序会异常崩溃
     */
    @Override
    public SourceVersion getSupportedSourceVersion() { }

}
```

注处理器的核心是process()方法，而process方法的核心是Element元素。Element里的元素可以是类、方法或是变量。在操作注解的过程中，编译器会扫描所有的Java文件，并将每一个部分看做是特定类型的的Element。可以代表包、类、接口、方法、字段等多种元素种类。

|     Element子类      |                            解释                            |
| :------------------: | :--------------------------------------------------------: |
|     TypeElement      |                        类或接口元素                        |
|   VariableElement    | 字段、enum常量、方法或构造方法参数、局部变量或异常参数元素 |
|  ExecutableElement   |         类或接口的方法、构造方法，或者注解类型元素         |
|    PackageElement    |                           包元素                           |
| TypeParameterElement |           类、接口、方法或构造方法元素的泛型参数           |

Element类常用方法如下：

```java
    /**
     * 返回此元素定义的类型，int,long这些
     */
    TypeMirror asType();

    /**
     * 返回此元素的种类:包、类、接口、方法、字段
     */
    ElementKind getKind();

    /**
     * 返回此元素的修饰符:public、private、protected
     */
    Set<Modifier> getModifiers();

    /**
     * 返回此元素的简单名称(类名)
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

    /**
     * 返回直接存在于此元素上的注解
     * 要获得继承的注解，可使用 getAllAnnotationMirrors
     */
    List<? extends AnnotationMirror> getAnnotationMirrors();

    /**
     * 返回此元素上存在的指定类型的注解
     */
    <A extends Annotation> A getAnnotation(Class<A> var1);
```

还有四个帮助类也是需要我们了解的:

- Elements：一个用来处理Element的工具类
- Types：一个用来处理TypeMirror的工具类
- Filer：用于创建文件(比如创建class文件)
- Messager：用于输出，类似printf函数

**参考**

[自定义注解和解析器实现ButterKnife](https://mp.weixin.qq.com/s?__biz=MzAxMTg2MjA2OA==&mid=2649842013&idx=1&sn=0267e1974f293501b504d71c18a0430d&chksm=83bf6806b4c8e110ff4443975548e91c5d4ea6fba4b28a9b21b516e97b1be9eed534b5eebd6c#rd)

[Android 自定义注解(Annotation)](https://blog.csdn.net/wuyuxing24/article/details/81139846)

[Java注解处理器](https://race604.com/annotation-processing/)

[JavaPoet源码初探](https://blog.csdn.net/baidu_33409651/article/details/51598467)
