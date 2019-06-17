# PHP设计模式（四）单例模式（Singleton）



## 一、什么是单例设计模式

单例模式，也叫单子模式，是一种常用的软件设计模式。在应用这个模式时，单例对象的类必须保证只有一个实例存在。



## **二、单例模式的技巧**

1. 利用$_instance私有变量来保存类的唯一实例化对象；
2. 设计一个getInstance对外公开的函数，可以获取类唯一实例；
3. 防止用户用new实例化，和克隆，构造两个__construct、__clone私有函数；



## **三、单例模式的应用场景**

　　数据库设计，我们发送一次请求，可能会需要访问不同的表，那么如果每次访问都 new 一个实例，那必然会造成资源的浪费，所以使用单例模式，可以很好的节省资源。

##### 单例类

```php
class DataBase
{
    /**
     * 静态成品变量，保存全局实例
     */
    private static $_instance = null;

    /**
     *  测试变量，存储日志信息
     */
    private static $_msg = null;

    /**
     * 私有构造方法，防止外界实例化对象
     */
    private function __construct()
    {
        $connect = "连接数据库操作";
    }

    /**
     * 私有化克隆方法，防止外键克隆对象
     */
    private function __clone()
    {
    }

    /**
     * 静态方法，外界获取实例的唯一接口
     * @return Object 返回对象唯一实例
     */
    public static function getInstance()
    {
        if (!self::$_instance){
            self::$_instance = new DataBase();
            self::$_msg = "这是一个新对象" . "<br>";
        }else{
            self::$_msg = "这个是一个旧的对象" . "<br>";
        }

        return self::$_instance;
    }

    public function log()
    {
        echo self::$_msg;
    }
}
```

##### 客户端测试代码

```php
    $dbA = DataBase::getInstance();
    $dbA->log();

    $dbB = DataBase::getInstance();
    $dbB->log();

    $dbC = DataBase::getInstance();
    $dbC->log();
```

##### 输出结果：

这是一个新对象

这个是一个旧的对象

这个是一个旧的对象

 

 **“对象”?，程序员怎么可能有对象！~**



## 参考

### 系列源地址

[WilburXu/design_pattern](WilburXu/design_pattern)

### 系列目录

1. [PHP 设计模式概述](https://segmentfault.com/a/1190000016629282)

2. [PHP设计模式（一）简单工厂模式 （Simple Factory For PHP）](https://segmentfault.com/a/1190000016635395)

3. [PHP设计模式（二）工厂方法模式（Factory Method）](https://segmentfault.com/a/1190000016646401)

4. [PHP设计模式（三）抽象工厂模式（Abstract Factory）](https://segmentfault.com/a/1190000016659904)

5. [PHP设计模式（四）单例模式（Singleton）](https://segmentfault.com/a/1190000016670292)
