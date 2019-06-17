# PHP设计模式（七）适配器模式（Adapter For PHP）

适配器模式：将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以在一起工作。

![](https://rawcdn.githack.com/WilburXu/blog/12a7e13c35897e3a3b8a68362876bdd0e1b2abfc/design-pattern/images/adapter/1.png)



#### 先来看一个案例：

#####  设置书的接口

```php
// 书接口
interface BookInterface
{
    // 翻页方法
    public function turnPage();

    // 打开书方法
    public function open();
}
```

```php
// 纸质书实现类
class Book implements BookInterface
{
    public function turnPage()
    {
        echo "纸质书翻页". "<br>";
    }

    public function open()
    {
        echo "纸质书打开". "<br>";
    }
}
```

##### 客户端测试：

```php
// 客户端测试
$book = new Book();
$book->open();
$book->turnPage();
```

##### 输出结果:

```
纸质书打开
纸质书翻页
```



#### 这时候，你想创建一个可以复用的类，该类可以与其他不相关的类或不可预见的类（即那些接口可能不一定兼容的类）协同工作。如下：

```php
// 待适配对象
class Kindle
{
    public function turnPage()
    {
        echo "电子书翻页". "<br>";
    }

    public function open()
    {
        echo "电子书打开". "<br>";
    }
}
```

```php
class KindleAdapter implements BookInterface
{
    protected $_kindle;

    public function __construct($obj)
    {
        $this->_kindle = $obj;
    }


    public function turnPage()
    {
        $this->_kindle->turnPage();
    }

    public function open()
    {
        $this->_kindle->open();
    }
}
```

##### 客户端测试：

```php
$kindle = new KindleAdapter(new Kindle());
$kindle->open();
$kindle->turnPage();
```

##### 输出结果：

```
电子书打开
电子书翻页
```

