# PHP设计模式（三）抽象工厂模式（Abstract Factory）

## 一、什么是抽象工厂模式

抽象工厂模式的用意为：给客户端提供一个接口，可以创建多个产品族中的产品对象 ，而且使用抽象工厂模式还要满足以下条件：

1. 系统中有多个产品族，而系统一次只可能消费其中一族产品。 
2. 同属于同一个产品族的产品可以使用。 

**产品族：位于不同产品等级结构中，功能相关联的产品组成的家族。下面例子的 汽车和空调就是两个产品树， 奔驰C200+格力某型号空调就是一个产品族， 同理， 奥迪A4+海尔某型号空调也是一个产品族。**

## 二、类图

![1](/images/abstract_factory/1.png)

## 三、案例

##### 产品类

```php
// 汽车（抽象产品接口）
interface AutoProduct
{
    public function dirve();
}


//奥迪A4（具体产品类）
class AudiA4Product implements AutoProduct
{
    //获取汽车名称
    public function dirve()
    {
        echo "开奥迪A4"."<br>";
    }
}

//奔驰C200（具体产品类）
class BenzC200Product implements AutoProduct
{
    //获取汽车名称
    public function dirve()
    {
        echo "开奔驰C200"."<br>";
    }
}


```

```php
//空调（抽象产品接口）
interface AirCondition
{
    public function blow();
}

//格力空调某型号（具体产品类）
class GreeAirCondition implements AirCondition
{
    public function blow()
    {
        echo "吹格力空调某型号"."<br>";
    }
}

//海尔空调某型号（具体产品类）
class HaierAirCondition implements AirCondition
{
    public function blow()
    {
        echo "吹海尔空调某型号"."<br>";
    }
}
```



##### 工厂类

```php
//工厂接口
interface Factory
{
    public function getAuto();
    public function getAirCondition();
}


//工厂A = 奥迪A4 + 海尔空调某型号
class AFactory implements Factory
{
    //汽车
    public function getAuto()
    {
        return new AudiA4Product();
    }

    //空调
    public function getAirCondition()
    {
        return new HaierAirCondition();
    }
}
```

```php
//工厂B = 奔驰C200 + 格力空调某型号
class BFactory implements Factory
{
    //汽车
    public function getAuto()
    {
        return new BenzC200Product();
    }

    //空调
    public function getAirCondition()
    {
        return new GreeAirCondition();
    }
}
```



##### 客户端类

```php
//客户端测试代码
$factoryA = new AFactory();
$factoryB = new BFactory();

//A工厂制作车
$auto_carA = $factoryA->getAuto();
$auto_airA = $factoryA->getAirCondition();

//B工厂制作车
$auto_carB = $factoryB->getAuto();
$auto_airB = $factoryB->getAirCondition();

//开奥迪车+吹海尔空调
$auto_carA->dirve();
$auto_airA->blow(); //热的时候可以吹吹空调

//开奔驰车+吹格力空调;
$auto_carB->dirve();
$auto_airB->blow(); //热的时候可以吹吹空调
```



## 四、抽象工厂模式的组成

1. 抽象工厂（AbstractFactory）：确定工厂的业务范围。
2. 具体工厂（ConcreteFactory）：每个具体工厂对应一个产品族。具体工厂决定生产哪个具体产品对象。
3. 抽象产品（AbstractProduct）：同一产品等级结构的抽象类。
4. 具体产品（ConcreteProduct）：可供生产的具体产品。

### 工厂方法模式：

- 一个抽象产品类，可以派生出多个具体产品类。
- 一个抽象工厂类，可以派生出多个具体工厂类。
- 每个具体工厂类只能创建一个具体产品类的实例。

### 抽象工厂模式：

- 多个抽象产品类，每个抽象产品类可以派生出多个具体产品类。
- 一个抽象工厂类，可以派生出多个具体工厂类。
- 每个具体工厂类可以创建多个具体产品类的实例。

 ### 三种工厂的比较
- 简单工厂 ：用来生产同一等级结构中的任意产品。（对于增加新的产品，无能为力）

- 工厂方法 ：用来生产同一等级结构中的固定产品。（支持增加任意产品）   

- 抽象工厂 ：用来生产不同产品族的全部产品。（对于增加新的产品，无能为力；支持增加产品族）

