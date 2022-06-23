## 函数式接口

```
@FunctionalInterface
```

- **1、该注解只能标记在"有且仅有一个抽象方法"的接口上。**
- **2、JDK8接口中的静态方法和默认方法，都不算是抽象方法。**
- **3、接口默认继承java.lang.Object，所以如果接口显示声明覆盖了Object中方法，那么 也不算抽象方法。**

## lambda

```
Thread t = new Thread(() -> {
    System.out.println();
});
Thread t = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println();
    }
});
```

()->{}

小括号表示形参，大括号表示方法体，如果只有一条语句，可以省略大括号

```java
Thread t = new Thread(() -> System.out.println());
```

如果需要实现的方法中有形参，在箭头前的括号中引入即可

```
public class TMain {
    public static void main(String[] args) {
        DemoI i = (String str1, String str2) -> {
            return str1 + str2;
        };
        System.out.println(i.test("1", "2"));
    }
}
@FunctionalInterface
interface DemoI{
    String test(String str1, String str2);
}
```

形参只有一个的时候，小括号可以省略

```
public class TMain {
    public static void main(String[] args) {
        DemoI x = i -> i++;
        x.test(1);
    }
}
@FunctionalInterface
interface DemoI{
    void test(int i);
}
```

只有返回值语句，可以省略return；语句的值会作为返回值

```
public class TMain {
    public static void main(String[] args) {
        DemoI x = i -> i++;
        System.out.println(x.test(1));
    }
}
@FunctionalInterface
interface DemoI{
    int test(int i);
}
```

#### function包中提供了很多函数式接口

##### Consumer消费型接口



##### Predicate断言式接口



##### Supplier供给型接口



##### Function函数式接口



#### 方法引用



##### 静态方法引用

类名::staticMethodName

//使用lambda表示已经存在的静态方法，如果静态方法的签名符合函数式接口中抽象方法的要求（出入参一致），可以使用方法引用来替代lambda

下面使用已经存在的方法来替代Comparator中应实现的compare方法

```
List<Integer> l = new ArrayList<>();
l.sort(Integer::compare);
//这是Integer中的静态方法
public static int compare(int x, int y) {
	return (x < y) ? -1 : ((x == y) ? 0 : 1);
}
```

##### 实例方法引用

instance::method

使用已有的方法作为Runnable中的run方法

```
R r = new R();
Thread t = new Thread(r::size);
Thread t1 = new Thread(()->{
	r.size();
});
```

##### 对象方法引用

类名::method

lambda表达式中第一个参数是调用者本身，第二个参数是其他参数时可以使用

```
BiPredicate<String, String> n = String::equals;
n.test("", "1");
```

##### 构造方法引用

类名::new

```
Supplier<R> s = R::new;
List<R> l = new ArrayList<>();
l.add(s.get());
```