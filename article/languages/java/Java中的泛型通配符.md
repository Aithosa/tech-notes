# Java 中的泛型通配符

Java 中 `? extends XXX` 和 `? super XXX` 的区别

在 Java 中，`? extends XXX` 和 `? super XXX` 是泛型通配符的两种常见形式，用于表示集合中元素类型的上界和下界。它们在处理集合时提供了强大的类型安全机制。

`? extends XXX`：上界限定

含义： 表示集合中元素的类型必须是 XXX 类或其子类。

使用场景：

- 只读操作： 当你只需要从集合中读取元素，而不需要向集合中添加元素时，使用 ? extends XXX 是安全的。因为你无法向集合中添加任何类型，只能获取 XXX 或其子类的对象。
- 获取最大值或最小值： 如果你想从一个包含不同类型的数字的集合中获取最大值或最小值，可以使用 ? extends Comparable。

示例：

```Java
List<? extends Number> numbers = new ArrayList<Integer>();
```

> - 上述代码表示 numbers 集合中的元素只能是 Number 或其子类（如 Integer、Double 等）。

`? super XXX`：下界限定

含义： 表示集合中元素的类型必须是 XXX 类或其父类。

使用场景：

- 写入操作： 当你只需要向集合中添加元素，而不需要从集合中读取元素的具体类型时，使用 ? super XXX 是安全的。你可以向集合中添加 XXX 或其子类的对象。

示例：

```Java
List<? super Number> numbers = new ArrayList<Object>();
```

> - 上述代码表示 numbers 集合中的元素可以是 Number 或其父类（如 Object）。

两者区别总结

| 通配符        | 含义                                  | 使用场景                    |
| ------------- | ------------------------------------- | --------------------------- |
| ? extends XXX | 上界限定，元素类型必须是 XXX 或其子类 | 只读操作，获取最大值/最小值 |
| ? super XXX   | 下界限定，元素类型必须是 XXX 或其父类 | 写入操作                    |

注意事项

- PECS 原则： Producer Extends, Consumer Super。生产者（Producer）使用 `? extends`，消费者（Consumer）使用 `? super`。
- 类型擦除： 在编译时，泛型信息会被擦除，因此在运行时无法获取具体的类型参数。
- 协变与逆变：
  - 协变： 子类型可以赋值给父类型。`List<Integer>` 是 `List<Number>` 的子类型。
  - 逆变： 父类型可以赋值给子类型。`List<? super Number>` 是 `List<Object>` 的子类型。

示例：

```Java
public static void printList(List<?> list) {
    for (Object obj : list) {
        System.out.println(obj);
    }
}

public static void addToList(List<? super Number> list) {
    list.add(new Integer(3));
    list.add(new Double(3.14));
}
```

`printList` 方法可以接受任何类型的 List，因为我们只对元素进行读取操作。addToList 方法只能接受元素类型为 Number 或其父类的 List，因为我们只对元素进行写入操作。

总结：

`? extends XXX` 和 `? super XXX` 是 Java 泛型中非常重要的概念，它们提供了灵活的方式来处理集合中的元素类型。通过理解它们的区别和使用场景，可以编写出更加安全、高效的 Java 代码。

希望这个解释能帮助你更好地理解这两个概念！

如果你还有其他问题，欢迎随时提出。

想深入了解哪些方面呢？

- 泛型擦除的具体过程？
- PECS 原则的更深入解释？
- 泛型在集合框架中的应用？
- 其他相关问题？

## PECS 原则详解：生产者与消费者

什么是 PECS 原则？

PECS 是 Producer Extends, Consumer Super 的缩写，翻译过来就是 生产者继承，消费者超级。这个原则在 Java 泛型中非常重要，它帮助我们理解和使用泛型通配符 `? extends T` 和 `? super T`。

为什么引入生产者和消费者概念？

在泛型编程中，我们经常会遇到这样的场景：

- 生产者（Producer）： 一个方法或类产生（返回）一个泛型类型的对象。
- 消费者（Consumer）： 一个方法或类消费（使用）一个泛型类型的对象。

为了保证类型安全，并且让代码更加灵活，Java 引入了 PECS 原则。

PECS 原则的具体含义

- `? extends T`：

  - 表示一个类型是 T 或 T 的子类。
  - 适合用作 生产者，因为我们可以从这样的集合中获取 T 或其子类的对象。
  - 比如：`List<? extends Number>`，我们可以从中获取 Integer、Double 等。

- `? super T`：

  - 表示一个类型是 T 或 T 的父类。
  - 适合用作 消费者，因为我们可以向这样的集合中添加 T 或其子类的对象。
  - 比如：`List<? super Number>`，我们可以向其中添加 Integer、Double 等。

为什么是生产者 extends，消费者 super？

- 生产者 extends：

  - 当我们从一个集合中获取一个元素时，我们希望获取到的元素的类型尽可能具体。
  - 如果一个集合的元素类型是 `? extends Number`，那么我们获取到的元素至少是一个 `Number`，甚至可能是其子类。
  - 这样可以保证我们获取到的元素的类型是安全的。

- 消费者 super：

  - 当我们向一个集合中添加一个元素时，我们希望这个集合能够容纳尽可能多的类型。
  - 如果一个集合的元素类型是 `? super Number`，那么我们可以向其中添加 `Number` 或其子类的任何对象。
  - 这样可以提高代码的灵活性。

举例说明

```Java
// 生产者
List<? extends Number> getNumbers() {
    // 返回一个包含 Number 或其子类的 List
}

// 消费者
void addNumbers(List<? super Number> numbers) {
    numbers.add(new Integer(10));
    numbers.add(new Double(3.14));
}
```

总结

PECS 原则通过引入生产者和消费者概念，帮助我们更好地理解和使用泛型通配符。它提供了一种在保证类型安全的同时，又能够灵活地使用泛型的方法。

- 生产者 使用 `? extends`，强调读取操作，获取到的元素类型是 T 或其子类。
- 消费者 使用 `? super`，强调写入操作，可以向集合中添加 T 或其子类的对象。

牢记 PECS 原则，可以帮助你写出更加健壮和灵活的 Java 代码。

常见问题

- 为什么不能 `List<? extends Number> numbers = new ArrayList<Object>();`？
  - 因为 `Object` 不是 `Number` 的子类，违反了 `? extends Number` 的约束。
- 为什么不能 `List<? super Number> numbers.add(new String("hello"));`？
  - 因为 `String` 不是 `Number` 的子类或父类，违反了 `? super Number` 的约束。

希望这个解释能帮助你更好地理解 PECS 原则。

如果你还有其他问题，欢迎随时提问！

## 为什么当只读取时使用 `? extends XXX` 是安全的？

理解 `? extends XXX`

- 含义： 表示集合中元素的类型必须是 XXX 类或其子类。
- 作用： 提供了一种限制集合元素类型的上界的方式。

安全性的原因

1. 读取操作的限制：

   - 当你声明一个 `List<? extends Number>` 时，编译器知道集合中的元素至少是 Number 类型或其子类。
   - 由于是 `extends`，即子类，所以你只能获取 Number 或其子类的对象。
   - 因为无法确定具体的子类类型，所以你只能将获取到的对象视为 Number 类型进行操作。

2. 写入操作的禁止：

   - 尝试向这样的集合添加元素时，编译器会报错。
   - 这是因为编译器不知道集合中元素的具体子类类型，无法保证你添加的元素是否与集合中的其他元素兼容。

举例说明

```Java
List<? extends Number> numbers = new ArrayList<Integer>();
```

合法操作：

- `Number num = numbers.get(0);` // 获取第一个元素，并将其赋值给 Number 类型的变量
- `Object obj = numbers.get(0);` // 将元素作为 Object 类型处理

非法操作：

- `numbers.add(new Number());` // 编译错误，因为不知道集合中元素的具体子类类型
- `numbers.add(new Integer(10));` // 编译错误，同样原因

为什么是安全的？

- 类型安全： 由于编译器限制了写入操作，确保了集合中的元素类型始终保持一致，避免了类型转换错误。
- 代码可读性： 使用 ? extends 明确表达了集合元素的类型限制，提高了代码的可读性。

深入理解

- PECS 原则： Producer Extends, Consumer Super。生产者（Producer）使用 `? extends`，消费者（Consumer）使用 `? super`。
- 类型擦除： 泛型在编译时会被擦除，所以 `List<? extends Number>` 在运行时实际上是 List。
- 协变： 子类型可以赋值给父类型。`List<Integer>` 是 `List<Number>` 的子类。

总结

当我们只关心从集合中读取数据时，使用 `? extends` 可以保证类型安全，并且可以灵活地处理不同类型的子类。这是因为编译器会严格限制写入操作，确保集合中的元素类型始终保持一致。

换句话说，`? extends` 在读取操作时提供了一种安全且灵活的类型约束。

希望这个解释能帮助你更好地理解 `? extends` 的作用和安全性。

如果你还有其他问题，欢迎随时提问！

## 为什么当只添加元素时使用 `? super XXX` 是安全的？

理解 `? super XXX`

- 含义： 表示集合中元素的类型必须是 XXX 类或其父类。
- 作用： 提供了一种限制集合元素类型的下界的方式。

安全性的原因

1. 写入操作的灵活性：

   - 当你声明一个 `List<? super Number>` 时，编译器知道集合中的元素可以是 Number 或其父类（比如 Object）。
   - 由于是 `super`，即父类，所以你可以向其中添加 Number 或其子类（比如 Integer、Double）的对象。
   - 因为所有这些类型都兼容 Number 的父类，所以添加操作是安全的。

2. 读取操作的限制：

- 尝试从这样的集合中读取元素时，编译器会将元素类型视为 Object。
- 这是因为你不知道集合中具体存放的是什么类型的对象，只能以 Object 的形式来处理。

举例说明

```Java
List<? super Number> numbers = new ArrayList<Object>();
```

合法操作：

- `numbers.add(new Integer(10));`
- `numbers.add(new Double(3.14));`

不推荐的操作：

- `Number num = numbers.get(0);` // 虽然可以编译，但获取到的元素类型不确定，可能发生 ClassCastException

为什么是安全的？

- 类型兼容性： 由于 `? super` 表示的是父类，所以向集合中添加子类对象是安全的，因为子类对象可以向上转型为父类对象。
- 代码简洁： 使用 `? super` 可以避免显式地进行类型转换，简化代码。

深入理解

- PECS 原则： Producer Extends, Consumer Super。生产者（Producer）使用 `? extends`，消费者（Consumer）使用 `? super`。
- 类型擦除： 泛型在编译时会被擦除，所以 `List<? super Number>` 在运行时实际上是 List。
- 逆变： 父类型可以赋值给子类型。`List<? super Number>` 是 `List<Object>` 的子类型。

总结

当我们只关心向集合中添加数据时，使用 `? super` 可以保证类型安全，并且可以灵活地添加不同类型的子类对象。这是因为编译器会将集合中的元素类型视为一个共同的父类，从而保证添加操作的安全性。

换句话说，`? super` 在写入操作时提供了一种安全且灵活的类型约束。

注意事项

- 虽然使用 `? super` 可以向集合中添加不同类型的对象，但读取时需要小心，最好将获取到的对象强制转换为具体的类型，或者使用 `instanceof` 操作符进行类型判断。
- 在实际开发中，如果既需要添加元素又需要读取元素，通常会使用具体的泛型类型，而不是通配符。

希望这个解释能帮助你更好地理解 `? super` 的作用和安全性。

如果你还有其他问题，欢迎随时提问！

## 疑问

### 为什么声明一个 `List<? extends Number>` 时，往里添加子类元素就不能向上转型，而到了 `List<? super Number>` 就可以向上转型了?

深入探讨一下 `List<? extends Number>` 和 `List<? super Number>` 在添加元素时的不同行为。

核心原因：泛型通配符的本质和类型擦除

1. 泛型通配符的本质：

   - `? extends Number`：表示集合中元素的类型是 Number 或其子类。
   - `? super Number`：表示集合中元素的类型是 Number 或其父类。

2. 类型擦除：

- 在编译时，泛型信息会被擦除。也就是说，`List<? extends Number>` 和 `List<? super Number>` 在运行时都会被视为 `List`。

为什么 `List<? extends Number>` 不能添加子类元素？

想象一下，如果允许向 `List<? extends Number>` 中添加子类元素，会发生什么？

- 不确定性： 编译器无法确定集合中元素的具体类型。它可能是一个 `Integer` 的 List，也可能是一个 `Double` 的 List。
- 类型安全问题： 如果你向一个 `List<Integer>` 中添加一个 `Double`，就会发生类型转换异常。

举例：

```Java
List<? extends Number> numbers = new ArrayList<Integer>();
numbers.add(new Double(3.14)); // 编译错误
```

原因分析：

- 虽然 `Double` 是 `Number` 的子类，但编译器无法保证 `numbers` 实际指向的是 `List<Double>`。
- 如果 `numbers` 指向的是 `List<Integer>`，那么添加 Double 就会导致类型不匹配。

为什么 `List<? super Number>` 可以向上转型添加元素？

- 向上转型： 子类可以向上转型为父类。
- 类型兼容性： 由于 `? super Number` 表示集合中元素的类型是 Number 或其父类，所以向其中添加 Number 或其子类的对象都是安全的。

举例：

```Java
List<? super Number> numbers = new ArrayList<Object>();
numbers.add(new Integer(10));
numbers.add(new Double(3.14));
```

原因分析：

- `Integer` 和 `Double` 都是 `Number` 的子类，可以向上转型为 `Number`。
- 由于 `numbers` 的元素类型至少是 `Number`，所以添加这些子类对象是安全的。

总结

- `? extends Number`： 提供了读取元素的安全性，但限制了写入操作。
- `? super Number`： 提供了写入元素的灵活性，但读取元素时需要小心类型转换。

关键点：

- 类型擦除： 泛型信息在运行时被擦除，导致编译器无法确定集合中元素的精确类型。
- 向上转型： 子类可以向上转型为父类，但父类不能向下转型为子类。
- 类型安全： 编译器会尽力保证类型安全，但在某些情况下，开发者需要手动进行类型转换或检查。

通俗地讲：

- `? extends Number` 就像一个只进不出的箱子，你只能从里面拿东西出来，但不能往里面放任何东西。
- `? super Number` 就像一个大口袋，你可以往里面放各种各样的东西（只要它们是 Number 或其子类），但当你拿出来的时候，你只能将其视为一个普通的对象。

举个形象的例子：

想象一个装满各种水果的盒子。

- `List<? extends Fruit>`：这个盒子里的水果可能是苹果、梨、香蕉等。你可以从盒子里拿水果吃，但你不能往里面放其他东西，比如蔬菜。
- `List<? super Fruit>`：这个盒子可以装任何水果，也可以装水果的包装箱。你可以往里面放苹果、梨、香蕉，甚至可以放一个装满水果的篮子。但是当你从盒子里拿东西出来时，你只能把它当作一个普通的水果来处理，不能确定它具体是什么水果。

希望这个解释能帮助你更好地理解 `? extends` 和 `? super` 在添加元素时的行为。

### `List<? extends Number> numbers = new ArrayList<Number>();` 向里面添加 Integer 和 Double 可以向上转型为 Number

将 `List<? extends Number> numbers = new ArrayList<Number>();` 这样改动是非常合理的。

原因分析：

- `? extends Number` 的含义： 这表示集合中元素的类型必须是 Number 或其子类。
- `ArrayList<Number>`: 这明确指定了集合中元素的类型就是 Number。
- 向上转型： Integer 和 Double 都是 Number 的子类，可以向上转型为 Number，因此可以顺利添加到这个集合中。

为什么这样改更好？

- 更明确的类型: `ArrayList<Number>` 直接告诉编译器，集合中存储的都是 Number 类型的对象，这样可以避免一些不必要的类型检查。
- 更好的可读性: 直接使用 `ArrayList<Number>` 比 `ArrayList<Object>` 更直观，更容易理解集合中元素的类型。

总结：

- `List<? extends Number> numbers = new ArrayList<Number>();` 是一种常见的用法，它表示一个可以存储 Number 或其子类的集合。
- 这种写法既保证了类型的灵活性，又提供了明确的类型信息，提高了代码的可读性和安全性。

与 `List<? super Number> numbers = new ArrayList<Number>();` 的区别：

`? extends Number` 表示集合中元素的类型是 Number 或其子类，适合用于读取操作。
`? super Number` 表示集合中元素的类型是 Number 或其父类，适合用于写入操作。

选择哪种方式取决于你的具体需求：

- 如果需要读取集合中的元素，并且知道元素的类型是 Number 或其子类，那么使用 `? extends Number` 更合适。
- 如果需要向集合中添加 Number 或其子类的元素，那么使用 `? super Number` 更合适。

在你的例子中，既然你只需要向集合中添加 Integer 和 Double，那么使用 `? super Number` 也是可以的，但是 `ArrayList<Number>` 这种写法更加明确和直观。

示例：

```Java
List<Number> numbers = new ArrayList<>();
numbers.add(new Integer(10));
numbers.add(new Double(3.14));
```

这样写法的好处是：

- 简洁明了: 直接使用 `ArrayList<Number>`，代码更简洁。
- 类型安全: 编译器可以更好地进行类型检查，避免潜在的类型错误。

总结来说，`List<Number> numbers = new ArrayList<>();` 是一种既灵活又安全的写法，适用于存储 Number 或其子类对象的场景。

## 我的想法

以上内容其实是 Gemini 告诉我的，里面最一困惑我的问题就是`List<? extends Number> numbers = new ArrayList<Number>();`这种写法到底能不能往里面添加`Integer`和`Double`元素。

目前得到的答案是**可以**，因为添加的元素可以向上转换成为`Number`。但是这是一种特殊情况，`List` 中可以是 `Number` 及其子类，如果在更普遍的情况下，我们没法确定具体的类型，所以很可能会有类型冲突的风险。

## 之前的认识误区

所以“当你声明一个 `List<? extends Number>` 时，并尝试向这样的集合添加元素时，编译器会报错”这个说法是有条件的。

为什么会出现这种误解？

这种误解可能源于对 `? extends Number` 的误解。很多人会认为 `? extends Number` 表示一个只能读取，不能写入的集合。但实际上，这是一种过于简化的理解。

`? extends Number` 的真正含义是： 集合中元素的类型是 Number 或其子类。

当我们把 `ArrayList` 初始化为 `Number` 类型时：

- 编译器知道集合中所有元素的类型都是 `Number` 或其子类。
- 因此，向这个集合添加 `Integer` 或 `Double` 这样的子类是完全合法的，因为它们可以向上转型为 `Number`。

总结

- `List<? extends Number>` 不代表完全不能添加元素，而是对添加元素的类型有限制。
- 当 `ArrayList` 初始化为 `Number` 类型时，就可以向其中添加 `Number` 或其子类。
- `? extends Number` 更适合用于读取操作，因为你可以保证读取出来的元素都是 `Number` 或其子类。

注意事项

虽然 `List<? extends Number> numbers = new ArrayList<Number>();` 可以添加 `Integer` 和 `Double`，但要注意以下几点：

- 读取时需要类型转换： 如果你想将读取出来的元素作为 `Integer` 或 `Double` 来使用，需要进行显式的类型转换。
- 泛型擦除： 在运行时，所有的泛型信息都会被擦除，所以编译器无法保证集合中所有元素的类型都是相同的。

所以，在使用 `? extends Number` 时，一定要清楚它的含义，并根据实际情况选择合适的方式来使用它。

## 其他情况

以上只是有初始化部分的情况，如果是作为方法参数的情况下（如形参），并没有具体的初始化部分又如何，如：

```Java
void addNumbers(List<? extends Number> numbers) {
   numbers.add(new Integer(10));
   numbers.add(new Double(3.14));
}
```

这个例子确实很有意思，它涉及到了 PECS 原则在实际应用中的一个常见误区。

为什么这个例子不能编译？

核心原因在于 `List<? extends Number>` 的限制。

- `? extends Number` 的含义： 它表示集合中元素的类型是 `Number` 或其子类。但更重要的是，编译器无法确定这个具体的子类是什么。
- 添加元素的限制： 由于编译器不知道具体的子类，为了保证类型安全，它禁止向 `List<? extends Number>` 中添加任何元素，即使是 `Number` 或其子类。

举个例子：

假设你有一个 `List<Integer>`，它完全符合 `List<? extends Number>` 的定义。如果你允许向这个 `List` 中添加 `Double`，那么就会出现类型不匹配的问题。

如何正确处理？

如果你确实需要向一个 `List<? extends Number>` 中添加元素，那么通常情况下是不可能的。这是因为 `? extends Number` 的设计初衷就是为了保证读取操作的安全性，而限制了写入操作。

解决方法：

- 使用 `List<Number>`： 如果你确定集合中存储的都是 `Number` 类型，那么直接使用 `List<Number>` 就可以了。这样你可以自由地添加 Integer、Double 等任何 Number 的子类。
- 使用泛型方法： 如果需要一个更加灵活的方法，可以定义一个泛型方法，将 List 作为参数，然后在方法内部进行类型检查。
- 使用协变返回类型： 如果一个方法返回一个 `List<? extends Number>`，那么这个方法的返回值可以是一个 `List<Integer>`、`List<Double>` 等。

总结

- `List<? extends Number>` 主要用于读取操作，限制了写入操作。
- 如果需要向集合中添加元素，通常需要使用 `List<Number>` 或其他更具体的类型。
- PECS 原则 是为了保证类型安全，避免在运行时出现类型转换异常。

回到你的例子：

```Java
void addNumbers(List<? extends Number> numbers) {
    numbers.add(new Integer(10));
    numbers.add(new Double(3.14));
}
```

这段代码之所以不能编译，就是因为违反了 `List<? extends Number>` 的限制。如果你想实现类似的功能，可以考虑使用泛型方法或者直接使用 `List<Number>`。

希望这个解释能帮助你更好地理解 `? extends Number` 的用法。

如果你还有其他问题，欢迎随时提出！
