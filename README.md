# Kala Lang Design

本仓库用于讨论 kala 语言（TODO）的设计与功能。

目前本仓库主要用途是记录对该语言的草案<del>（卫星），具体啥时候做出来还得看我要摸到啥时候</del>。讨论请前往本仓库的 [Issue 页](https://github.com/Glavo/kala-lang/issues)。

## 概述

Kala 语言目标是在完全兼容 Java 语言的基础上，提供更强大易用的语言功能，并且绝大部分功能无需引入额外的运行时依赖。

Kala 语言应该能够完全兼容最新版本 Java 的语法，在此基础上扩展，并允许将编译目标设定为更低版本的 JVM 平台（譬如 Java 8），同时为主目标为低版本平台的程序适配高版本特性（譬如 Primitive Class）提供帮助。

Kala 编译器工具链应该实现 Java 注解处理器 API，兼容用户已有的注解处理器。

应该允许纯 Java 类库通过使用特殊注解提供对 Kala 友好的 API。

## 特性

一些特性的草案<del>（卫星）</del>。待补充，待完善。欢迎通过 [Issue](https://github.com/Glavo/kala-lang/issues) 或者 [Discussions](https://github.com/Glavo/kala-lang/discussions) 讨论。

这些特性细节以及可能的实现方式参见[实现](#实现)一节。

* 取消跨类常量折叠。

* 扩展交集类型/并集类型：允许交集类型（`A & B`）和并集类型（`A | B`）出现在所有其他类型可以出现的地方（变量声明，函数参数，函数返回值类型，等等）。

* 将 `void` 视为类型（待定）：允许 `void` 作为类型使用。

* `this` 类型

  ```java
  class BaseClass {
      Class<? extends this-type> getClass2() -> getClass();
      
      this-type append(Object obj) {...} 
      /*
          Equivalent to:
              @ReturnThis
              BaseClass append(Object obj){...}
       */
  }
  
  class MyClass extends BaseClass {
      void myMethod() {...}
  }
  
  MyClass myClass = new MyClass();
  BaseClass baseClass = myClass;
  
  myClass.getClass2();   // static type: Class<? extends MyClass>
  baseClass.getClass2(); // static type: Class<? extends BaseClass>
  
  myClass.append(...)
         .append(...)
         .myMethod();    //ok
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

  * 将有 Java Bean 风格的 Getter/Setter 方法的 Java 属性视为 Kala 属性。

    ```java
    class C {
        static String getField1() -> ...;
        static void setField1(String value) -> ...;
        
        String getField2() -> ...;
        
        boolean isField3() -> ...;
        void setField3(boolean value) -> ...;
    }
    
    C.field1;
    C.field1 = "str";
    
    var c = new C();
    c.field2;
    // c.field2 = "str"; // error
    
    c.field3;
    c.field3 = false;
    ```
  
  * 简化声明属性的语法。
  
    ```java
    class Test {
        public property String name = "Glavo";
        
        /* 
            Equivalent to:
                public property String name {
                    private field = "Glavo";
                    public get() -> field;
                    public set(value) -> field = value;
                }
        
            Or:
                private String name = "Glavo";
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
  // Seq.of("str0", "str1", "str2", "str3").sum(); // compile time error
  ```

* 嵌套方法

  ```java
  void outer(String arg) {
      void inner() {
          System.out.println(arg);
      }
      
      static int helper(int a, int b) {
          // System.out.println(arg); // compile time error
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

* 嵌套类分离声明（？）

  ```java
  class Seq<covariant E> { ... }
  static class Seq.Iterator<covariant E> { ... }
  ```

* 宏（待定?）

  ```java
  inline void fun(int arg) = macro funImpl;
  Expression<void> funImpl(Expression<int> argExpr) {...}
  ```

* 语法树（待定?）

  ```java
  Expression<int>  addExpr = '(1 + 2);
  Expression<void> printExpr = '{
    System.out.println($addExpr);
  };
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
    
    Java 中的 `return` 语句被视为类型为 `bottom-type` 的表达式。
    
    ```java
    var obj = a != null ? a : return null;
    ```
    
  * `throw` 表达式
    
    Java 中的 `throw` 语句被视为类型为 `bottom-type` 的表达式。
    
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
    trait TraitA<T> {
        abstract static T create();
        
        abstract void foo();
    }
    
    class C {
        static C create() -> new C();
        
        void foo() {
            System.out.println("this is C");
        }
    }
    
    <T : TraitA> T createAndDoFoo() { // Equivalent to: <T> T createAndDoFoo() where T : TraitA
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

## 实现

### 兼容性

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

### this 类型

可用 `this-type` 关键字（暂定名称）引用的特殊类型。该类型表示该值被使用处的静态类型。

`this-type` 与协变的类型参数遵循类似的规则，不能作为方法参数类型使用。

可以通过 `ReturnThis` 注解（暂定名称）一个方法，该方法返回值类型应为所处上下文中 `this` 的静态类型的超类型（待定：或者 `void`），对于这样的方法，kala 将其返回值类型视为 `this-type`。

翻译方案：`this-type` 应被擦除为上下文中 `this` 的静态类型。`this-type` 作为方法返回值类型时，为方法添加 `ReturnThis` 注解。其他位置使用 `this-type` 的具体翻译方案待定。

### Bottom 类型

Bottom 类型是所有类型的子类型，可用 `bottom-type`（暂定名称）引用该类型。

被 `NoReturn` 所注解的方法返回值类型被视为 `bottom-type`。

作为方法返回值类型时，该方法会被翻译为被 `NoReturn` 注解的方法，其返回值类型为 `void`。当该方法覆盖或实现其他方法时，会合成与超类或超接口中函数签名相同的桥方法。

（fixme：该翻译方案下，向超类/超接口添加方法或变更方法签名可能导致链接错误或无法正常覆盖，需要重新编译。）

其他用途下翻译方案未定。

调用返回值类型为 `bottom-type` 的方法后的代码被视为不可达代码。

### 属性

kala 支持属性语法。

kala 属性的基础语法：

```java
属性修饰符* 'property' 属性类型 属性名称 属性体? ';'
```

可用于属性的修饰符有：

* `public`、`protected` 和 `private`：修饰属性的可访问性，默认为包内可见。
* `readonly`：软关键字，声明属性是只读的，没有 Setter 方法。
* `static`：声明属性是静态的。
* `abstract`：声明属性是抽象的。

属性可以使用 `值或类名::属性名` 引用，在其后使用 `.` 号访问属性的成员。

`值或类名.属性名` 等价于 `值或类名::属性名.get()`，`值或类名.属性名 = 值` 等价于 `值或类名::属性名.set(值)`。

属性体可以省略。省略属性体时，kala 会生成默认的属性体：

```java
如果属性是抽象的：
    {
        与属性相同的可访问性修饰符 abstract get();
        与属性相同的可访问性修饰符 abstract set(value);
    }
    
否则： 
    {
        与属性相同的可访问性修饰符 与属性相同的静态修饰符 field;
        与属性相同的可访问性修饰符 与属性相同的静态修饰符 get() -> field;
        与属性相同的可访问性修饰符 与属性相同的静态修饰符 set(value) -> field = value;
    }
```

属性体内可以使用声明这些属性成员：

* 字段声明：默认被翻译为类中名称为 `属性名$字段名` 的成员。字段名不可为 `field`。
* 方法声明：默认被翻译为类中名称为 `属性名$方法名` 的成员。
* `field` 声明：使用软关键字 `field` 声明启用属性的后备字段，默认字段名称与属性名相同。`field` 前可以使用访问性修饰符修饰，也可以可选的明确声明其类型（默认类型与属性类型相同，但可以声明为其他类型）。`field` 可以被修饰为 `final` 的。
* Getter 声明：使用软关键字 `get` 声明属性的 Getter。Getter 声明与方法声明类似，但不可以声明返回值类型，其返回值类型永远与属性类型相同。Getter 默认使用 Java Bean 风格的名称。
* Setter 声明：使用软关键字 `set` 声明非 `readonly` 属性的 Setter。Setter 声明与方法声明类似，但不可以声明返回值或参数类型。其参数列表只能声明单独一个名称，用于指代其参数。其参数类型永远与属性类型相同，返回值类型永远为 `void`。Setter 默认使用 Java Bean 风格的名称。

只有抽象类或接口可以有抽象属性，只有抽象属性才可以有抽象的属性成员。

接口中的抽象属性的成员函数（包括 Getter 和 Setter）可以声明为 `default` 并提供默认实现。

### Record

增强 `record` ，允许支持指定其他超类。

在超类显式或隐式被指定为 `java.lang.Record` 时才会被真正视为 Java 中的 `record` 进行翻译，完全遵循 [JEP 395](https://openjdk.java.net/jeps/395) 中的限制。

当使用其他类作为超类时，不为类生成 `Record_attribute`。

（多目标翻译：在未显式指定超类时，为高于 Java 16 的目标生成超类为 `java.lang.Record` 版本，为低于 Java 16 的目标生成超类为 `java.lang.Object` 的版本）

### Sealed Class

（待确认）应该可以为任意版本目标生成 `PermittedSubclasses` 属性，低版本平台会自动忽略该项，但不确认 Java 15/16 是否必须在运行时开启 `--enable-preview` 选项。

### Primitive Class

必须保证实现多目标翻译才能正确应对 Primitive Class。

对于 Primitive Class 在低版本平台上的翻译，应该为其生成工厂方法（`<new>` 方法），将对该类型的 `new` 调用翻译为工厂方法调用而非构造器调用。

TODO：对 `Q` 类型的兼容方案。

### 通用泛型

使用 Universal Generics 的 [JEP 草案](https://openjdk.java.net/jeps/8261529)中所描述的方式翻译。

### 空安全

对于引用类型，`T?` 和 `T!` 与 `T` 的擦除一致，由编译器进行流敏感分析验证可空性。

对于原始类型，`T?` 应该被擦除为其包装器类型。

`T` 与 `T? ` ，`T` 与 `T! `，两组类型之间允许非受检的转化。。

允许以下不同的参数化类型之间的互相转化：

* 将参数化类型的类型参数由平台类型修改为对应的可空类型或非空类型，反之亦然。

  ```java
  Seq<String > seq  = ...;
  Seq<String!> seq1 = seq; // ok
  Seq<String?> seq2 = seq; // ok
  ```

* 将参数化类型的类型通配符边界由平台类型修改为对应的可空类型或非空类型，反之亦然。

  ```java
  Seq<? extends String > seq  = ...;
  Seq<? extends String!> seq1 = seq; // ok
  Seq<? extends String?> seq2 = seq; // ok
  ```

* 递归地将以上转化规则应用于参数化类型的所有类型参数或通配符边界。

  ```java
  Seq<Map<String,  ? extends CharSequence >> seq  = ???;
  Seq<Map<String?, ? extends CharSequence?>> seq1 = seq;
  Seq<Map<String!, ? extends CharSequence!>> seq2 = seq;
  ```

以上规则也适用于方法覆盖。
