---
title: 向贫血模型宣战
toc: true
date: 2020-12-14 18:00:45
---

## 什么是贫血模型

回想一下我们定义的经典代码

```java
public class UserPo {
    private String id;
    private String name;    
    private String age;
    //getter、setter
}
```

这个`UserPo`类没有任何行为，只是数据容器，只是为了适应`Hibernate`、`Mybatis`这些ORM框架而存在。
使用这种模型带来的后果是什么呢？大量的业务逻辑，校验规则都被放到了service层。

举个简单的例子，修改年龄，最常见的做法是：

```java
public class UserService{
  
    private UserDao userDao;
  
    public void changeAge(Long id, Integer newAge) {
        if (newAge < 0) {
            throw new IllegalArgumentException();
        }
        UserPo user = userDao.findById(id);
        user.setAge(newAge);
    }
}
```

哪里有不妥帖的地方？我们把所有的业务逻辑放在了`UserService`这个类中，比如校验年龄是否符合规则。

## 什么是充血模型

让我们换一种做法：

```java
public class UserPo {
    private String id;
    private String name;    
    private String age;
  
    public void changeAge(Integer newAge) {
        if (newAge < 0) {
            throw new IllegalArgumentException();
        }
      this.age = newAge;
    }
  
    public void rename(String newName) {
      if (newName == null) {
        throw new IllegalArgumentException();
      }
      this.name = newName;
    }
}
```

看出变化了么：

1. 舍弃了常见的getter、setter方法，转而使用更面向业务且更具表现力的命名方式；
2. 将参数的校验放到了持久化对象中，使得`UserPo`也拥有了业务逻辑。

## 贫血模型有什么问题

我们用一个小例子解释了什么是[贫血模型](https://en.wikipedia.org/wiki/Anemic_domain_model)，什么是充血模型，但说起来就算用贫血模型，又有什么问题呢？我们有必要舍弃习以为常的模型去追求什么充血模型么？

> **简单的业务系统采用这种贫血模型和过程化设计是没有问题的，**但在业务逻辑复杂了，业务逻辑、状态会散落到在大量方法中，原本的代码意图会渐渐不明确，我们将这种情况称为由贫血症引起的失忆症。
>
> 来自[领域驱动设计在互联网业务开发中的实践](https://tech.meituan.com/2017/12/22/ddd-in-practice.html)

贫血模型跟分层的思想紧密相连，在这种风格中模型只是用来承载数据，业务逻辑由其它类（比如service）来承载。

这种做法带来的一个问题是：我们以为对象就是数据的容器而已，从而失去了面向对象思想的真正关注点。

另外一个实际的问题是，对象的处理会散落在项目各处。还是上面的例子，如果使用充血模型，那么校验年龄就是`UserPo`对象的行为，只要使用`changeAge()`方法，就会去校验年龄的有效性。我们不必考虑其他service层设置`changeAge()`方法时还要校验年龄[^1]。

## 重新理解面向对象

回想一下Java编程语言第一课，必然会介绍说：Java是一门面向对象的编程语言，并且有：封装、继承、多态三个特征。

理想状态下我们的对象应该是这个样子的：


```java
public class Dog {
  public String state;
  
  public void walk() {
    this.state = "walking";
  }
  
  public void lieDown() {
    this.state = "lying";
  }
  
  public String getState() {
    return this.state;
  }
}
```

这时候`Dog`是一个更加富含行为，更**“面向对象”**的对象，我们让它`walk()`它就能`walking`，让它`lieDown()`它马上就`lying`了。

但往往下面的代码更为常见的：

```java
public class Dog {
  public String state;
  public void setState(String aState) {
    this.state = aState;
  }
  public String getState() {
    return this.state;
  }
}
```

```java
public class StateService {
  
  public void changeState(String state) {
    dog.setState(state);
  }
  
}
```

上面的场景里把业务逻辑都放入service层理解起来都没有什么问题，那为什么还要说**向贫血模型宣战**呢？

因为我们面对的场景从来都不会简单。

面向对象本质上是为了让人更加舒服。以往计算机性能捉襟见肘，程序员们绞尽脑汁想的都是怎么压榨计算机性能，所以本质上那时候程序语言都是服务于计算机的（比如说C，C++）。后来计算机性能上来了有了些富余加上软件的规模越来越大，于是大家开始考虑怎么写出让人更容易理解的代码，这才是面向对象的初心：以人为本。


## 总结

虽然本文的题目叫做**向贫血模型宣战**，但我也认为做好抽象设计很难，因此一些编程的方法论还是得读得看。可能有人觉得简单的CRUD项目不需要搞的那么复杂，但好的编程范式尽量遵守，这样在未来面对大型复杂的需求时才能游刃有余。

[^1]: 实际上这不是一个好例子，至少我不认为修改年龄这个方法会散乱在各个service类中。
