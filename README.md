写一个玩具Java虚拟机
FEB 25TH, 2017 12:00 AM
本文描述了一个用Java实现的玩具JVM，用Java实现的好处是可以不用处理JVM中的垃圾回收。

Java虚拟机是基于栈的虚拟机。栈虚拟机的特点是所有临时操作数都存放在栈中。编译器生成的指令都会围绕着这个栈展开，相对而言，解释执行这些指令会比较容易。基于栈的虚拟机可能会生成如下指令：

1
2
3
push 3   # 把立即数3压栈
push 4   # 把立即数4压栈
add      # 从栈中弹出两个操作数进行相加，结果压回栈中
Java .class文件存储的主要就是编译后的指令，一个玩具JVM，简单来说就是解释执行这里面的指令。接下来就说说为了让这个JVM跑起来需要实现哪些功能。

class 文件解析
推荐一下 Java class viewer，里面有个工具可以可视化class文件内容。另外我直接复用了他解析class文件的代码。

class文件描述的信息是以class为单位的，一个类如果有嵌套类，这个嵌套类也会生成为单独的class文件。从c/c++程序员的视角来看，class文件的生成有点类似编译，编译器在编译期间只做依赖符号存在与否的检查。所有引用其他class的地方，不同于c/c++，java class的引用都是在运行期定位的。这里看看一个简单的类class文件结构是怎样的：

1
2
3
4
5
6
7
8
9
package test;

public class Simple {
  private int data;

  public int add(int a, int b) {
    return a + b;
  }
}


一个class文件比较重要的有：

constant pool(常量池)：存储字符串字面量、函数原型描述、类成员描述、class引用描述。字节码中经常会引用常量池中的内容，例如要设置某个成员变量，字节码中的操作数就是常量池索引，从索引中获取出具体是哪个成员变量
fields：描述类成员变量
methods: 描述类成员函数
attributes: 分布在很多地方，可能嵌套，用于描述method字节码、调试符号信息等。
常量池非常重要，这里看看class文件中是如何使用常量池的。例如，一个field描述：



其中name_index和descriptor_index的值，指向的就是常量池索引，通过前面推荐的class viewer去常量池中找就会找到对应的值：



descriptor_index描述field类型，I指的是整数。Java里有一套描述类型的规则，这个规则在函数定义的地方也会看到。

methods只要有实现，就都会有一个Code attribute，也就是这个函数的具体实现字节码，例如前面的add函数字节码为：

1
2
3
4
opcode [1B] - 0000: iload_1     # 将第1个局部变量值压栈
opcode [1C] - 0001: iload_2     # 将第2个局部变量值压栈
opcode [60] - 0002: iadd        # 弹出2个操作数相加，结果压栈
opcode [AC] - 0003: ireturn     # 弹出1个操作数作为函数返回值
解析出class文件中的信息后，玩具JVM就完成一半了。

JVM指令
JVM中已经有200多条指令了。但是这些指令很多都是相似的。具体实现这些指令时可以参考指令表。JVM指令就像x86指令一样，由操作码以及可选的操作数组成。操作码表示具体是哪条指令，占1个字节；操作数表示该操作码需要的参数，变长。class文件中字节码连续存放，像上一节的例子就是4条指令，每条指令只有操作码没有操作数，他们存放在class文件中就是：1B 1C 60 AC。

JVM依次读取这些指令并解释执行。这个过程同真实计算机CPU执行过程类似。用代码描述为：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
while (true) {
  code = fetchOpCode()
  if (code == iload_1) {
    push(1)
  } elif (code == iadd) {
    i1 = pop()
    i2 = pop()
    push(i1 + i2)
  } elif (code == bipush) { // 需要操作数的指令
    b = fetchOpValue()
    push(b)
  }
  ...
}
程序执行过程中有一个虚拟指针PC，用于指示当前字节码处理到哪个位置了。有些跳转指令会强制改变PC。不同于c/c++程序，JVM中跳转指令跳转的都是相对位移。JVM启动时，不同于c/c++，也没有地址重定位的过程(修正相对地址为实际地址)。

执行环境
线程执行环境
JVM中每个线程都是独立的执行单元。但是对于类符号等信息则是全局共享，堆上创建的对象也是全局可访问的。单个线程中调用函数会产生帧(frame)，每一帧都有一个独立的栈用于存储该帧执行的临时数据。从main函数开始执行，每进入一个函数创建一个帧，函数退出(执行return系列指令)清除当前帧。这里的帧也可以被实现为一个栈，当这个栈里没有帧时就表示这个线程退出。这个过程可以描述为：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
while (thread.topFrame() != null) {
  thread.topFrame().run() // 内部实现就是从Code属性处不断地取指令执行
}

// 执行到return语句时，就弹出该帧
if (code == op_return) {
  frame.getThread().popFrame()
}

// 遇到函数调用时，就根据目标函数创建出一帧
if (code == op_invoke) {
  method = findMethod(ref) // 函数调用时操作数是常量池索引，需要加载目标类，获取目标方法
  newFrame = createFrame(method) // 每个方法都会描述该方法内有多少局部变量，所需栈多大，根据这些信息初始化帧
  frame.getThread().pushFrame(newFrame) 
}
注释中提到了，class文件中method包含了“有多少局部变”、“需要多大的栈”等信息。局部变量的实现是一个数组，数组的下标表示局部变量是第几个。字节码中要访问局部变量时，全部是以这个下标来查找的。例如指令istore_1，表示从栈中弹出一个整数，并写到局部变量1中。



全局环境
由于使用Java实现，堆内存的管理完全不用操心。如果我们代码中创建了一个类对象，或者简单点调用了另一个类的静态方法，这个时候会发生什么以及如何处理？例如以下代码：

1
int a = Simple2.inc(2);
生成以下字节码：

1
2
opcode [05] - 0004: iconst_2  # 压入常量2到栈，作为函数调用的参数
opcode [B8] - 0005: invokestatic 2 [Methodref: test.Simple2.inc, parameter = (int), returns = int]
可以看出调用静态函数invokestatic的操作数是2，指向的是常量池中的2。工具直接显示了常量池2是一个method，及该method的原型。

遇到这样的指令时，我们就需要找到并加载目标类。所以，全局信息里需要维护类列表。考虑到Java中类与类之间是否相同，除了看类名（全限定名）外，还得看类的加载器(class loader)。所以，玩具JVM中也需要有class loader机制（至少是个简化版）。程序启动时设定一个默认的类加载器，加载主类，执行主类main方法，执行过程中遇到对其他类的引用时，就使用当前类加载器继续加载目标类，如果已经加载就直接返回。

类加载及main方法
前面已经提到了类加载。其实类加载本质上就是把目标class文件加载到内存，保存该class信息。在调用一个类的方法时，也是根据方法名(考虑到方法重载，还得考虑方法的原型，在class file中也就是descriptor)找到具体的方法，根据方法初始化调用帧，以及根据方法获得其要执行的字节码。

所以，我们的JVM要跑起来，也就是找到并加载主类，然后找到主类中的main函数并执行。

1
2
3
4
5
6
  public void run(String mainClass) {
    Class clazz = mRootLoader.loadClass(mainClass);
    MethodInfo method = clazz.findMethod("main", "([Ljava/lang/String;)V");
    Thread thread = new Thread();
    thread.run(clazz, method);
  }
当然，这个过程严格来说还得判定类访问控制、方法访问控制等。

至此，这个玩具JVM就可以跑起来了。可以设定它的class path，加载类，从main方法开始执行，调用其他类的静态方法，写写阶乘的实现是没有问题了。但是Java中还有很多其他特性：类对象、调用类实例方法、异常处理、调用native方法等待。接下来我再讲讲这些特性的实现，一窥Java核心语法的实现。

扩展实现
类对象及实例方法调用
类对象的创建通过new指令完成，本质上也就是分配个对象，并关联类信息到这个对象。我们的实现中自然会有一个类用来表示玩具JVM中所有的对象：

1
2
3
4
5
6
7
8
// 为了与java.lang.Object区分开
public class VObject {
  // 简单起见，直接以field名作为key，来保存该对象所有的成员变量
  private Map<String, Slot> mFields; 
  // 与之关联的类信息
  private final Class mClazz;
    ...
}
注意这里的Class是我们自己定义的Class，而不是java.lang.Class。Slot类型用于存储整数或一个引用(其他对象)。new指令的大概实现：

1
2
3
4
5
6
7
8
9
10
register(Opcode.op_new, (codes, frame) -> {
  int b1 = codes.readByte();
  int b2 = codes.readByte();
  int idx = (b1 << 8) + b2; // 常量池索引，指向一个类信息描述
  String clazzName = frame.getClazz().resolveClassName(idx); // 根据这个类信息描述解析到具体的类名
  Class clazz = frame.getClazz().getClassLoader().loadClass(clazzName); // 使用当前的类加载器加载该类
  initClass(frame.getThread(), clazz); 
  VObject object = new VObject(clazz); // 创建该类对应的对象，完成new指令的对象创建动作
  frame.pushRef(object); // 根据new指令的语义，我们需要将创建好的对象引用压回栈中
});
需要注意的是，当我们在Java中写下代码 new SomeClass() 时，实际会产生两个功能的指令：a) 创建对象；b) 调用类的构造函数()

1
2
3
opcode [BB] - 0004: name_new 2 [Class: test.Simple2]  # new 指令，操作数是类信息常量池索引
opcode [59] - 0007: dup 
opcode [B7] - 0008: invokespecial 3 [Methodref: test.Simple2. , parameter = (), returns = void] # 调用目标类的构造函数，也就是<init>方法
调用类构造函数同普通类实例方法原理相同，都会先压入对象引用。invokespecial指令用于调用类对象实例方法，从栈顶依次出栈参数，最后出栈类对象实例引用。具体可以看看指令表里的描述。

类静态区域初始化
首次加载某个类时，会执行其static区域代码。这个写测试看下生成的代码就懂了，就是生成一个的静态方法，在加载类时先执行这个方法。

异常处理
当一个方法中有try/catch时，该方法就会生成出一个异常处理表，存储在Code属性中。如下图：



异常处理表每一项都包含：start_pc、end_pc、handler_pc 及catch_type，表示在start_pc/end_pc间发现异常，且异常类型是catch_type时，则跳转到handler_pc处执行代码，也就是异常处理代码。其中catch_type也是常量池中的索引，当其为0时，则不是常量池索引，而是表示catch所有类型，其实就是finally块。从这里也可以看出，常量池索引是从1开始，而不是0。

当异常发生时，JVM首先从当前帧对应的方法中的异常处理表查找异常处理代码，没有的话则弹出当前帧，回到上一帧，也就是调用者继续查找，直到找完所有调用帧。这个实现相对较多，就不列举代码了。

调用本地方法
前面实现的JVM都没有输出字符串的能力，要提供一个类似System.out.println的方法，就需要注册本地方法到JVM中。这里可以简单地为整个JVM设置一个本地方法表，在JVM启动时完成注册。类似以下代码：

1
public static void native println(String s);
也会在class文件中留下一个method，但这个method会被标记为native，自然也没有Code属性，没有字节码可执行。当执行invoke系列指令时，发现调用的是native方法，就需要从全局本地方法表中查找。注册本地方法类似：

1
2
3
4
mMethods.put(makeKey("java/lang/System", "println", "(Ljava/lang/String;)V"), (frame) -> {
  String s = (String) frame.popRef();
  System.out.println(s);
});
本地方法执行时，通过frame参数就可以取出调用该方法传入的参数。

在实现了本地方法后，就可以给这个玩具JVM添加一些系统库，类似OpenJDK中jre目录下的lib。这些系统库可以包含java.lang.System.println、java.lang.String、java.lang.StringBuilder。简单起见，我实现的这些类和Java标准库有些不同。

最后
完整代码这里。实现的指令很有限，可以跑通test/AllTest.java中的代码。对于long/double等类型没有支持，各种限定访问的判断也没有。总之就是简单到仅仅可以一窥原理。基于java8，测试例子class文件java7编译(应该没关系)。祝玩得开心。
