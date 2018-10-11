# PHP设计模式（二）工厂方法模式（Factory Method）



## **简单工厂简述：**

简单工厂模式实现了产品类的代码跟客户端代码分离，但会有一个问题，优秀的代码是符合“开闭原则”如果你要加一个C类产品，你就要修改工厂类里面的代码，也就是说要增加条件语句如：switch---case。对于这个问题，接下来的工厂方法模式可以解决这个问题。



## 一、 什么是工厂方法模式

工厂方法就是为配一个产品提供一个独立的工厂类，通过不同的工厂实例来创建不同的产品实例。



## **二、 工厂方法模式的优点**

1. 拥有良好的封装性，代码结构清晰。对于每一个对象的创建都是有条件约束的。如：调用一个具体的产品对象，只需要知道这个产品的类名和约束参数就可以了，不用知道创建对象自身的复杂过程。降低模块之间的耦合度。
2. 拥有良好的扩展性，新增一个产品类，只需要适当的增加工厂类或者扩展一个工厂类，如下面的例子中，当需要增加一个数据库Oracle的操作，则只需要增加一个Oracle类，工厂类不用修改任务就可完成系统扩展。
3. 屏蔽产品类。这一特点非常重要，产品类的实现如何变化，调用者都不需要关心，它只需要关心产品的接口，只要接口保持不变，系统中的上层模块就不要发生变化。



## 三、使用场景

1. 支付宝、微信、银联的连接方式(connectMode)，支付方式(payMode)。   使用工厂模式，“客户”就不需要要知道具体的连接方式和支付方式了， 只需要调用connectMode 和 payMode即可。 

2. MySQL、SQL Server、Oracle等数据库的连接方式（connectMode）、查询方式（selectMode）等操作可以使用工厂模式进行封装。

#### 接下来看具体的案例：

#####产品类：

```php
	//抽象产品类
    abstract class DataBase
    {
        abstract function connect();
        abstract function getOne();
    }
    
　　//具体产品类
    class MySql extends DataBase
    {
        function connect()
        {
            return "MySQL连接对象返回";
        }
    
        function getOne()
        {
            return "MySQL返回查询结果";
        }
    }
    
　　//具体产品类
    class SqlServer extends DataBase
    {
        function connect()
        {
            return "SQL Server连接对象返回";
        }
    
        function getOne()
        {
            return "SQL Server返回查询结果";
        }
    }
```

#####工厂类：

```php
//抽象工厂类
    abstract class FactoryDataBase{
        function createDataBase(){}
    }
    
　　//具体工厂类
    class FactoryMySql extends FactoryDataBase
    {
        public function createDataBase()
        {
            return new MySql();
        }
    }
    
　　//具体工厂类
    class FactorySqlServer extends FactoryDataBase
    {
        public function createDataBase()
        {
            return new SqlServer();
        }
    }
```

#####客户：

```php
 $mysql = new FactoryMySql();
 $db1 = $mysql->createDataBase();
```



##四、工厂方法模式的组成

1. 抽象工厂角色：这是工厂方法模式的核心，它与应用程序无关。是具体工厂角色必须实现的接口或者必须继承的父类。
2. 具体工厂角色：它含有和具体业务逻辑有关的代码。由应用程序调用以创建对应的具体产品的对象。
3. 抽象产品角色：它是具体产品继承的父类或者是实现的接口。
4. 具体产品角色：具体工厂角色所创建的对象就是此角色的实例。



工厂方法模式仿佛已经把对象的创建进行了很完美的包装，使得客户程序中仅仅处理抽象产品角色提供的接口。那我们是否一定要在代码中遍布工厂呢？大可不必。也许在下面情况下你可以考虑使用工厂方法模式： 

1. **当客户程序不需要知道要使用对象的创建过程。**      
2. **客户程序使用的对象存在变动的可能，或者根本就不知道使用哪一个具体的对象。**