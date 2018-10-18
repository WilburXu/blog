# PHP设计模式（八）桥接模式（Bridge For PHP）

## 概述

　　桥接模式：将两个原本不相关的类结合在一起，然后利用两个类中的方法和属性，输出一份新的结果。　



## 适用性

1. 类的抽象以及它的实现都应该可以通过生成子类的方法加以扩充。通过使用Bridge模式对不同的抽象接口和实现部分进行组合，并分别对它们进行扩充。
2. 不希望在抽象和它的实现部分之间有一个固定的绑定关系。
3. 一个构件有多于一个的抽象化角色和实现化角色，系统需要它们之间进行动态耦合。



## 效果

1. Bridge模式使用“对象间的组合关系”解耦了抽象和实现之间固有的绑定关系，使得抽象和实现可以沿着各自的维度来变化。
2. 所谓抽象和实现沿着各自维度的变化，即“子类化”它们，得到各个子类之后，便可以任意它们，从而获得不同员工组和不同信息发送模式。
3. Bridge模式的应用一般在“两个非常强的变化维度”，有时候即使有两个变化的维度，但是某个方向的变化维度并不剧烈——换言之两个变化不会导致纵横交错的结果，并不一定要使用Bridge模式。



## 案例

1. 模拟毛笔（[转](http://blog.csdn.net/hguisu)）

　　需求：现在需要准备三种粗细（大中小），并且有五种颜色的比

　　如果使用蜡笔，我们需要准备3*5=15支蜡笔，也就是说必须准备15个具体的蜡笔类。而如果使用毛笔的话，只需要3种型号的毛笔，外加5个颜料盒，用3+5=8个类就可以实现15支蜡笔的功能。

　　实际上，蜡笔和毛笔的关键一个区别就在于笔和颜色是否能够分离。即将抽象化(Abstraction)与实现化(Implementation)脱耦，使得二者可以独立地变化"。关键就在于能否脱耦。蜡笔由于无法将笔与颜色分离，造成笔与颜色两个自由度无法单独变化，使得只有创建15种对象才能完成任务。而毛笔与颜料能够很好的脱耦（比和颜色是分开的），抽象层面的概念是："毛笔用颜料作画"，每个参与者（毛笔与颜料）都可以在自己的自由度上随意转换。

　　Bridge模式将继承关系转换为组合关系，从而降低了系统间的耦合，减少了代码编写量。

2. 模拟企业分组发短信

　　需求：公司现在需要按分组（临时工、正式工、管理层等）以多种形式（QQ、Email、微博等）给员工发送通知。



### 员工分组：

```php
abstract class Staff
{
    abstract public function staffData();
}

class CommonStaff extends Staff
{
    public function staffData()
    {
        return "小名，小红，小黑";
    }
}

class VipStaff extends Staff
{
    public function staffData()
    {
        return '小星、小龙';
    }
}
```

### 发送形式

```php
// 抽象父类
abstract class SendType
{
    abstract public function send($to, $content);
}

class QQSend extends SendType
{
    public function __construct()
    {
        // 与QQ接口连接方式
    }

    public function send($to, $content)
    {
        return $content. '（To '. $to . ' From QQ）<br>';
    }
}
```

### 桥接类

```php
class SendInfo
{
    protected $_level;
    protected $_method;

    public function __construct($level, $method)
    {
        //  这里可以使用单例控制资源的消耗
        $this->_level = $level;
        $this->_method = $method;
    }

    public function sending($content)
    {
        $staffArr = $this->_level->staffData();
        $result = $this->_method->send($staffArr, $content);
        echo $result;
    }
}
```

### 客户端调用：

```php
$info = new SendInfo(new VipStaff(), new QQSend());
$info->sending( '回家吃饭');

$info = new SendInfo(new CommonStaff(), new QQSend());
$info->sending( '继续上班');
```

### 输出结果：

```php
回家吃饭（To 小星、小龙 From QQ）
继续上班（To 小名，小红，小黑 From QQ）
```



## 小结

从上面可以看出，如果增加分组或者是发送信息的类型，都可以直接创建一个类，来拓展，十分方便。

但是Bridge模式虽然是一个非常有用的模式，也非常复杂，它很好的符合了开放-封闭原则和优先使用对象，而不是继承这两个面向对象原则。