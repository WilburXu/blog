# PHP设计模式（五）建造者模式（Builder）

## 什么是建造者设计模式

**建造者模式：**将一个复杂对象的构造与它的表示分离，使同样的构建过程可以创建不同的表示的设计模式。

**设计场景：**

##### 有一个用户的UserInfo类，创建这个类，需要创建用户的姓名，年龄，爱好等信息，才能获得用户具体的信息结果。如：

##### 这是一个用户类：

```php
class UserInfo
{
    protected $_userName;
    protected $_userAge;
    protected $_userHobby;

    public function setUserName($userName)
    {
        $this->_userName = $userName;
    }

    public function setUserAge($userAge)
    {
        $this->_userAge = $userAge;
    }

    public function setUserHobby($userHobby)
    {
        $this->_userHobby = $userHobby;
    }

    public function getPeopleInfo()
    {
        echo  "<br>这个人的名字是：" . $this->_userName . "<br>年龄为：" . $this->_userAge . "<br>爱好：" . $this->_userHobby;
    }
}
```

##### 这时候我们要获取一个用户的信息，过程是这样的：

```php
$modelUser = new UserInfo();
$modelUser->setUserName('松涛');
$modelUser->setUserAge('23');
$modelUser->setUserHobby('推理小说');
$modelUser->getPeopleInfo();
```

##### 得到的结果是：

```php
这个人的名字是：松涛
年龄为：23
爱好：推理小说
```



------



这时候我们来看建造者设计模式的设计：

##### 创建一个UserBuilder 用户建造者类，这个类，将UserInfo复杂的创建姓名，年龄，爱好等操作封装起来，简化用户类的创建过程：

这个是将复杂的创建过程封装在了**builderPeople**这个方法里面。 接下来是创建对象:

```php
class UserBuilder
{
    protected $_obj;

    public function __construct()
    {
        $this->_obj = new UserInfo();
    }

    public function builderPeople($userInfo)
    {
        $this->_obj->setUserName($userInfo['userName']);
        $this->_obj->setUserAge($userInfo['userAge']);
        $this->_obj->setUserHobby($userInfo['userHobby']);
    }

    public function getBuliderPeopleInfo()
    {
        $this->_obj->getPeopleInfo();
    }
}
```

##### 客户端获取数据：

```php
$userArr = array(
    'userName' => '松涛',
    'userAge' => '23',
    'userHobby' => '推理小说');

$modelUserBuilder = new UserBuilder();
$modelUserBuilder->builderPeople($userArr);
$modelUserBuilder->getBuliderPeopleInfo();
```

##### 输出结果为：

```php
这个人的名字是：松涛
年龄为：23
爱好：推理小说
```



## 建造者的优缺点

### **优点：**

建造者模式可以很好的将一个对象的实现与相关的“业务”逻辑分离开来，从而可以在不改变事件逻辑的前提下,使增加(或改变)实现变得非常容易。

### **缺点：**

建造者接口的修改会导致所有执行类的修改。



### **以下情况应当使用建造者模式：**

1、 需要生成的产品对象有复杂的内部结构。
2、 需要生成的产品对象的属性相互依赖，建造者模式可以强迫生成顺序。
3、 在对象创建过程中会使用到系统中的一些其它对象，这些对象在产品对象的创建过程中不易得到。

### **根据以上例子，我们可以得到建造者模式的效果：**
1、 建造者模式的使用使得产品的内部表象可以独立的变化。使用建造者模式可以使客户端不必知道产品内部组成的细节。
2、 每一个Builder都相对独立，而与其它的Builder（独立控制逻辑）无关。
3、 模式所建造的最终产品更易于控制。



### **建造者模式与工厂模式的区别：**

我们可以看到，建造者模式与工厂模式是极为相似的，总体上，建造者模式仅仅只比工厂模式多了一个“导演类”的角色。在建造者模式的类图中，假如把这个导演类看做是最终调用的客户端，那么图中剩余的部分就可以看作是一个简单的工厂模式了。

与工厂模式相比，建造者模式一般用来创建更为复杂的对象，因为对象的创建过程更为复杂，因此将对象的创建过程独立出来组成一个新的类——导演类。也就是说，工厂模式是将对象的全部创建过程封装在工厂类中，由工厂类向客户端提供最终的产品；而建造者模式中，建造者类一般只提供产品类中各个组件的建造，而将具体建造过程交付给导演类。由导演类负责将各个组件按照特定的规则组建为产品，然后将组建好的产品交付给客户端。



