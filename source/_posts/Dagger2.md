---
title: Dagger2
---

# Dagger2
## 1. Dagger2简介
> Dagger2是Dagger的升级版，是一个**==依赖注入==**框架，现在由Google接手维护。这里所说的“依赖注入”，就是目标类对所依赖类的初始化的一个过程，只是这个过程现在不用程序员手动输入，而是由Dagger2框架为我们完成。
## 2. Dagger2优点
- ### 增加开发效率
> 我们之前创建对象的方式通过new关键字来完成，这是一个重复的，技术含量比较低的工作，浪费了我们的时间，Dagger2框架，让我们不用去管理这些问题。让我们更好的专注于业务。

> 另外，我们可以使用Dagger2提供的操作符，完成一些稍复杂的问题。例如，SingleTon（单例），我们不用担心自己写的单例是否线程安全，以及使用了什么模式。
- ### 类实例变得清晰
> 使用Component和Module，可以使得整个app的类实例结构变的很清晰。
- ### 解耦
> 当构造方法发生变化的时候，我们不用再目标类里面做修改，我们是需要修改对应的@Inject标注的构造方法或者Module中对应的@Provides标注的创建方法。
## 3. Dagger2用法
- ### 之前创建类实例的方式

---
    public class TestA{
        TestB b = new TestB();
    }
    
    public class TestB{
        public TestB(){
            //...
        }
    }
---
- ### Dagger2基本写法
#### 所依赖的类的构造方法无形参且不属于第三方类库
---
    public class App extends MultiDexApplication {
        private AppComponent mAppComponent;
        @Override
        public void onCreate() {
            //...
            mAppComponent = DaggerAppComponent.builder()
                    .appModule(new AppModule(this))
                    .build();
            //...
        }
    }
---
---
    public class TestA{
        @Inject
        TestB b;
        
        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            App.getInstance().AppComponent().inject(this);
        }
    }

    public class TestB{
        @Inject
        public TestB(){
            //...
        }
    }
---
他们通过Component把他们连接起来，Component连接目标类属性及其所依赖类。
#### Component是什么？
> 一个类如果是Component类，必须用Component注解来标注该类，并且该类是接口或抽象类。Component在目标类中所依赖的其他类与其他类的构造函数之间可以起到一个桥梁的作用。

> Component需要引用到目标类的实例，Component会查找目标类中用Inject注解标注的属性，查找到相应的属性后会接着查找该属性对应的用Inject标注的构造函数（这时候就发生联系了），剩下的工作就是初始化该属性的实例并把实例进行赋值


```
graph LR
Inject注解标注的构造方法-->Component桥梁
Component桥梁-->目标类中用Inject注解标注的属性
```
#### 流程

- 用Inject注解标注目标类中其他类，即属性
- 用Inject注解标注其他类的构造方法
- 若构造方法还依赖于其他的类，则重复进行上面2个步骤
- 调用Component的inject(Object o)方法
ComPonent会把目标类所依赖的实例注入到目标类中！

#### 如果所依赖的类是第三方的类库或者是所依赖的类的构造方法中有形参

> 项目中使用到了第三方的类库，第三方类库又不能修改，所以根本不可能把Inject注解加入这些类中

> 另外目标类所依赖的其他类的构造方法中，有形参的需要，也无法直接通过上面的方式获得

> 把上面这两种情况下的对象实例化，可以放在Module中

#### Module又是什么鬼？

> 把封装第三方类库以及构造有形参的代码放入Module中，有Module统一管理。Module其实是一个简单工厂模式，Module里面的方法基本都是创建类实例的方法。Module类需要使用Module进行标注

---
    @Module
    public class AppModule {

        @Provides
        @Singleton
        Gson provideGson() {
            return new GsonBuilder()
                    .setDateFormat("yyyy-MM-dd HH:mm:ss z")
                    .registerTypeAdapter(LocalDate.class, new LocalDateGsonAdapter())
                    .registerTypeAdapterFactory(PeriodGsonAdapterFactory.create())
                    .registerTypeAdapter(PushData.PushType.class, new PushTypeAdapter())
                    .create();
        }

        @Provides
        @Singleton
        RecordDatabaseManager provideRecordDatabaseManager(SQLiteOpenHelper SQLiteOpenHelper, UserHelper userHelper, PeriodApi periodApi, SharePreferenceUtils sharePreferenceUtils) {
            return new RecordDatabaseManagerImpl(SQLiteOpenHelper, userHelper, periodApi, sharePreferenceUtils);
        }
        
    }
---

> 目前提供实例的地方有了，但是如何让提供的实例注入到目标类中呢，有上面的分析我们知道Component是实现注入的，那么如何让Module和Component关联起来呢？


---

    @Singleton
    @Component(modules = AppModule.class)
    public interface AppComponent {
        void inject(NumManagerActivity numManagerActivity);
    }
---

> 其中Component的属性modules是关键，该属性是一个数组，可以包含多个Module。其中Component把Module提供的实例注入到目标类中。

#### Component是如何查找Module中提供的实例？

> Module中的创建类实例方法用Provides进行标注，Component在搜索到目标类中用Inject注解标注的属性后，Component就会去Module中去查找用Provides标注的对应的创建类实例方法

> 注： Component会首先查找Module中的实例，如果Module中没有，它才回去所依赖的类去查找被@Inject标注的构造方法，所以Module的优先级要比构造方法被@Inject标注的优先级高

#### 依赖注入迷失

> 若一个类的实例有多种方法可以创建出来，Component应该选择哪种方法来创建该类的实例呢？


```
sequenceDiagram
A->>Component: A中两构造方法：A()、A(...)
Component->>C: C中@Inject A a；
```

> Qualifier（限定符）可以解决依赖注入迷失问题的