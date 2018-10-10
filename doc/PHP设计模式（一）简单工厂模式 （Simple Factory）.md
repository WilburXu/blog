# PHP设计模式（一）简单工厂模式 （Simple Factory For PHP）



## 一、什么是简单工厂模式

简单工厂 （Simple Factory）又称静态工厂方法模式（Static Factory Method Pattern）

　　使用的频率可以说是非常之高，它的官方解释为：定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂模式使一个类的实例化延迟到其子类。

　　这个模式本身很简单而且使用在业务较简单的情况下。一般用于小项目或者具体产品扩展教师较少的情况（这样工厂类才不用经常更改）。

　　**PS：不修改代码的话，是无法扩展的。**

## 二、简单工厂模式的作用
简单工厂的作用是实例化对象，而不需要客户了解这个对象属于哪个具体的子类。**简单工厂实例化的类具有相同的接口或者基类，在子类比较固定并不需要扩展时，可以使用简单工厂，一定程度上可以很好的降低耦合度。**

## 三、案例
  1. 支付宝、微信、银联的连接方式(connectMode)，支付方式(payMode)。使用工厂模式，“客户”就不需要不要知道具体的连接方式和支付方式了，只需要调用connectMode 和payMode即可。 
  2. MySQL、SQL Server、Oracle等数据库的连接方式（connectMode）、查询方式（selectMode）等操作可以使用工厂模式进行封装。下面的例子会讲到。

我们以数据库类创建的案例来说：
#### 产品类
```
/** 
 * 数据库系列 
 * 
 */  
abstract Class DataBase
{  
    abstract function getOne($sql); //获取一条数据的方法
}  

Class SqlServer extends DataBase
{  
    function __construct() { 
        $connect = "SqlServer 连接方法操作 （腾讯云服务器）";
        return $connect
    }

　　function getOne($sql){
        return "查询后返回数据结果";
    }
}  

Class MySql extends DataBase
{  
   function __construct(){  
       $connect = "MySql 连接方法操作 （阿里云服务器）";
       return $connect
   }

    function getOne($sql){
        return "查询后返回数据结果";
    }
}
```

#### 工厂类
```
/** 
 *  
 * 创建数据库的工厂类 
 */  
class Factory {  
      static function  createDataBase($type) {  
        switch ($type) {  
          case SqlServer:  
             return new SqlServer();  
          case MySql:  
             return new MySql();  
        //....  
   }  

}  
```

#### 客户类

```
/** 
 *  
 * 客户通过工厂获取数据 
 */  
class Customer {  
    private $database;  
    
    function getDataBase($type) {  
        return $this->database =  Factory::createDataBase($type);  
    } 
}

$custome = new Customer;
$db = $custome->getDataBase("SqlServer"); // 我要获取阿里云的SQL Server数据库的数据。
$data = $db->getOne($sql);
```

## 四、组成部分
通过以上案例可以得知一般情况下工厂模式由以下几个部分组成：
  1. 工厂类角色：这是本模式的核心，含有一定的商业逻辑和判断逻辑，根据逻辑不同，产生具体的工厂产品。如例子中的Factory类。
  2. 抽象产品角色：它一般是具体产品继承的父类或者实现的接口。由接口或者抽象类来实现。如例中的DataBase接口。
  3. 具体产品角色：工厂类所创建的对象就是此角色的实例。在JAVA中由一个具体类实现，如例子中的MySql和SqlServer类。

使用工厂设计模式时必须先归类你的产品（需求）找到共同点和特征，然后根据共同的地方创建各自的产品类，这时候如果没有无法通过客户类去调用每一个产品类，那么耦合度会大大增高（在需求变动的时候）， 这时候创建一个工厂类统一管理产品类，再通过客户类调用。 那么可以很好的管理代码并一定程度上的解耦。 