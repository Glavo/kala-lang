# Kala Lang Design

本仓库用于讨论 kala 语言的设计与功能。

目前本仓库主要用途是记录对该语言的设想<del>（卫星），具体啥时候做出来还得看我要摸到啥时候</del>。讨论请前往本仓库的 [Issue 页](https://github.com/Glavo/kala-lang/issues)。

## 概述

Kala 语言目标是在完全兼容 Java 语言的基础上，提供更强大易用的语言功能。无需引入额外的运行时依赖。

Kala 语言应该能够完全兼容最新版本 Java 的语法，在此基础上扩展，并允许将编译目标设定为更低版本的 JVM 平台（譬如 Java 8），同时为主目标为低版本平台的程序适配高版本特性（譬如 Primitive Class）提供帮助。

## 特性

一些特性的草案<del>（卫星）</del>。待补充，待完善。欢迎通过 [Issue](https://github.com/Glavo/kala-lang/issues) 或者 [Discussions](https://github.com/Glavo/kala-lang/discussions) 讨论。

* 取消跨类常量折叠。

* 扩展交集类型/并集类型：允许交集类型（`A & B`）和并集类型（`A | B`）出现在所有其他类型可以出现的地方（变量声明，函数参数，函数返回值类型，等等）。

* 将 `void` 视为类型：允许 `void` 作为类型使用。

* Bottom 类型

* 属性语法

  * 允许使用属性访问语法（`o.a`）替代 Getter/Setter 方法（`o.getA()`/`o.setA(...)`）调用。

  * 简化声明属性的语法。（待定）（可能参考 Kotlin 或者 C#）（支持属性内部成员？）

    ```java
    class Test {
        public String name { get; set; }
        
        /* Or:
        public String name {
            private String field;
            get() -> field;
            set(value) -> field = value;
        }
        */
    
        public final String value {
            private String field;
            boolean isInitialized = false;
        
            private String initValue() {
                // ...
                return someString;
            }
        
            get() {
                return if(isInitialized) field else {
                    field = initValue();
                    isInitialized = true;
                    field;
                }
            }
        }
        
        public boolean valueIsInitialized() -> this::value.isInitialized;
    }
    ```
  * 抽象属性
    
    ```java
    interface I {
        String value {
            get();
        };
    }
    
    abstract class C {
        abstract String value {
            get();
        }
    }
    ```
  
* 单行方法

  ```java
  public String toString() -> "[" + this.name + "]";
  
  public static String[] newStringArray(int size) = String[]::new;
  
  public static Pair<String, int> pairOf(String, int) = Pair::new;
  ```

* 扩展方法

  ```java
  public static void printMe(String this) {
      System.out.println(this);
  }
  ```

* 嵌套方法

* 宏（待定）（可能参考 Scala 2/3）

* 增强 Record

  * 允许继承自其他父类

    * 以此可以实现在低于 Java 14 的平台上使用 Record。

      ```java
      public record Pair<A, B>(A first, B second) extends Object {}
      ```

* 使大多数语句成为表达式

  * Block 表达式（待定，可能与数组初始化器语法冲突）

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
    var obj = xxx ? "str" : return null;
    ```
    
  * `throw` 表达式
    
    ```java
    var obj = xxx ? "str" : throw new Exception();
    ```
  
* 空安全

  * 可空类型（`T?`）/不可空类型（`T!`）/平台类型（`T`）

    * 识别 Java 中的可空性注解（见 [Kotlin 编译器源码](https://github.com/JetBrains/kotlin/blob/master/core/compiler.common.jvm/src/org/jetbrains/kotlin/load/java/JvmAnnotationNames.kt)）。
    
  * 空处理增强
    
    ```java
    Seq<String>? seq = someFunction();
    System.out.println(seq?.toString() ?? "seq is null");
    seq ??= Seq.empty();
    ```

* 泛型增强

  * 泛型约束

    ```java
    <T> T f(T t) where T extends CharSequence & Comparable<? super T> -> t;
    // 等价于 <T extends CharSequence & Comparable<? super T>> T f(T t) { return t; }
    ```

  * 鸭子类型约束（待定？）

    ```java
    some-keyword HasFactory<T> {
        abstract static T create();
        
        abstract void foo();
    }
    
    class C {
        static C create() -> new C();
        
        void foo() {
            System.out.println("this is C");
        }
    }
    
    <T> T createAndDoFoo() where T : HasFactory { 
        var res = T.create(); 
        res.foo();
        return res;
    }
    
    C c = create(); // print "this is C"
    ```

    

  * 泛型具化（待定）

    ```java
    class TypeMirror {
        static <T> Type<T> typeOf() where T : Reified;
    }
    
    Type<Seq<String | int>> tpe = TypeMirror.typeOf();
    System.out.println(tpe); // print "kala.collection.Seq<java.lang.String | int>"
    
    <T> T[] newArray(int size) where T : Reified -> new T[size];
    ```

    