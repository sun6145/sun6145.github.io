---
layout: post
title: Set集合
abbrlink: 126e19ab11ad41fab9daaebd86dfb014
tags:
  - java
categories:
  - JDK
  - 基础篇
date: 1745339269520
updated: 1746417445722
---

Java 集合框架通过统一接口和多样化实现平衡了灵活性与效率。

<!-- more -->

***

## set 接口

### 特点

- Set 集合: 可以保证元素的**唯一性** , 但是**不保证有序**(有序: 指的是存储和取出的顺序一致)

* 注意:set 集合在存储元素的时候,需要对元素进行处理,按照一定的算法对元素进行排序,而有的时候我们的添加元素的顺序巧号是 set 集合计算完毕以后的顺序,但是这也不能说明是有序的

* 获取 7 个不重复的随机数

  - `(int)(Math.rando*(n-m+1)+m)` 产生任意范围内\[m,n]的随机数

  ```java
  //获取7个不重复的随机数
  public static void main(String[] args) {
      // Set
      Set<Integer> set = new HashSet<>();
      int r;
      while(true) {
          r = (int)(Math.random()*(30-1+1)+1);
          set.add(r);// int ->Integer
          if(set.size() == 7) {
              break;
          }
      }
      set.forEach(System.out::println);
  }
  ```

## HashSet

- 特点

  - 底层数据结构是哈希表，哈希表结构底层依赖:hashCode()和 equals()方法。
  - HashSet 集合保证元素的唯一性依赖于两个方法一个是 hashCode 方法,一个是 equals 方法如果，你认为对象的成员变量值相同即为同一个对象的话，你就应该重写这两个方法。
  - HashSet 实现了 Set 接口，不允许插入重复的元素，允许包含一个 null 元素，且不保证元素迭代顺序，特别是不保证该顺序恒久不变。Set 集合的元素无序,但是,作为集合来说,它肯定有自己的存储顺序。
  - HashSet 的代码十分简单，去掉注释后的代码不到两百行。HashSet 底层是通过 HashMap 来实现的。
  - 在进行比较的时候先比较的是对象的 hashCode 值,如果对象的 hashCode 值是相同的,那么在调用 equals 方法比较.

  ```java
  @Override
  public int hashCode() {
      // 哈希地址
      return name.hashCode() + no;
  }

  @Override
  public boolean equals(Object obj) {
      // this ,obj
      // obj _ > Employee
      Employee e = (Employee)obj;
      return this.no == e.no && this.name.equals(e.name);
  }
  ```

- **如何保证元素唯一性**

  - 而我们的字符串是重写 hashCode 方法和 equals 方法,那么在这里相同的字符串的 hashCode 值就是相同的,并且 equals 方法比较是内容。当 hashCode 值相同,并且调用 equals 方法返回的是 true 的时候,说明两个对象是同一个,于是不进行存储,即就保证了元素的唯一性

- **使用 HashSet 集合存储自定义对象,保证元素的唯一性**

  - 如果两个对象的成员变量是相同的,那么我们认为就是同一个对象

  ```java
  // 创建集合对象
  Set<String> set = new HashSet<String>();

  // 添加元素
  set.add("hello");
  set.add("java");
  set.add("world");
  set.add("java");
  set.add("world");

  // 遍历
  // 方式一:迭代器
  Iterator<String> it = set.iterator();
  while (it.hasNext()) {
  	String s = it.next();
  	System.out.println(s);
  }
  //方式二:
  for(String s : set){
  	System.out.println(s);
  }
  结果:
  	java
  	world
  	hello
  // 无序 唯一
  ```

## HashSet 如何去重

- 在向 HashSet 添加元素时，HashSet 会将该操作转换为向 HashMap 添加键值对，
- 如果 HashMap 中包含 key 值与待插入元素相等的键值对
  - （hashCode() 方法返回值相等，通过 equals() 方法比较也返回 true）
  - 则待添加的键值对的 value 会覆盖原有数据，但 key 不会有所改变，
  - 因此如果向 HashSet 添加一个已存在的元素时，元素不会被存入 HashMap 中，从而实现了 HashSet 元素不重复的特征。

### hashCode()方法

- 由于成员变量值影响了哈希值，所以我们把成员变量值相加即可
  `this.name.hashCode() + this.age;`

- 可能出现下面情况,对象不同,但 hashCode 一样,会进入 equals 方法
  `// s1:name.hashCode()=40,age=30`
  `// s2:name.hashCode()=20,age=50`

- 尽可能的区分,我们可以把它们乘以一些整数

  ```java
  @Override
  public int hashCode() {
  	final int prime = 31;
  	int result = 1;
  	result = prime * result + age;
  	result = prime * result + ((name == null) ? 0 : name.hashCode());
  	return result;
  }

  System.out.println("hello".hashCode());		//99162322
  System.out.println("hello".hashCode());		//99162322
  System.out.println("world".hashCode());		//113318802
  ```

### equals 方法

- 先比较对象地址值，类名，最后属性的值。

  ```java
  @Override
  public boolean equals(Object obj) {
  	if (this == obj)
  		return true;
  	if (obj == null)
  		return false;
  	if (getClass() != obj.getClass())
  		return false;
  	Student other = (Student) obj;
  	if (age != other.age)
  		return false;
  	if (name == null) {
  		if (other.name != null)
  			return false;
  	} else if (!name.equals(other.name))
  		return false;
  	return true;
  }
  ```

## HashSet 源码分析

- 源码分析如下所示

  ```java
  public class HashSet<E>
      extends AbstractSet<E>
      implements Set<E>, Cloneable, java.io.Serializable{

      //序列化ID
      static final long serialVersionUID = -5024744406713321676L;

      //HashSet 底层用 HashMap 来存放数据
      //Key值由外部传入，Value则由 HashSet 内部来维护
      private transient HashMap<E,Object> map;

      //HashMap 中所有键值对都共享同一个值
      //即所有存入 HashMap 的键值对都是使用这个对象作为值
      private static final Object PRESENT = new Object();

      //无参构造函数，HashMap 使用默认的初始化大小和装载因子
      public HashSet() {
          map = new HashMap<>();
      }

      //使用默认的装载因子，并以此来计算 HashMap 的初始化大小
      //+1 是为了弥补精度损失
      public HashSet(Collection<? extends E> c) {
          map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
          addAll(c);
      }

      //为 HashMap 自定义初始化大小和装载因子
      public HashSet(int initialCapacity, float loadFactor) {
          map = new HashMap<>(initialCapacity, loadFactor);
      }

      //为 HashMap 自定义初始化大小
      public HashSet(int initialCapacity) {
          map = new HashMap<>(initialCapacity);
      }

      //此构造函数为包访问权限，只用于对 LinkedHashSet 的支持
      HashSet(int initialCapacity, float loadFactor, boolean dummy) {
          map = new LinkedHashMap<>(initialCapacity, loadFactor);
      }

      //将对 HashSet 的迭代转换为对 HashMap 的 Key 值的迭代
      public Iterator<E> iterator() {
          return map.keySet().iterator();
      }

      //获取集合中的元素数量
      public int size() {
          return map.size();
      }

      //判断集合是否为空
      public boolean isEmpty() {
          return map.isEmpty();
      }

      //判断集合是否包含指定元素
      public boolean contains(Object o) {
          return map.containsKey(o);
      }

      //如果 HashSet 中不包含元素 e，则添加该元素，并返回 true
      //如果 HashSet 中包含元素 e，则不会影响 HashSet ，并返回 false
      //该方法将向 HashSet 添加元素 e 的操作转换为向 HashMap 添加键值对
      //如果 HashMap 中包含 key 值与 e 相等的结点（hashCode() 方法返回值相等，通过 equals() 方法比较也返回 true）
      //则新添加的结点的 value 会覆盖原有数据，但 key 不会有所改变
      //因此如果向 HashSet 添加一个已存在的元素时，元素不会被存入 HashMap 中
      //从而实现了 HashSet 元素不重复的特征
      public boolean add(E e) {
          return map.put(e, PRESENT)==null;
      }

      //移除集合中的元素 o
      //如果集合不包含元素 o，则返回 false
      public boolean remove(Object o) {
          return map.remove(o)==PRESENT;
      }

      //清空集合中的元素
      public void clear() {
          map.clear();
      }

      @SuppressWarnings("unchecked")
      public Object clone() {
          try {
              HashSet<E> newSet = (HashSet<E>) super.clone();
              newSet.map = (HashMap<E, Object>) map.clone();
              return newSet;
          } catch (CloneNotSupportedException e) {
              throw new InternalError(e);
          }
      }

      private void writeObject(java.io.ObjectOutputStream s)
          throws java.io.IOException {
          // Write out any hidden serialization magic
          s.defaultWriteObject();

          // Write out HashMap capacity and load factor
          s.writeInt(map.capacity());
          s.writeFloat(map.loadFactor());

          // Write out size
          s.writeInt(map.size());

          // Write out all elements in the proper order.
          for (E e : map.keySet())
              s.writeObject(e);
      }

      /**
       * Reconstitute the <tt>HashSet</tt> instance from a stream (that is,
       * deserialize it).
       */
      private void readObject(java.io.ObjectInputStream s)
          throws java.io.IOException, ClassNotFoundException {
          // Read in any hidden serialization magic
          s.defaultReadObject();

          // Read capacity and verify non-negative.
          int capacity = s.readInt();
          if (capacity < 0) {
              throw new InvalidObjectException("Illegal capacity: " +
                                               capacity);
          }

          // Read load factor and verify positive and non NaN.
          float loadFactor = s.readFloat();
          if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
              throw new InvalidObjectException("Illegal load factor: " +
                                               loadFactor);
          }

          // Read size and verify non-negative.
          int size = s.readInt();
          if (size < 0) {
              throw new InvalidObjectException("Illegal size: " +
                                               size);
          }

          // Set the capacity according to the size and load factor ensuring that
          // the HashMap is at least 25% full but clamping to maximum capacity.
          capacity = (int) Math.min(size * Math.min(1 / loadFactor, 4.0f),
                                    HashMap.MAXIMUM_CAPACITY);

          // Create backing HashMap
          map = (((HashSet<?>)this) instanceof LinkedHashSet ?
                 new LinkedHashMap<E,Object>(capacity, loadFactor) :
                 new HashMap<E,Object>(capacity, loadFactor));

          // Read in all elements in the proper order.
          for (int i=0; i<size; i++) {
              @SuppressWarnings("unchecked")
              E e = (E) s.readObject();
              map.put(e, PRESENT);
          }
      }

      //为了并行遍历数据源中的元素而设计的迭代器
      public Spliterator<E> spliterator() {
          return new HashMap.KeySpliterator<E,Object>(map, 0, -1, 0, 0);
      }

  }
  ```

## LinkedHashSet

- 特点
  - LinkedHashSet: 底层的数据结构是链表和哈希表
  - 元素唯一 ,并且有序(根据插入的顺序)，哈希表保证元素的唯一性。链表保证元素有素。(存储和取出是一致)
  - LinkedHashSet 的内部实现都是来自于 HashMap 、HashSet 和 LinkedHashMap 容器类，其内部源码十分简单，简单到它只有一个成员变量、四个构造函数、一个 Set 接口的方法。
- 示例

  ```java
  LinkedHashSet<String> hs = new LinkedHashSet<String>();

  // 创建并添加元素
  hs.add("hello");
  hs.add("world");
  hs.add("java");
  hs.add("world");
  hs.add("java");

  // 遍历
  for (String s : hs) {
  	System.out.println(s);
  }
  结果:
  	hello
  	world
  	java
  ```

## LinkedHashSet 源码分析

- LinkedHashSet 的所有源码如下所示

  ```java
  public class LinkedHashSet<E>
      extends HashSet<E>
      implements Set<E>, Cloneable, java.io.Serializable {

      //序列化ID
      private static final long serialVersionUID = -2851667679971038690L;

      //自定义初始容量与装载因子
      public LinkedHashSet(int initialCapacity, float loadFactor) {
          super(initialCapacity, loadFactor, true);
      }

      //自定义初始容量
      public LinkedHashSet(int initialCapacity) {
          super(initialCapacity, .75f, true);
      }

      //使用默认的初始容量以及装载因子
      public LinkedHashSet() {
          super(16, .75f, true);
      }

      //使用初始数据、默认的初始容量以及装载因子
      public LinkedHashSet(Collection<? extends E> c) {
          super(Math.max(2*c.size(), 11), .75f, true);
          addAll(c);
      }

      //并行遍历迭代器
      @Override
      public Spliterator<E> spliterator() {
          return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
      }

  }
  ```

- LinkedHashSet 继承于 HashSet，而 LinkedHashSet 调用的父类构造函数均是

  ```java
  private transient HashMap<E,Object> map;

  HashSet(int initialCapacity, float loadFactor, boolean dummy) {
      map = new LinkedHashMap<>(initialCapacity, loadFactor);
  }
  ```

- 即 LinkedHashSet 底层是依靠 LinkedHashMap 来实现数据存取的，而 LinkedHashMap 继承于 HashMap，在内部自己维护了一条双向链表用于保存元素的插入顺序。

- 因此使得 LinkedHashSet 也具有了存取有序，元素唯一的特点。

## TreeSet

- **TreeSet 集合特点**

  - TreeSet 存储对象的时候, 可以排序, 但是需要指定排序的算法。
  - 底层数据结构是**红黑树**(自平衡二叉树，也就是 TreeMap, 放入数据不能重复且不能为 null
  - 保证元素唯一性，依赖于 compareTo 方法的返回值是否为 0.

- 排序

  - Integer 能排序(有默认顺序), String 能排序(有默认顺序), 自定义的类存储的时候出现异常(没有顺序)。
  - 自然排序(元素具备比较性)那么必须实现 Comparable 接口
  - 比较器排序:让集合的构造方法接收一个比较器的子类对象 Comparator
  - 可以重写 compareTo()方法来确定元素大小，从而进行升序排序。
  - **二叉树不需要重写 hashCode()和 equals()**
    - 定义比较器是实现 Comparator 接口,重写 compare 方法和 equals 方法,但是由于所有的类默认继承 Object,而 Object 中有 equals 方法,所以自定义比较器类时,不用重写 equals 方法,只需要重写 compare 方法

  ```java
  compareTo()方法中:
  	return 0: 不添加到集合
  	return正数: 作为右孩子
  	return 负数: 作为左孩子
  通过中序遍历可以得到有序
  ```

- 去重复: 产生 10 个 1-20 之间的随机数要求随机数不能重复案例

  ```java
  /**
   * 产生10个1-20之间的随机数,要求不能重复
   * 分析:
   *         1: 创建一个HashSet集合对象 , 作用: 存储产生的随机数
   *         2: 生成随机数 , 把随机数添加到集合中
   *         3: 使用循环,当集合的长度大于等于10退出循环 , 小于10就一直循环
   */
  // 创建一个HashSet集合对象 , 作用: 存储产生的随机数
  HashSet<Integer> hs = new HashSet<Integer>() ;
  while(hs.size() < 10) {
      // 使用Random类
      Random random = new Random() ;
      int num = random.nextInt(20) + 1 ;
      // 把num添加到集合中
      hs.add(num) ;
  }
  // 遍历
  for(Integer i : hs) {
      System.out.println(i);
  }
  ```

- **如何进行排序**

  - 集合可以对元素进行排序,而排序有两种方式一种是自然排序,一种是比较强器排序
  - 到底使用的是自然排序还是比较器排序,主要取决于我们使用的构造方法

  ```java
  public TreeSet():                                     使用自然排序
  public TreeSet(Comparator<? super E> comparator)      使用的就是比较器排序
  ```

## TreeSet 排序

### 自然排序

- **使用 TreeSet 集合存储自定义对象,并排序案例**

  - 基本数据类型都实现了自然排序接口(Comparable)
  - `Student implements Comparable<Student>`
  - `TreeSet ts = new TreeSet(); //自然排序`

  ```java
  /**
   * 需求: 使用TreeSet集合存储自定义对象,并排序
   */
  // 创建自定义对象
  Student s1 = new Student("刘亦菲" , 18) ;
  Student s2 = new Student("貂蝉" , 16) ;
  Student s3 = new Student("杨玉环" , 22) ;
  Student s4 = new Student("王昭君" , 14) ;
  Student s5 = new Student("西施" , 25) ;
  Student s6 = new Student("范冰冰" , 25) ;
  // 创建TreeSet集合对象
  TreeSet<Student> ts = new TreeSet<Student>() ;        // 自然排序
  // 添加元素
  ts.add(s1) ;
  ts.add(s2) ;
  ts.add(s3) ;
  ts.add(s4) ;
  ts.add(s5) ;
  ts.add(s6) ;
  // 遍历集合
  for(Student s : ts) {
      System.out.println(s.getName() + "---" + s.getAge());
  }

  //在student类中
  public class Student implements Comparable<Student>{
      //省略部分代码
      @Override
      public int compareTo(Student o) {
          /**
           * 按照年龄进行排序
           */
          int num = this.age - o.age  ;
          // 比较姓名
          int num2 = num == 0 ? this.name.compareTo(o.name) : num ;
          return num2;
      }
  }
  ```

- 使用 TreeSet 集合存储自定义对象,安排姓名的长度进行排序

  ```java
  /**
   * 需求: 使用TreeSet集合存储自定义对象,安排姓名的长度进行排序
   */
  // 创建自定义对象
  Student s1 = new Student("liubei" , 23) ;
  Student s2 = new Student("guanyunchang" , 21) ;
  Student s3 = new Student("zhangfei" , 18) ;
  Student s4 = new Student("lvbu" , 25) ;
  Student s5 = new Student("zhugeliang" , 23) ;
  Student s6 = new Student("guanyu" , 23) ;
  Student s7 = new Student("guanyu" , 25) ;

  // 创建集合对象
  TreeSet<Student> ts = new TreeSet<Student>() ;

  // 添加元素
  ts.add(s1) ;
  ts.add(s2) ;
  ts.add(s3) ;
  ts.add(s4) ;
  ts.add(s5) ;
  ts.add(s6) ;
  ts.add(s7) ;

  // 增强for遍历
  for(Student s : ts) {
      System.out.println(s.getName() + "---" + s.getAge());
  }

  //-------------------------
  public class Student implements Comparable<Student>{
      @Override
      public int compareTo(Student o) {
          /**
           * 比较姓名的长度
           */
          int num = this.name.length() - o.name.length() ;
          // 比较内容
          int num2 = (num == 0) ? this.name.compareTo(o.name) : num ;
          // 比较年龄
          int num3 = (num2 == 0) ? this.age - o.age : num2 ;
          return num3;
      }
  }
  ```

### 比较器排序

- `public TreeSet(Comparator comparator(接口)) //比较器排序`

- 示例：`TreeSet<Student> ts = new TreeSet<Student>(new MyComparator());`

- 通过匿名内部类

  ```java
  TreeSet<Student> ts = new TreeSet<Student>(new Comparator<Student>(){
      @Override
      public int compare(Student s1, Student s2){
          // 姓名长度
          int num = s1.getName().length() - s2.getName().length();
          // 姓名内容
          int num2 = num == 0 ? s1.getName().compareTo(s2.getName())
              : num;
          // 年龄
          int num3 = num2 == 0 ? s1.getAge() - s2.getAge() : num2;
          return num3;
      }
  });
  ```

- 自己写一个实现了 Comparator 接口的类

  ```java
  public class Mycomparator implements Comparator<Student>{
      @override
      public int compare(Student s1,Stuedent s2){
          int num = s1.getAge()-s2.getAge();
          // 年龄大小
          int num2 = num==0?s1.getName().compareTo(s2.getName()):num;
          // 年龄相同,比较姓名
          return num2;
      }
  }
  ```

## SortedSet 接口

> 是 Set 的子接口, 实现类 TreeSet; 底层数据结构是 二叉树

1. first() 获得第一个元素
2. last() 获得最后一个元素
3. subSet(start, end) 得到子集,\[起始元素,终止元素)

```
public static void main(String[] args) {
	SortedSet<Integer> set = new TreeSet<>();
	set.add(111);
	set.add(25);
	set.add(153);

	System.out.println(set);// [25, 111, 153]
	// 获得第一个元素
	System.out.println(set.first());// 25

	// 获得最后一个元素
	System.out.println(set.last());// 153

	// 子集,[起始元素,终止元素)
	System.out.println(set.subSet(25, 111));// [25]
}
```

### 数:表示层次结构的

> 树：是由节点集及连接每对节点的有向边集组成。
> 二叉树：树形结构任意节点不能超过两个孩子。

1. 根节点
2. 枝节点
3. 叶节点
4. 中序遍历: 左中右
5. 要管理树(维持持续),效率不如 hashSet

## NavigableSet 接口

> 是 sortedSet 的子类接口
> 提供了接近匹配原则的检索元素的方法

1. floor(e):小于 等于 指定参数 的最大元素
2. ceiling:大于等于 指定参数的 最小元素
3. descendingSet():集合元素降序排序
4. descendingIterator():降序的迭代器
5. pollFirst():移除第一个元素
6. pollLast(): 移除最后一个元素

```java
public static void main(String[] args) {
	// NavigableSet
	NavigableSet<Double> set = new TreeSet<>();
	set.add(11.1);
	set.add(55.5);
	set.add(22.2);
	set.add(99.9);
	System.out.println(set);	//[11.1, 22.2, 55.5, 99.9]

	//小于 等于 指定参数 的最大元素
	System.out.println(set.floor(20.0));	//11.1

	//大于等于 指定参宿的 最小元素
	System.out.println(set.ceiling(20.0));	//22.2

	//降序的集合
	set.descendingSet().forEach(System.out::println);

	//降序的迭代器
	set.descendingIterator().forEachRemaining(System.out::println);

	结果降序:
	/*
		99.9
		55.5
		22.2
		11.1
	*/

	//移除第一个元素
	set.pollFirst();
	System.out.println(set);	//[22.2, 55.5, 99.9]

	//移除最后一个元素
	set.pollLast();
	System.out.println(set);	//[22.2, 55.5]
}
```
