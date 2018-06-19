> 写在开头:最近在翻读一些开源库的时候,发现大多使用了注解,于是不得不来仔细了解一下Android下的注解知识

### 什么是注解 ###

`java.lang.annotation`，接口 Annotation，在JDK5.0及以后版本引入。  

注解是代码里的特殊标记，这些标记可以在**编译**、**类加载**、**运行时被读取**，并执行相应的**处理**。通过使用Annotation，开发人员可以在不改变原有逻辑的情况下，在源文件中嵌入一些补充的信息。代码分析工具、开发工具和部署工具可以通过这些补充信息进行验证、处理或者进行部署。

Annotation不能运行，它只有成员变量，没有方法。Annotation跟public、final等修饰符的地位一样，都是程序元素的一部分，Annotation不能作为一个程序元素使用。

### 注解的作用 ###

注解将一些本来重复性的工作，变成程序自动完成，简化和自动化该过程。比如用于生成Java doc，比如编译时进行格式检查，比如自动生成代码等，用于提升软件的质量和提高软件的生产效率。

### 常见的注解 ###

> Android已经定义好的注解大致分为4种,称之为4大元注解  

**@Retention：定义该Annotation被保留的时间长度**  

- `RetentionPoicy.SOURCE`:注解只保留在源文件，当Java文件编译成class文件的时候，注解被遗弃；用于做一些检查性的操作，比如 `@Override` 和 `@SuppressWarnings`
- `RetentionPoicy.CLASS:`注解被保留到class文件，但jvm加载class文件时候被遗弃，这是默认的生命周期；用于在编译时进行一些预处理操作，比如生成一些辅助代码（如 `ButterKnife`）
- `RetentionPoicy.RUNTIME:`注解不仅被保存到class文件中，jvm加载class文件之后，仍然存在；用于在运行时去动态获取注解信息。这个注解大都会与反射一起使用

**@Target:定义了Annotation所修饰的对象范围**

- `ElementType.CONSTRUCTOR`:用于描述构造器
- `ElementType.FIELD`:用于描述域
- `ElementType.LOCAL_VARIABLE`:用于描述局部变量
- `ElementType.METHOD`:用于描述方法
- `ElementType.PACKAGE`:用于描述包
- `ElementType.PARAMETER`:用于描述参数
- `ElementType.TYPE`:用于描述类、接口(包括注解类型) 或enum声明

未标注则表示可修饰所有

**@Inherited:是否允许子类继承父类的注解,默认是false**

**@Documented 是否会保存到 Javadoc 文档中**

### 自定义注解 ###

自定义注解中使用到较多的是**运行时注解**和**编译时注解**

> 运行时注解

下面通过一个简单的动态绑定控件的例子来说明

首先定义一个简单的自定义注解,

	@Target({ElementType.FIELD})
	@Retention(RetentionPolicy.RUNTIME)
	public @interface BindView {
	    int value() default  -1;
	}

然后在app运行时,通过反射将findViewbyId()得到的控件，注入到我们需要的变量中。

	public class AnnotationActivity extends AppCompatActivity {
	
	    @BindView(R.id.annotation_tv)
	    private TextView mTv;
	
	    @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_annotation);
	
	        getAllAnnotationView();
	
	        mTv.setText("Annotation");
	    }
	
	    private void getAllAnnotationView() {
	        //获得成员变量
	        Field[] fields = this.getClass().getDeclaredFields();
	
	        for (Field field : fields) {
	            try {
	                //判断注解
	                if (field.getAnnotations() != null) {
	                    //确定注解类型
	                    if (field.isAnnotationPresent(BindView.class)) {
	                        //允许修改反射属性
	                        field.setAccessible(true);
	                        BindView bindView = field.getAnnotation(BindView.class);
	                        //findViewById将注解的id，找到View注入成员变量中
	                        field.set(this, findViewById(bindView.value()));
	                    }
	                }
	            } catch (Exception e) {
	                e.printStackTrace();
	            }
	        }
	    }
	}

最后`mTv`上显示的就是我们想要的“Annotation”文字,这看起来是不是有点像`ButterKnife`,但是要注意反射是很消耗性能的,  

所以我们常用的控件绑定库`ButterKnife`并不是采用运行时注解,而是采用的编译时注解.


> 编译时注解

**定义**

在说编译时注解之前,我们得先提一提注解处理器`AbstractProcessor`。  
它是javac的一个工具,用来在编译时扫描和处理注解`Annotation`,你可以自定义注解,并注册到相应的注解处理器,由注解处理器来处理你的注解。

一个注解的注解处理器，以Java代码(或者编译过的字节码)作为输入,生成文件(通畅是.java文件)作为输出。这些由注解器生成的.java代码和普通的.java一样,可以被javac编译。

**导入**

因为`AbstractProcessor`是javac中的一个工具,所以在Android的工程下没法直接调用。下面提供一个本人尝试可行的导入方式。

> File-->New Module-->java library 新建一个java module,注意一定要是java library,不是Android library

接下来就可以在对应的library中使用`AbstractProcessor`了  

**准备工作完成之后,下面通过一个简单的注解绑定控件的例子来讲述**

> 工程目录
	
	--app                 (主工程)
	--app_annotation      (java module 自定义注解)
	--annotation-api      (Android module)
	--app_compiler        (java module 注解处理器逻辑)

在annotation module下创建注解

	@Retention(RetentionPolicy.CLASS)
	public @interface BindView {
		//绑定控件
    	int value();
	}

在compiler module下创建注解处理器 `CustomProcessor`

	public class CustomProcessor extends AbstractProcessor {
	    //文件相关的辅助类
	    private Filer mFiler;
	    //元素相关的辅助类
	    private Elements mElements;
		
		//初始化参数
	    @Override
	    public synchronized void init(ProcessingEnvironment processingEnvironment) {
	        super.init(processingEnvironment);
	        mElements = processingEnvironment.getElementUtils();
	        mFiler = processingEnvironment.getFiler();
	    }
		
		//核心处理逻辑,相当于java中的主函数main(),你需要在这里编写你自己定义的注解的处理逻辑
		//返回值 true时表示当前处理,不允许后续的注解器处理
	    @Override
	    public boolean process(Set<? extends TypeElement> set, RoundEnvironment env) {
	        return true;
	    }
	
	    //自定义注解集合
	    @Override
	    public Set<String> getSupportedAnnotationTypes() {
	        Set<String> types = new LinkedHashSet<>();
	        types.add(BindView.class.getCanonicalName());
	        return types;
	    }
	
	    @Override
	    public SourceVersion getSupportedSourceVersion() {
	        return SourceVersion.latestSupported();
	    }
	}

其中核心代码`process`函数有两个参数,我们重点关注第二个参数,因为`env`表示的是所有注解的集合

首先我们先简单的说明一下`porcess`的处理流程

1. 遍历env,得到我们需要的元素列表
2. 将元素列表封装成对象,方便之后的处理(如同平时解析json数据一样)
3. 通过JavaPoet库将对象以我们期望的形式生成java文件

> 1. 遍历env,得到我们需要的元素列表

	for(Element element : roundEnvironment.getElementsAnnotatedWith(BindView.class)){
		// todo ....

        // 判断元素的类型为Class
        if (element.getKind() == ElementKind.CLASS) {
            // 显示转换元素类型
            TypeElement typeElement = (TypeElement) element;
            // 输出元素名称
            System.out.println(typeElement.getSimpleName());
            // 输出注解属性值
            System.out.println(typeElement.getAnnotation(BindView.class).value());
        }
    }

直接通过`getElementsAnnotatedWith`函数就能获取到需要的注解的列表,函数体内加了些`element`简单的使用

> 2.将元素列表封装成对象,方便之后的处理

首先,我们需要明确,在绑定控件的这个事件下，我们需要的是控件的id。

新建类 `BindViewField.class` 用来保存自定义注解`BindView`相关的属性

**BindViewField.class**

	public class BindViewField {

	    private VariableElement mFieldElement;
	
	    private int mResId;
	
	    public BindViewField(Element element) throws IllegalArgumentException {
	        if (element.getKind() != ElementKind.FIELD) {
	            throw new IllegalArgumentException(String.format("Only field can be annotated with @%s",
	                    BindView.class.getSimpleName()));
	        }
	        mFieldElement = (VariableElement) element;
	        BindView bindView = mFieldElement.getAnnotation(BindView.class);
	        mResId = bindView.value();
	        if (mResId < 0) {
	            throw new IllegalArgumentException(String.format("value() in %s for field % is not valid",
	                    BindView.class.getSimpleName(), mFieldElement.getSimpleName()));
	        }
	    }
	
	    public Name getFieldName() {
	        return mFieldElement.getSimpleName();
	    }
	
	    public int getResId() {
	        return mResId;
	    }
	
	    public TypeMirror getFieldType() {
	        return mFieldElement.asType();
	    }
	}

上述的`BindViewField`只能表示一个自定义注解`bindView`对象,而一个类中很可能会有多个自定义注解,所以还需要创建一个对象`Annotation.class`来管理自定义注解集合、

**AnnotatedClass.class**

	public class AnnotatedClass {

	    //类
	    public TypeElement mClassElement;
	
	    //类内的注解变量
	    public List<BindViewField> mFiled;
	
	    //元素帮助类
	    public Elements mElementUtils;
	
	    public AnnotatedClass(TypeElement classElement, Elements elementUtils) {
	        this.mClassElement = classElement;
	        this.mElementUtils = elementUtils;
	        this.mFiled = new ArrayList<>();
	    }
		
		//添加注解变量
	    public void addField(BindViewField field) {
	        mFiled.add(field);
	    }
		
		//获取包名
	    public String getPackageName(TypeElement type) {
	        return mElementUtils.getPackageOf(type).getQualifiedName().toString();
	    }
		
		//获取类名
	    private static String getClassName(TypeElement type, String packageName) {
	        int packageLen = packageName.length() + 1;
	        return type.getQualifiedName().toString().substring(packageLen).replace('.', '$');
	    }
	}

**给上完整的解析流程**

	//解析过后的目标注解集合
    private Map<String, AnnotatedClass> mAnnotatedClassMap = new HashMap<>();

	@Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        mAnnotatedClassMap.clear();
        try {
            processBindView(roundEnvironment);
        } catch (Exception e) {
            e.printStackTrace();
            return true;
        }
        return true;
    }

	private void processBindView(RoundEnvironment env) {
        for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
            AnnotatedClass annotatedClass = getAnnotatedClass(element);
            BindViewField field = new BindViewField(element);
            annotatedClass.addField(field);
            System.out.print("p_element=" + element.getSimpleName() + ",p_set=" + element.getModifiers());
        }
    }

	private AnnotatedClass getAnnotatedClass(Element element) {
        TypeElement encloseElement = (TypeElement) element.getEnclosingElement();
        String fullClassName = encloseElement.getQualifiedName().toString();
        AnnotatedClass annotatedClass = mAnnotatedClassMap.get(fullClassName);
        if (annotatedClass == null) {
            annotatedClass = new AnnotatedClass(encloseElement, mElements);
            mAnnotatedClassMap.put(fullClassName, annotatedClass);
        }
        return annotatedClass;
    }

> 3.通过JavaPoet库将对象以我们期望的形式生成java文件

通过上述两步成功获取了自定义注解的元素对象,但是还是缺少一步关键的步骤,缺少一步`findViewById()`,实际上ButterKnife这个很出名的库也并没有省略`findViewById()`这一个步骤,只是在编译的时候，在build/generated/source/apt/debug下生成了一个文件,帮忙执行了`findViewById()`这一行为而已。

同样的,我们这里也需要生成一个java文件,采用的是JavaPoet这个库。具体的使用 [参考链接](https://www.jianshu.com/p/95f12f72f69a)

在`process`函数中增加生成java文件的逻辑

	@Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        mAnnotatedClassMap.clear();
        try {
            processBindView(roundEnvironment);
        } catch (Exception e) {
            e.printStackTrace();
            return true;
        }

        try {
            for (AnnotatedClass annotatedClass : mAnnotatedClassMap.values()) {
                annotatedClass.generateFinder().writeTo(mFiler);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return true;
    }

其中核心逻辑`annotatedClass.generateFinder().writeTo(mFiler);`  
具体实现在`AnnotatedClass`中


	public JavaFile generateFinder() {

        //构建 inject 方法
        MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder("inject")
                .addModifiers(Modifier.PUBLIC)
                .addAnnotation(Override.class)
                .addParameter(TypeName.get(mClassElement.asType()), "host", Modifier.FINAL)
                .addParameter(TypeName.OBJECT, "source")
                .addParameter(Utils.FINDER, "finder");

        //inject函数内的核心逻辑,
        // host.btn1=(Button)finder.findView(source,2131427450);  ----生成代码
        // host.$N=($T)finder.findView(source,$L)                 ----原始代码
        // 对比就会发现这里执行了实际的findViewById绑定事件
        for (BindViewField field : mFiled) {
            methodBuilder.addStatement("host.$N=($T)finder.findView(source,$L)", field.getFieldName()
                    , ClassName.get(field.getFieldType()), field.getResId());
        }

        String packageName = getPackageName(mClassElement);
        String className = getClassName(mClassElement, packageName);
        ClassName bindClassName = ClassName.get(packageName, className);

        //构建类对象
        TypeSpec finderClass = TypeSpec.classBuilder(bindClassName.simpleName() + "$$Injector")
                .addModifiers(Modifier.PUBLIC)
                .addSuperinterface(ParameterizedTypeName.get(Utils.INJECTOR, TypeName.get(mClassElement.asType())))   //继承接口
                .addMethod(methodBuilder.build())
                .build();

        return JavaFile.builder(packageName, finderClass).build();
    }

到这里,大部分逻辑都已实现,用来绑定控件的辅助类也已通关JavaPoet生成了，只差最后一步,**宿主注册**,如同ButterKnife一般，`ButterKnife.bind(this)`

> 编写调用接口

在annotation-api下新建

注入接口`Injector`

	public interface Injector<T> {
	
	    void inject(T host, Object source, Finder finder);
	}

宿主通用接口`Finder`(方便之后扩展到view和fragment)

	public interface Finder {

	    Context getContext(Object source);
	
	    View findView(Object source, int id);
	}

activity实现类 `ActivityFinder`

	public class ActivityFinder implements Finder{
	
	    @Override
	    public Context getContext(Object source) {
	        return (Activity) source;
	    }
	
	    @Override
	    public View findView(Object source, int id) {
	        return ((Activity) (source)).findViewById(id);
	    }
	}

核心实现类 `ButterKnife`

	public class ButterKnife {
	
	    private static final ActivityFinder finder = new ActivityFinder();
	    private static Map<String, Injector> FINDER_MAP = new HashMap<>();
	
	    public static void bind(Activity activity) {
	        bind(activity, activity);
	    }
	
	    private static void bind(Object host, Object source) {
	        bind(host, source, finder);
	    }
	
	    private static void bind(Object host, Object source, Finder finder) {
	        String className = host.getClass().getName();
	        try {
	            Injector injector = FINDER_MAP.get(className);
	            if (injector == null) {
	                Class<?> finderClass = Class.forName(className + "$$Injector");
	                injector = (Injector) finderClass.newInstance();
	                FINDER_MAP.put(className, injector);
	            }
	            injector.inject(host, source, finder);
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	    }
	}


### 主工程下调用 ###

对应的按钮可以直接使用,不需要`findViewById()`

	public class MainActivity extends AppCompatActivity {
	
		@BindView(R.id.annotation_tv)
		public TextView tv1;
		
		@Override
		protected void onCreate(Bundle savedInstanceState) {
		    super.onCreate(savedInstanceState);
		    setContentView(R.layout.activity_main);
		    ButterKnife.bind(this);
		    tv1.setText("annotation_demo");
		}
	}



### JavaPoet的简单介绍 ###

常用的几个类

- MethodSpec 代表一个构造函数或方法声明。
- TypeSpec 代表一个类，接口，或者枚举声明。
- FieldSpec 代表一个成员变量，一个字段声明。
- JavaFile包含一个顶级类的Java文件。

常用的占位符

$L for variable (变量)
$S for Strings
$T for Types
$N for Names(我们自己生成的方法名或者变量名等等)

### 源码地址 ###

[demo地址](https://github.com/niknowzcd/annotation)

### 参考 ###

[探究Android中的注解](https://droidyue.com/blog/2016/08/14/android-annnotation/)  
[注解快速入门](https://www.jianshu.com/p/9ca78aa4ab4d)  

[自定义注解之编译时注解](https://blog.csdn.net/github_35180164/article/details/52121038)

[一小时搞明白注解处理器](https://blog.csdn.net/u013045971/article/details/53509237)

[javapoet——让你从重复无聊的代码中解放出来](https://www.jianshu.com/p/95f12f72f69a)

[JavaPoet源码无法正常导入Modifier类的讨论](https://github.com/square/javapoet/issues/491)