# JUnit单元测试-1

## 概念
一个典型的单元测试通常可以描述为：“确保方法接受预期范围内的输入，并且为每一次测试输入返回预期的值”。  

单元测试检查一个独立的工作单元的行为。在Java应用程序中，独立工作单元经常是（但不总是）一个独立方法。一个工作单元就是一项任务，不直接依赖与其他任何任务的完成。  

## API契约
单元测试通常关注的是一个方法是否遵循了它的API契约中的条款。  
API契约是一份由方法签名而生成的正式协议，方法需要它的调用者提供特定的对象引用或原始类型数值，然后返回一个对象引用或原始类型数值。如果方法不能遵守契约，测试将抛出异常，于是我们就说方法已经破坏了它的契约。  

API契约是一种观点：把一个应用程序编程接口（API）当作是在调用者和被调用者之间的正式协议。通常，单元测试通过证明预期的行为来帮助定义API契约。  

单元测试的核心是：任何没有经过自动测试的程序功能都可以当作不存在。  

## 单元测试框架遵循的规则
所有单元测试框架应该遵循的三大规则：  
- 1、每个单元测试都必须独立于其他所有单元测试而运行；  
- 2、框架应该以单个测试为单位来检测和报告错误；  
- 3、应该易于定义要运行哪个单元测试。  

## JUnit
`JUnit`是一个单元测试框架，它使得创建、运行和修改单元测试更加简单。  
`JUnit`团队已经为框架定义了3个不相关的目标：  
- 1、框架必须帮助我们编写有用的测试；  
- 2、框架必须帮助我们创建具有长久价值的测试；  
- 3、框架必须帮助我们通过复用代码来降低编写测试的成本。  

定义一个测试类的要求：这个类必须是公共的并且包含一个无参构造函数。  
定义一个测试方法的要求：这个方法必须使用`@Test`注释，是公共的，不带任何参数，并且返回void类型。  

`JUnit`在调用（执行）每个`@Test`方法之前，为测试类创建一个新的实例。   
这样做的好处：  
- 有助于提供测试方法之间的独立性  
- 避免在测试代码中产生意外的副作用    
因为每个测试方法都运行于一个新的测试类实例上，所以我们就不能在测试方法之间重用各个实例变量值。  

当需要一次运行多个测试类时，就要创建另一个叫做测试集的对象。测试集也是一个特定的测试运行器，因此可以像运行测试类那样运行它。  

**<font color='blue'>测试类</font>**：一个包含一个或多个测试的类，而这些测试就是指那些用`@Test`注释的方法。  

**<font color='blue'>测试集</font>**：（Suite或test suite）一组测试，测试集是一种把多个相关测试归入一组的便捷方式。比如，如果没有为测试类定义一个测试集，那么JUnit会自动提供一个测试集，包含测试类中所有的测试。一个测试集通常会将同一个包中的测试类归入一个组。  

**<font color='blue'>测试运行器</font>**：（Runner或test runner）执行测试集的程序。  

## JUnit核心对象职责

|JUnit概念|责任|
|:---:|:---|
|Assert|让你去定义你想测试的条件，当条件成立时，assert方法保持沉默；但条件不成立时则抛出异常|
|测试|一个以@Test注释的方法定义了一个测试。<br/>为了运行这个方法，JUnit会创建一个包含类的新实例，然后再调用这个被注释的方法|
|测试类|一个测试类是@Test方法的容器|
|Suite|Suite允许你将测试类归为一组|
|Runner|Runner类用来运行测试|

要运行一个基础的测试类，你不需要做什么特别的工作；JUnit会代替你使用一个测试运行器来管理你的测试类的生命周期，包括创建类、调用测试以及搜集结果。  

## 运行参数化测试
`Parameterized`（参数化）的测试运行器运行你使用不同的参数多次运行同一个测试。
使用`Parameterized`的测试运行器来运行一个测试类，需要满足以下要求：
- 1、测试类必须使用`@RunWith`注释，并且要将`Parameterized`类作为它的参数
- 2、必须声明测试中所使用的实例变量
- 3、提供一个用`@Parameters`注释的方法，该方法的签名必须是`public static java.util.Collection`，无任何参数，`Collection`元素必须是相同长度的数组，这个数组的长度必须要和这个唯一的公共构造函数的参数数量相匹配。
- 4、指定有参构造
- 5、定义一个测试`@Test`方法，该方法实例化程序并调用所提供的参数

代码示例如下：  
一个类包含一个add方法用于单元测试：  
```java
public class Calculator{
    public double add(double number1, double number2) {
        return number1 + number2;
    }
}
```
单元测试代码如下：  
```java
@RunWith(value = Parameterized.class)
public class ParamterizedTest {
    private double expected;
    private double value1;
    private double value2;

    @Parameters
    public static Collection<Integer[]> getTestParameters(){
        return Arrays.asList(new Integer[][]{
                {2, 1, 1},
                {3, 2, 1},
                {4, 3, 1}
        });
    }

    public ParamterizedTest(double expected, double value1, double value2){
        this.expected = expected;
        this.value1 = value1;
        this.value2 = value2;
    }

    @Test
    public void addTest(){
        Calculator cal = new Calculator();
        Assert.assertEquals(expected, cal.add(value1, value2), 0);
    }
}
```
分析运行过程：  
`JUnit`调用了静态方法`getTestParameters`，`JUnit`为`getTestParameters`集合中的每个数组进行循环。  
然后`JUnit`调用了唯一的公共构造函数，如果存在多个构造，`JUnit`就回抛出一个断言错误。  
`JUnit`使用由数组元素构成的一系列参数来调用构造函数。然后`JUnit`分别调用`@Test`方法。  

可以发现，参数化的`JUnit`测试运行器运行了同一个方法3次，为`@Parameters`集合中的每个值运行一次；而上述示例只运行了一个测试。  

`Parameterized`类是`JUnit`众多测试运行器中的一个，还有其他常用的运行器。

## JUnit测试运行器
JUnit4的测试运行器

|运行器|目 的|
|:--|:--|
|org.junit.internal.runners.JUnit38ClassRunner|这个运行器包含当前的JUnit版本中，仅仅是为了向后兼容。它将测试用例作为JUnit3.8的测试用例来启动|
|org.junit.runners.JUnit4|这个运行器将测试用例作为JUnit4的测试用例来启动|
|org.junit.runners.Parameterized|这个测试运行器可以使用不同参数来运行相同的测试集|
|org.junit.runners.Suite|Suite是一个包含不同测试的容器，同时Suite也是一个运行器，可以运行一个测试类中的所有以@Test注释的方法|

如果测试类中没有提供任何运行器，那么JUnit将会使用默认运行器。如果要指定某个特定的测试运行器，那么就用`@RunWith`注释来指定测试运行器。  

## JUnitCore facade
为了能够尽可能快捷的运行测试，JUnit提供了一个facade（org.junit.runner.JUnitCore），它可以运行任何测试运行器。  

facade是一种设计模式，它为子系统中的一组接口提供了一个统一的接口，facade定义了一个更高级别的接口，使得子系统更易于使用。你可以使用facade来将一些复杂的对象交互简化成一个单独的接口。  

## 使用Suite组合测试
组合一组测试类，就是运行多个测试类，为了简化这个任务，`JUnit`提供了测试`Suite`。这个`Suite`是一个容器，用来把几个测试归在一起，并把它们作为一个集合一起运行。  

`JUnit`设计`Suite`的目的就是为了运行一个或者多个测试用例。测试运行器会启动`Suite`；然后运行哪个测试用例是由`Suite`来决定的。  
为了使事情可以保持简单，如果没有定义一个`Suite`，那么测试运行器会自动创建一个`Suite`。
默认的`Suite`会扫描你的测试类，找出所有以`@Test`注释的方法。默认的`Suite`会在内部为每个`@Test`方法创建一个测试类的实例。然后`JUnit`就会独立的执行每个`@Test`方法，以避免潜在的负面影响。  
`Suite`对象其实是一个`Runner`，可以执行测试类中所有的`@Test`注释的方法。  
下面的代码显示了如何将多个测试类组合成一个单独的测试集（test suite）  
```java
@RunWith(value=org.junit.runners.Suite.class)
@SuiteClasses(value={XXTest1.class, XXTest2.class})
public class MyJUnitTest {
}
```
上述代码使用`@RunWith`注释指定了相应的运行器，并且通过在`@SuiteClasses`注释找那个指定测试类，来列出我们想要在这个测试中包含的所有测试。这些测试类中的所有`@Test`方法都将包含到该`Suite`中。

### 组合一组测试集
代码示例如下：
```java
public class TestCaseA{
  @Test
  public void teatA1(){

  }
}
```
```java
public class TestCaseB{
  @Test
  public void teatB1(){

  }
}
```
```java
@RunWith(value=Suite.class)
@SuiteClasses{value={TestCaseA.class}}
public class TestSuiteA{

}
```
```java
@RunWith(value=Suite.class)
@SuiteClasses{value={TestCaseB.class}}
public class TestSuiteB{

}
```
```java
@RunWith(value=Suite.class)
@SuiteClasses{value={TestCaseA.class, TestCaseB.class}}
public class MasterTestSuite{

}
```
为了简化这个示例，我们的测试集TestSuiteA 与TestSuiteB 分别只有一个测试用例。而真正的测试集应该包含多个测试类，就
像我们的主测试集。  
可以运行这段代码中的任何一个类， 比如一个只JUnit测试、两个测试类中的任意一个、两个测试集中的任意一个，甚至是主测试集。  
