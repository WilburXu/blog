# PHP 设计模式概述
## 一、设计模式（Design pattern）是什么
设计模式是一套被反复使用、多数人知晓、经过分类编目的代码设计的经验总结。使用设计模式是为了可重用代码、让代码更容易被他人理解、保证代码可靠性。

## 二、为什么会有设计模式
在软件开发过程中，一个功能的实现方式多种多样，不同方法的可扩展性、可维护性以及复用性都是不一样的。随着一个人对自己项目代码的要求增加，他会逐渐思考和实践出自己的一套方法或者思想，这种方法或思想决定了他设计出的架构或者编写出的代码的质量优劣。设计模式就属于这样一种经验的积累，是由大量优秀的工程师或者架构师总结和提炼的精华，**学习好设计模式等于让我们站在了巨人的肩膀上，从一个高的起点出发，可以避免走很多弯路。**

## 三、设计模式的分类
一般情况下，我们把设计模式分成了三大类：
### 创建型（Creational patterns）
创建型模式是为了解决创建对象时候遇到的问题。因为基本的对象创建方式可能会导致设计上的问题，或增加设计的复杂度，创建型设计模式有两个主导思想：**一是将系统使用的具体类封装起来，二是隐藏这些具体类的实例创建和结合方式。**
#### 创建型模式主要有以下五种：

1. [简单工厂模式（Simple Factory）](https://github.com/WilburXu/design_pattern/blob/master/doc/PHP%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%88%E4%B8%80%EF%BC%89%E7%AE%80%E5%8D%95%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F%20%EF%BC%88Simple%20Factory%EF%BC%89.md)
2. [工厂方法模式（Factory method）](https://github.com/WilburXu/design_pattern/blob/master/doc/PHP%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%88%E4%BA%8C%EF%BC%89%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95%E6%A8%A1%E5%BC%8F%EF%BC%88Factory%20Method%EF%BC%89.md)
3. [抽象工厂模式（Abstract factory）](https://github.com/WilburXu/design_pattern/blob/master/doc/PHP%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%88%E4%B8%89%EF%BC%89%E6%8A%BD%E8%B1%A1%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F%EF%BC%88Abstract%20Factory%EF%BC%89.md)
4. [单例模式（Singleton）](https://github.com/WilburXu/design_pattern/blob/master/doc/PHP%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%88%E5%9B%9B%EF%BC%89%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F%EF%BC%88Singleton%EF%BC%89.md)
5. [建造者模式（Builder）](https://github.com/WilburXu/design_pattern/blob/master/doc/PHP%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F%EF%BC%88%E4%BA%94%EF%BC%89%E5%BB%BA%E9%80%A0%E8%80%85%E6%A8%A1%E5%BC%8F%EF%BC%88Builder%EF%BC%89.md)
6. 原型模式（Prototype） 

GOF在《设计模式》一书中将工厂模式分为两类：工厂方法模式（Factory Method）与抽象工厂模式（Abstract Factory）。将简单工厂模式（Simple Factory）看为工厂方法模式的一种特例，两者归为一类。

### 结构型模式（Structural pattern）
结构型模式是通过定义一个简单方法来实现和了解实体间关系，从而简化设计。
  1. 适配器模式（Adapter）
  2. 桥接模式（Bridge）
  3. 合成模式（Composite）
  4. 装饰者模式（Decorator）
  5. 表象模式（Facade）
  6. 享元模式（Flyweight）
  7. 代理模式（Proxy） 

### 行为型模式（Behavioral pattern）
行为型模式是用来识别对象之间的常用交流模式并加以实现，使得交流变得更加灵活。
  1. 策略模式（Strategy）
  2. 模板方法模式（Template method）
  3. 观察者模式（Observer）
  4. 迭代器模式（Iterator）
  5. 责任链模式（Chain of responsibility）
  6. 命令模式（Command）
  7. 备忘录模式（Memento）
  8. 状态模式（State）
  9. 访问者模式（Visitor）
  10. 中介者模式（Mediator）
  11. 解释器模式（Interpreter）

## 四、 各个设计模式之间的关系 (这图可以对设计模式有一定了解后，再回头看会比较清晰)
![图片描述][1]


[1]: /doc/images/summary/1.png