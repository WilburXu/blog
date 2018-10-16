# PHP设计模式（六）原型模式（Prototype For PHP）

原型设计模式： 用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象。

原型设计模式简单的来说，就是不去创建新的对象进而保留原型的一种设计模式。



## 案例

##### 原型类：

```php
interface Prototype {
    public function copy();
}
```

```php
class PrototypeDemo implements Prototype
{
    private $_name;

    public function __construct($name)
    {
        // 这里可能是复杂的逻辑
        $this->_name = $name;
    }

    public function getMul()
    {
        return $this->_name * $this->_name;
    }

    public function copy()
    {
        // 克隆后的逻辑
        $this->_name ++;
        return clone $this;
    }
}
```

##### 客户类：

```php
class Client
{
    public function main()
    {
        $pro1 = new PrototypeDemo('10');
        echo $pro1->getMul();

        echo "<br>";

        $pro2 = $pro1->copy();
        echo $pro2->getMul();
    }
}
```

```php
$obj = new Client();
$obj->main();
```

输出结果：

```
100
121
```