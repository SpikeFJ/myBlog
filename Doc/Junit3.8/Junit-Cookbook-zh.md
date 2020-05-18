# JUnit Cookbook

Kent Beck, Erich Gamma

***

本篇文章展示了读者如何利用`Junit`按照步骤编写和组织自己的单元测试。

## 简单的单元测试

如何编写测试代码？

最简单的方式就是在调试器中执行表达式。你无需编译就可以调整表达式，并且你可以等待最终结果出来之后再做决定如何编码。你也可以编写一行测试语句将结果打印到标准输出上。但是这两种方式都是受限的，因为它们都需要**人**来评判和分析执行结果。而且，它们两者组合并不友好-你只能在某个时刻执行一个调试语句，大量的打印语句更是令人烦恼。

`Junit`测试不需要人来评判和理解，并且很容易就可以在同一时刻运行大量测试。当你想测试一些东西时，下面就是你需要做的：

1. 创建一个`TestCase`的子类
2. 重写`runTest()`方法
3. 当你想测试一个值时，调用`assertTrue()`并且传递一个`boolean`,如果返回为真则表明测试通过

举例来说：如果想测试两种相同的货币相加是否等于指定值时候，可以编写如下代码：

```java
public void testSimpleAdd() {
    Money m12CHF= new Money(12, "CHF"); 
    Money m14CHF= new Money(14, "CHF"); 
    Money expected= new Money(26, "CHF"); 
    Money result= m12CHF.add(m14CHF); 
    assertTrue(expected.equals(result));
}
```

如果你编写一个和以前类似的单元测试，那么就写一个`Fixture`；如果你写运行多个单元测试，那么就创建一个`Suite`。

## Fixture（组合）

如果有多个测试作用于一组相同或相似的对象时该如何是好呢？

测试运行需要和一组已知的对象一起协作。这些对象称之为`Fixture`。当你编写单元测试时，你常常发现，组织`Fixture`比实际测试花费更多的时间。

在某种程度上，你可以小心的组织要编写的内容以便更加容易的编写`fixture`，但是共享`fixture`可以带来更大的节省。通常你可以针对不同的单元测试编写一些相同的`fixture`，每一个case都会向`fixture`发送有些微差异的消息或参数并检查不同的结果。

当你遇到相同的`fixture`时，你可以：

1. 创建一个`TestCase`的子类
2. 针对`fixture`的每一部分创建一个实例变量
3. 重写`setup()`来初始化变量
4. 重写`tearDown()`来释放在`setup`中分配的永久资源

举例：如果想编写一些测试用例来测试`12法郎`，`14瑞士法郎`和`28美元`不同的组合，首先创建一个`fixture`:

```java
public class MoneyTest extends TestCase { 
    private Money f12CHF; 
    private Money f14CHF; 
    private Money f28USD; 
    
    protected void setUp() { 
        f12CHF= new Money(12, "CHF"); 
        f14CHF= new Money(14, "CHF"); 
        f28USD= new Money(28, "USD"); 
    }
}
```

一旦准备好了`Fixture`,那么就可以写尽可能多的单元测试了。

## Test Case（测试用例）

当你拥有`Fixture`时，如何编写并调用一个独立的 测试用例呢？

当没有`fixture`时编写一个单元测试就很容易的-在`TestCase`的匿名子类中重写`runTest`即可。针对`Fixture`编写单元测试也一样，为准备好的代码创建`TestCase`子类，然后为每个独立的测试用例生成匿名子类。然后，在经历了一系列测试之后你会发现很大一部分代码都是在处理语法。

`Junit`提供了一种更简洁的方式来针对`Fixture`测试：

1. 在`fixture`的class类中编写`public void `方法，按照惯例，方法名称以`test`开头。

如，要测试钱包和钱的相加，编写如下：

```java
public void testMoneyMoneyBag() { 
    // [12 CHF] + [14 CHF] + [28 USD] == {[26 CHF][28 USD]} 
    Money bag[]= { f26CHF, f28USD }; 
    MoneyBag expected= new MoneyBag(bag); 
    assertEquals(expected, f12CHF.add(f28USD.add(f14CHF)));
}
```

创建一个`MoneyTest`实例并且会如下执行测试用例：

```java
new MoneyTest("testMoneyMoneyBag")
```

当测试运行时，测试的名称可以用来检索哪些方法正在运行。

一旦你拥有多个测试用例，将它们组织成`Suite`

## Suite（套件）

如何立即执行多个测试用例呢？

一旦你拥有两个测试，你就会希望它们一起执行。你可以一次运行一个，不过你将会很快厌烦。`Junit`提供了一种`TestSuite`对象,可以将任意数量的测试用例一起执行

例如，你想执行单个测试用例，你可以这样：

```java
TestResult result= (new MoneyTest("testMoneyMoneyBag")).run();
```

如果想执行多个则创建一个`suite`包含两个测试用例并一起执行：

```java
TestSuite suite= new TestSuite();
suite.addTest(new MoneyTest("testMoneyEquals"));
suite.addTest(new MoneyTest("testSimpleAdd"));
TestResult result= suite.run();
```

另一种方法就是让`Junit`从测试用例中提取`suite`，为此你需要将测试用例的`class`传递给`TestSuite`构造函数：

```java
TestSuite suite= new TestSuite(MoneyTest.class);
TestResult result= suite.run();
```

当你希望一个`suite`值包含测试用例的一个subset时，请采用手动方式；否则自动提取`suite`是首选。它避免了z在添加新的测试用例时需要更新`suite`创建代码。

`TestSuite`并不是只能包含`TestCase`，它可以包含任何实现了`Test`接口的对象。如你可以创建一个`TestSuite`，我也可以创建我自己的`TestSuite`，然后一起执行：

```java
TestSuite suite= new TestSuite();
suite.addTest(Kent.suite());
suite.addTest(Erich.suite());
TestResult result= suite.run();
```

## TestRunner（测试执行器）

如何运行你的测试用例并且收集执行结果？

一旦你拥有一个`test suite`,就需要执行。`Junit`提供了一些工具来定义要运行的`suite`和显示执行结果。你可以通过静态方法返回一个`test suite`使其可以被工具访问。

如想让一个`MoneyTest`对`TestRunner`可见，添加如下代码至`MoneyTest`中：

```java
public static Test suite() { 
    TestSuite suite= new TestSuite(); 
    suite.addTest(new MoneyTest("testMoneyEquals")); 
    suite.addTest(new MoneyTest("testSimpleAdd")); 
    return suite;
}
```

或者简单一些：

```java
public static Test suite() { 
    return new TestSuite(MoneyTest.class); 
}
```

如果`TestCase`类没有定义一个类似的方法，那么`TestRunner`将会自己提取一个`suite`方法并将所有以`test`开头的方法填充进去。

`Junit`同时提供了图形和文本的`TestRuner`工具。可以通过`java` `junit.awtui.TestRunner`或者`junit.swingui.TestRunner`启动它们，GUI代表了一个窗体包含以下：

1. 

