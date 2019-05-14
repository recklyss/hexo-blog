---
title: Java序列化和transient关键字的理解与学习
date: 2019-05-11 20:44:31
categories: [Java基础]
tags: [序列化,Java基础,transient关键字]
---

#### Java序列化

![Java序列化过程](xuliehua.png)

> 在我们平时开发中，经常会遇到将对象转成可传输的字节流或者保存在某些文件中去使用的场景。这种将对象转成字节序列的过程称之为序列化。反之，将字节序列转成对象的过程我们称之为反序列化。序列化是保存与传输对象相关数据的一种方式，并不是保存类信息的一种方式。

<!--more-->

##### Java中如何进行序列化与反序列化

  - 在Java中，对象一般是无法进行序列化与反序列化的。而使得对象能够被序列化的方式也很简单，即实现接口 `Serializable` 。如下代码即将对象序列化以及反序列化的过程。

  ```
    public class TestSerializable implements Serializable {
        private static final long serialVersionUID = 1L;
        private Integer age;
        private String name;
        TestSerializable() {
            age = 20;
            name = "aachuanpu";
        }
        public static void main(String[] args) throws IOException, ClassNotFoundException {
            TestSerializable test = new TestSerializable();
            File file = new File("e:/test.txt");
            ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(file));
            out.writeObject(test);
            ObjectInputStream in = new ObjectInputStream(new FileInputStream(file));
            TestSerializable newTest = (TestSerializable) in.readObject();
            System.out.println(newTest.name);
        }
    }
  ```

##### serialVersionUID的作用

  - serialVersionUID作为实现序列化接口的一个非必须非必须声明的静态常量经常不被开发者所重视，忘记声明。其实serialVersionUID的作用是为了保证序列化之前和之后的对象是同一对象。我们知道JVM判断对象是否相同是根据对象的类路径全限定名确定的，而虚拟机决定一个对象是否允许序列化和反序列化成这个类还取决于其serialVersionUID是否一致。不一致的话会导致`java.io.InvalidClassException的异常`，也可以不指定serialVersionUID，如果不指定的话java会根据class计算serialVersionUID。
  - 对于两个相同的类及拥有相同的serialVersionUID，如果两个类字段不一致也会序列化和反序列化成功。这时Java会在反序列化的时候忽略掉不一致的字段。

##### 静态变量的序列化

  - 在序列化的时候，静态变量能够被序列化成功吗？
  ```
  public class TestSerializable implements Serializable {
      private static final long serialVersionUID = 1L;
      public static String staticName;
      private Integer age;
      private String name;
      TestSerializable() {
          age = 20;
          name = "aachuanpu";
      }
      public static void main(String[] args) throws IOException, ClassNotFoundException {
          TestSerializable test = new TestSerializable();
          TestSerializable.staticName = "name11111";
          File file = new File("e:/test.txt");
          ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(file));
          out.writeObject(test);
          TestSerializable.staticName = "name222";
          ObjectInputStream in = new ObjectInputStream(new FileInputStream(file));
          TestSerializable newTest = (TestSerializable) in.readObject();
          System.out.println(newTest.name);
          System.out.println(TestSerializable.staticName);
      }
  }
  ```

  - 以上代码会输出什么？

  ```
  aachuanpu
  name222

  Process finished with exit code 0
  ```

  - 如上所见：将对象序列化之后，修改静态变量的值，再将对象反序列化，输出的静态变量的值是修改之后的。即序列化保存的是对象的状态，静态变量属于类，因此序列化并不保存静态变量。

##### transient关键字与自定义序列化

  - 对象的序列化是将对象中的数据写入本地文件或者用于网络传输的过程，但是很多时候会有一些数据无需进行序列化保存起来或者传输出去。我们可以使用`transient`关键字修饰成员变量。那么在Java序列化的时候就**不会使用Java本身的序列化方式对其进行序列化**。但是我们依然可以自定义自己的序列化行为对其进行序列化！

  **自定义序列化：** 定义自己的`writeObject`和`readObject`方法

  - 对于使用transient修饰的成员变量，可以编写`writeObject`和`readObject`方法实现对于该成员变量(不仅仅只是针对该成员变量)的自定义序列化。在编写`writeObject`和`readObject`方法的时候需要注意的地方在于：这俩方法没有在Object中定义，也没有在`Serializable`接口中声明，JVM是如何调用到这俩方法的呢？答案是通过反射，去根据方法名和参数寻找到相应的方法，找到之后会被ObjectOutputStream调用，没有这俩方法就调用默认的序列化呗。还有就是因为ObjectOutputStream使用getPrivateMethod，所以这些方法不得不被声明为priate以至于供ObjectOutputStream来使用。

  - 通过这种方法，我们实现自己的序列化与反序列化可以实现很多场景下的需求。比如网络传输的时候对于特殊字段进行加密等等。

  - 如下，你会发现我在这俩方法中调用了defaultWriteObject()和defaultReadObject()用于处理未被transient修饰的成员变量。

  ```
  public class TestSerializable implements Serializable {
      private static final long serialVersionUID = 1L;
      public static String staticName;
      private Integer age;
      private transient String name;
      TestSerializable() {
          age = 20;
          name = "aachuanpu";
      }
      public static void main(String[] args) throws IOException, ClassNotFoundException {
          TestSerializable test = new TestSerializable();
          TestSerializable.staticName = "name11111";
          File file = new File("e:/test.txt");
          ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(file));
          out.writeObject(test);
          TestSerializable.staticName = "name222";
          ObjectInputStream in = new ObjectInputStream(new FileInputStream(file));
          TestSerializable newTest = (TestSerializable) in.readObject();
          System.out.println(newTest.name);
          System.out.println(TestSerializable.staticName);
      }
      private void writeObject(ObjectOutputStream oos) throws IOException {
          oos.defaultWriteObject();
          name = "自定义名称";
          oos.writeObject(name);
          System.out.println("调用writeObject");
      }
      private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
          ois.defaultReadObject();
          String name = (String) ois.readObject();
          this.name = name;
          System.out.println("读出的name=" + name);
          System.out.println("调用readObject");
      }
  }
  ```
  输出如下：
  ```
  调用writeObject
  读出的name=自定义名称
  调用readObject
  自定义名称
  name222

  Process finished with exit code 0
  ```

##### 父类的序列化

  - 一个子类实现了 Serializable 接口，它的父类都没有实现 Serializable 接口，序列化该子类对象，然后反序列化后输出父类定义的某变量的数值，该变量数值与序列化时的数值不同。要想将父类对象也序列化，就需要让父类也实现Serializable 接口。如果父类不实现的话的，就需要有默认的无参的构造函数。 在父类没有实现 Serializable 接口时，虚拟机是不会序列化父对象的，而一个 Java 对象的构造必须先有父对象，才有子对象，反序列化也不例外。所以反序列化时，为了构造父对象，只能调用父类的无参构造函数作为默认的父对象。因此当我们取 父对象的变量值时，它的值是调用父类无参构造函数后的值。如果你考虑到这种序列化的情况，在父类无参构造函数中对变量进行初始化，否则的话，父类变量值都 是默认声明的值。

##### 常问：ArrayList中数组使用transient修饰为何还能被序列化

  **ArrayList源码：**
  ```
  /**
   * The array buffer into which the elements of the ArrayList are stored.
   * The capacity of the ArrayList is the length of this array buffer. Any
   * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
   * will be expanded to DEFAULT_CAPACITY when the first element is added.
   */
  transient Object[] elementData; // non-private to simplify nested class access
  ```
  transient用来表示一个域不是该对象序行化的一部分，当一个对象被序行化的时候，transient修饰的变量的值是不包括在序行化的表示中的。但是ArrayList又是可序行化的类，elementData是ArrayList具体存放元素的成员，用transient来修饰elementData，需要实现自己的序列化方式去处理。即对于数组中多余的空间不去进行序列化。如下：
  ```
  /**
   * Save the state of the <tt>ArrayList</tt> instance to a stream (that
   * is, serialize it).
   *
   * @serialData The length of the array backing the <tt>ArrayList</tt>
   *             instance is emitted (int), followed by all of its elements
   *             (each an <tt>Object</tt>) in the proper order.
   */
  private void writeObject(java.io.ObjectOutputStream s)
      throws java.io.IOException{
      // Write out element count, and any hidden stuff
      int expectedModCount = modCount;
      s.defaultWriteObject();

      // Write out size as capacity for behavioural compatibility with clone()
      s.writeInt(size);

      // Write out all elements in the proper order.
      for (int i=0; i<size; i++) {
          s.writeObject(elementData[i]);
      }

      if (modCount != expectedModCount) {
          throw new ConcurrentModificationException();
      }
  }

  /**
   * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
   * deserialize it).
   */
  private void readObject(java.io.ObjectInputStream s)
      throws java.io.IOException, ClassNotFoundException {
      elementData = EMPTY_ELEMENTDATA;

      // Read in size, and any hidden stuff
      s.defaultReadObject();

      // Read in capacity
      s.readInt(); // ignored

      if (size > 0) {
          // be like clone(), allocate array based upon size not capacity
          ensureCapacityInternal(size);

          Object[] a = elementData;
          // Read in all elements in the proper order.
          for (int i=0; i<size; i++) {
              a[i] = s.readObject();
          }
      }
  }
  ```
  **elementData是一个缓存数组，它通常会预留一些容量，等容量不足时再扩充容量，那么有些空间可能就没有实际存储元素，采用上诉的方式来实现序列化时，就可以保证只序列化实际存储的那些元素，而不是整个数组，从而节省空间和时间。**


##### 其余补充

[来自文章](https://bluepopopo.iteye.com/blog/486548) ← 点击链接查看参考博客

> 1.Write的顺序和read的顺序需要对应，譬如有多个字段都用wirteInt一一写入流中，那么readInt需要按照顺序将其赋值;

> 2.Externalizable,该接口是继承于Serializable ,所以实现序列化有两种方式。区别在于Externalizable多声明了两个方法readExternal和writeExternal，子类必须实现二者。Serializable是内建支持的也就是直接implement即可，但Externalizable的实现类必须提供readExternal和writeExternal实现。对于Serializable来说，Java自己建立对象图和字段进行对象序列化，可能会占用更多空间。而Externalizable则完全需要程序员自己控制如何写/读，麻烦但可以有效控制序列化的存储的内容。

> 3.正如Effectvie Java中提到的，序列化就如同另外一个构造函数，只不过是有由stream进行创建的。如果字段有一些条件限制的，特别是非可变的类定义了可变的字段会反序列化可能会有问题。可以在readObject方法中添加条件限制，也可以在readResolve中做。参考56条“保护性的编写readObject”和“提供一个readResolve方法”。

> 4.当有非常复杂的对象需要提供deep clone时，可以考虑将其声明为可序列化，不过缺点也显而易见，性能开销。
