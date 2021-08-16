# Kala Lang Design

本仓库用于讨论 kala 语言（TODO）的设计与功能。

目前本仓库主要用途是记录对该语言的设想<del>（卫星），具体啥时候做出来还得看我要摸到啥时候</del>。讨论请前往本仓库的 [Issue 页](https://github.com/Glavo/kala-lang/issues)。

## 概述

Kala 语言目标是在完全兼容 Java 语言的基础上，提供更强大易用的语言功能，并且无需引入额外的运行时依赖。

Kala 语言应该能够完全兼容最新版本 Java 的语法，在此基础上扩展，并允许将编译目标设定为更低版本的 JVM 平台（譬如 Java 8），同时为主目标为低版本平台的程序适配高版本特性（譬如 Primitive Class）提供帮助。

Kala 编译器工具链应该实现 Java 注解处理器 API，兼容用户已有的注解处理器。

Kala 应该内置对 Lombok 大部分功能的支持。

应该允许纯 Java 类库通过使用特殊注解提供对 Kala 友好的 API。

## 特性

一些特性的草案<del>（卫星）</del>。待补充，待完善。欢迎通过 [Issue](https://github.com/Glavo/kala-lang/issues) 或者 [Discussions](https://github.com/Glavo/kala-lang/discussions) 讨论。

* 取消跨类常量折叠。

* 扩展交集类型/并集类型：允许交集类型（`A & B`）和并集类型（`A | B`）出现在所有其他类型可以出现的地方（变量声明，函数参数，函数返回值类型，等等）。

* 将 `void` 视为类型（待定）：允许 `void` 作为类型使用。

* `this` 类型

  ```java
  class BaseClass {
      Class<this-type> myGetClass() -> getClass();
      this-type append(Object obj) {...} 
      /*
      Equivalent to:
      @ReturnThis
      void append(Object obj){...}
      
      Or:
      @ReturnThis
      BaseClass append(Object obj){...}
       */
  }
  
  class MyClass extends BaseClass {
      this-type myMethod() {...}
  }
  
  MyClass myClass = new MyClass();
  BaseClass baseClass = myClass;
  
  myClass.myGetClass();   // type: Class<MyClass>
  baseClass.myGetClass(); // type: Class<BaseClass>
  
  myClass
      .append(...)
      .append(...)
      .myMethod(); //ok
  ```

* Bottom 类型

  ```java
  bottom-type exit() {
      System.exit(0);
  }
  
  /*
  Equivalent to:
  @NoReturn
  void exit() {
      System.exit(0);
  }
   */
  
  int fun() {
      exit();
  } // ok
  ```
  
* 属性语法

  * 允许使用属性访问语法（`o.a`）替代 Getter/Setter 方法（`o.getA()`/`o.setA(...)`）调用。

  * 简化声明属性的语法。

    ```java
    class Test {
        public property String name;
        
        /* 
        Or:
        public property String name {
            private field;
            public get() -> field;
            public set(value) -> field = value;
        }
        
        Equivalent to:
        private String name;
        public String getName() -> name;
        public void setName(String value) -> name = value;
        */
    
        public readonly property String value {
            private field;
            private boolean isInitialized = false;
        
            private String initValue() {
                // ...
                return someString;
            }
        
            public get() {
                if(!isInitialized) {
                    field = initValue();
                    isInitialized = true;
                }
                return field;
            }
        }
        
        public boolean valueIsInitialized() -> this::value.isInitialized;
        
        public property String title {
            private StringProperty field;
            
            public get() -> field.get();
            public set(value) -> field.set(value);
        }
        
        public StringProperty titleProperty() -> this::title.field;
    }
    ```
  * 抽象属性
    
    ```java
    interface I {
        readonly property String name;
        
        StringProperty valueProperty();
        property String value {
            default get() -> valueProperty().get();
            default set(value) -> valueProperty().set(value);
        }
    }
    
    abstract class C {
        abstract readonly property String name;
        
        abstract property String value {
            abstract get();
            final set(value) -> throw new UnsupportedOperationException();
        }
    }
    ```
  
* 单行方法

  ```java
  public String toString() -> $"[${this.name}]";
  
  public static String[] newStringArray(int size) = String[]::new;
  
  public static Pair<String, int> pairOf(String, int) = Pair::new;
  ```

* 扩展方法

  ```java
  public static void printMe(String this) {
      System.out.println(this);
  }
  
  "Hello world!".printMe(); // print "Hello world!"
  ```

* 成员方法自类型限定

  ```java 
  interface Seq<covariant E> {
      default int sum(Seq<int> this) {
          int res = 0;
          for(var elem : this) {
              res += elem;
          }
          return res;
      }
  }
  
  Seq.of(0, 1, 2, 3).sum(); // 6
  // compile time error: Seq.of("str0", "str1", "str2", "str3").sum();
  ```

* 嵌套方法

  ```java
  void outer(String arg) {
      void inner() {
          System.out.println(arg);
      }
      
      static int helper(int a, int b) {
          // compile time error: System.out.println(arg);
          return a * 2 + b;
      }
      
      inner();
      Seq.of(1, 2, 3).reduce(::helper); // 11
  }
  ```

* 局部 `import`

  ```java
  void f() {
      import static kala.Tuple.*;
      import static java.lang.System.out;
      
      var tuple = of("A", "B");
      out.println(tuple); // print "(A, B)"
  }
  ```

* `TargetName`

  ```java
  interface Seq<covariant E> {
      @TargetName("sumOfInt")
      default int sum(Seq<int> this) {...}
      
      @TargetName("sumOfLong")
      default long sum(Seq<long> this) {...}
      
      @TargetName("sumOfFloat")
      default float sum(Seq<float> this) {...}
      
      @TargetName("sumOfDouble")
      default double sum(Seq<double> this) {...}
  }
  ```

* 宏（待定?）

  ```java
  inline void fun(int arg) = macro funImpl;
  Expression<void> funImpl(Expression<int> argExpr) {...}
  ```

* 语法树

  ```java
  Expression<int> expr = '(1 + 2);
  ```

* 增强 Record

  * 允许继承自其他父类

    * 以此可以实现在低于 Java 14 的平台上使用 Record。

      ```java
      public record Pair<A, B>(A first, B second) extends Object {}
      ```

* 使大多数语句成为表达式

  * Block 表达式（待定，与初始化器类的语法冲突，二选一）

    ```java
    Object[] arr = {
        Object[] tmp = new Object[size];
        for (int i = 0; i < size; i++) {
            res[i] = it.next();
        }
        tmp;
    };
    ```
    
  * `if` 表达式

    ```java
    var str = if(c1) {
        fun1();
        "str1";
    } else if(c2) {
        fun2();
        "str2";
    } else {
        fun3();
        "str3";
    }
    ```
    
  * `return` 表达式
    
    ```java
    var obj = a != null ? a : return null;
    ```
    
  * `throw` 表达式
    
    ```java
    var obj = a != null ? a : throw new Exception();
    ```
  
* 字符串插值

  ```java
  public String toString() -> $"MyClass{name=$name, value=$value}";
  ```
  
* 弱化受检异常检测

  ```java
  void fun() {
      try {
          var a = 1 + 2;
      } catch(IOException ex) {
          // ok
      }
      
      throw new IOException(); // ok
  }
  ```

* 运算符重载（待定）

* 空安全

  * 可空类型（`T?`）/不可空类型（`T!`）/平台类型（`T`）

    * 识别 Java 中的可空性注解（见 [Kotlin 编译器源码](https://github.com/JetBrains/kotlin/blob/master/core/compiler.common.jvm/src/org/jetbrains/kotlin/load/java/JvmAnnotationNames.kt)）。
    
  * 空处理增强
    
    ```java
    Seq<String>? seq = ...;
    System.out.println(seq?.toString() ?? "seq is null");
    seq ??= Seq.empty();
    ```

* 泛型增强

  * 声明处形变

    ```java
    interface Seq<covariant E> {
        static <T> Seq<T> from(Seq<T> seq) {...} // Equivalent to: static <T> Seq<T> from(Seq<? extends T> seq) {...}
    }
    ```

  * 泛型约束

    ```java
    <T> T f(T t) where T extends CharSequence & Comparable<? super T> -> t;
    // Equivalent to: <T extends CharSequence & Comparable<? super T>> T f(T t) { return t; }
    
    <T> T f(T t) 
        where T extends CharSequence & Comparable<? super T>, 
              T super StringBuilder,
              T : Reified {...}
    /*
     Equivalent to:
     <T extends CharSequence & Comparable<? super T> super StringBuilder : Reified>
     T f(T t) {...}
     */
    ```
  
  * 通用泛型
  
    ```java
    Seq<int> list = Seq.of(1, 2, 3);
    ```
    
  * ???约束（待定？）
  
    ```java
    trait HasFactory<T> {
        abstract static T create();
        
        abstract void foo();
    }
    
    class C {
        static C create() -> new C();
        
        void foo() {
            System.out.println("this is C");
        }
    }
    
    <T : HasFactory> T createAndDoFoo() { // Equivalent to: <T> T createAndDoFoo() where T : HasFactory
        var res = T.create(); 
        res.foo();
        return res;
    }
    
    C c = createAndDoFoo(); // print "this is C"
    ```
  
  * 泛型具化
  
    ```java
    class TypeMirror {
        static <T> Type<T> getType() where T : Reified;
    }
    
    Type<Seq<String | int>> tpe = TypeMirror.getType();
    System.out.println(tpe); // print "kala.collection.Seq<java.lang.String | int>"
    
    <T : Reified> T[] newArray(int size) -> new T[size];
    
    String[] arr = newArray(10);
    ```

## 兼容性

Kala 语言应该能将高版本 Java 语言特性编译至低版本平台。对于纯语法层面的特性（模式匹配，`var` 等），这是一件很轻松的事情，但是对于需要运行时协同的功能（`record` 等），需要寻找其他实现方式实现兼容。

有一个值得考虑的选择（后文将该选择简称为“多目标翻译”）：支持类似 `kalac --release 8 -d out/ --release 16 -d out-java16/ ...`  的方式指定多个输出路径，为不同 target 目标生成不同的类文件，最后装入一个 Multi-Release JAR 文件中。

若确定将要支持多目标翻译，可以考虑在此基础上实现多目标宏，以此简化为不同 Java 版本提供不同实现的方式：

```java
class Utils {
#if JAVA_VERSION < 9
    private static Unsafe unsafe = with {
        try {
            Field field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            (Unsafe) field.get(null);
        } catch (Exception e) {
            null;
        }
    };
    
    public static void fullFence() {
        unsafe.fullFence();
    }
#else 
    public static void fullFence() {
        VarHandle.fullFence();
    }
#endif
    
}
```

### Record

Kala 可能会增强 `record` 支持指定其他超类，只有在超类显式或隐式被指定为 `java.lang.Record` 时才会被真正视为 Java 中的 `record` 进行翻译。

（多目标翻译：在未显式指定超类时，为高于 Java 16 的目标生成超类为 `java.lang.Record` 版本，为低于 Java 16 的目标生成超类为 `java.lang.Object` 的版本）

### Sealed Class

（待确认）应该可以为任意版本目标生成 `PermittedSubclasses` 属性，低版本平台会自动忽略该项，但不确认 Java 15/16 是否必须在运行时开启 `--enable-preview` 选项。

### Primitive Class

必须保证实现多目标翻译才能正确应对 Primitive Class。

对于 Primitive Class 在低版本平台上的翻译，应该为其生成工厂方法（`<new>` 方法），将对其 `new` 调用翻译为工厂方法调用而非构造器调用。

但是 `Q` 类型的兼容方案未知。

### 通用泛型

使用 Universal Generics 的 [JEP 草案](https://openjdk.java.net/jeps/8261529)中所描述的方式翻译。

