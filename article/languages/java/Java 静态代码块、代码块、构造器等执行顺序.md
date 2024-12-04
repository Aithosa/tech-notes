# Java 静态代码块、代码块、构造器等执行顺序

## 结论先行

静态代码块 → 普通代码块 → 构造方法

父类静态代码块 → 子类静态代码块 → 父类普通代码块 → 父类构造方法 → 子类普通代码块 → 子类构造方法

Java 初始化顺序

1. 在 new B()一个实例时首先要进行类的装载。（类只有在使用 New 调用创建的时候才会被 java 类装载器装入）
2. 在装载类时，先装载父类 A，再装载子类 B
3. 装载父类 A 后，完成静态动作（包括静态代码和变量，它们的级别是相同的，安装代码中出现的顺序初始化）
4. 装载子类 B 后，完成静态动作
   类装载完成，开始进行实例化
5. 在实例化子类 B 时，先要实例化父类 A
6. 实例化父类 A 时，先成员实例化（非静态代码）
7. 父类 A 的构造方法
8. 子类 B 的成员实例化（非静态代码）
9. 子类 B 的构造方法

## 代码后置

### 单个类

执行顺序：静态代码块 → 普通代码块 → 构造方法

```Java
/**
 * 静态代码块 -> 代码块 -> 构造方法
 *
 */
public class Parent {
    static int index = 0;

    static {
        System.out.println("父类静态代码块: " + index);
        index++;
    }

    {
        System.out.println("父类中的代码块：" + index);
        index++;
    }

    public Parent() {
        System.out.println("父类构造方法：" + index);
        index++;
    }

    // 显示调用才执行
    public static void test () {
        System.out.println("父类静态方法：" + index);
        index++;
    }

    /**
     * 父类静态代码块: 0
     * 父类中的代码块：1
     * 父类构造方法：2
     * 父类静态方法：3
     * @param args
     */
    public static void main(String[] args) {
        Parent parent = new Parent();
        parent.test();
    }
}
```

### 父子类

执行顺序：父类静态代码块 → 子类静态代码块 → 父类普通代码块 → 父类构造方法 → 子类普通代码块 → 子类构造方法

```Java
/**
 * 父类静态代码块 -> 子类静态代码块 -> 父类普通代码块 -> 父类构造方法 -> 子类普通代码块 -> 子类构造方法
 *
 */
public class Child extends Parent{
    static int index = 0;

    static {
        System.out.println("子类静态代码块: " + index);
        index++;
    }

    {
        System.out.println("子类中的代码块：" + index);
        index++;
    }

    public Child () {
        System.out.println("子类构造方法：" + index);
        index++;
    }

    // 显示调用才执行
    public static void test () {
        System.out.println("子类静态方法：" + index);
        index++;
    }

    /**
     * 父类静态代码块: 0
     * 子类静态代码块: 0
     * 父类中的代码块：1
     * 父类构造方法：2
     * 子类中的代码块：1
     * 子类构造方法：2
     * 子类静态方法：3
     * @param args
     */
    public static void main(String[] args) {
        Child child = new Child();
        Child.test();
    }
}
```

## 变量与代码块的执行顺序

静态的是随着类的加载而执行，普通的则是实例化的时候执行：

- 静态早于普通
- 谁先声明谁先执行：
  - 静态变量与静态代码块谁先声明谁先执行；
  - 普通变量与普通代码块谁先声明谁先执行；

```Java
/**
 * 静态变量与静态代码块谁先声明谁先执行
 */
public class Parent {
    static {
        System.out.println("父类静态代码块: " + index);
        // 编译错误， index还未声明
        index++;
    }

    static int index = 0;

    public static void main(String[] args) {
        Parent parent = new Parent();
    }
}
```
