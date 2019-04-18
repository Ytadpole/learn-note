## mybatis sql执行分析1

### 核心

1. sqlSession执行crud操作，都是交给Executor去完成。
2. Executor的主要需要创建jdbc的statement，然后执行
3. 创建statement交给StatementHandler完成。
4. 如果需要参数设置，还需要用的类型转换TypeHandler
5. statementHandler执行sql，封装结果

### 实际分析

以SqlSessionTest的单测为例

![image-20190418111527009](images/mybatis-sql1/image-20190418111527009.png)

sqlSession.selectOne进入到

![image-20190418111743715](images/mybatis-sql1/image-20190418111743715.png)

查询处理交给Executor处理

Executor实现有

![image-20190418111901301](images/mybatis-sql1/image-20190418111901301.png)

![image-20190418112153423](images/mybatis-sql1/image-20190418112153423.png)

分析：

执行Executor的直接实现有两个

1. BaseExecutor

2. CachingExecutor

BaseExecutor

CachingExecutor的作用相对于BaseExecutor做一个代理者，提供缓存处理，自身缓存处理完，后续操作再交由BaseExecutor的具体实现进行完成。

![CachingExecutor](images/mybatis-sql1/image-20190418112806194.png)

本身做一个静态代理，最后交由BaseExecutor的继承类SimpleExecutor完成查询操作

对于BaseExecutor使用到了模版设计模式

参考：<http://www.cnblogs.com/qq-361807535/p/6854191.html>



首先进入SimpleExecutor父类BaseExecutor的query方法中，**BaseExecutor的query方法定义了查询操作的执行算法，包含了一些缓存操作，具体的查询操作是在具体子类的doQuery中实现。对应的子类有四个，见上面的uml图**

![image-20190418135309351](images/mybatis-sql1/image-20190418135309351.png)

先不考虑缓存问题， 程序最终进入的BaseExecutor的queryFromDatabase()方法，根据方法命名即可知道是去数据库查询。

![image-20190418135553478](images/mybatis-sql1/image-20190418135553478.png)

这里开始调用具体子类的 查询实现逻辑doQuery，这时进入SimpleExecutor的具体doQuery实现。

![image-20190418140402128](images/mybatis-sql1/image-20190418140402128.png)

开始创建StatementHandler

![image-20190418140804863](images/mybatis-sql1/image-20190418140804863.png)

StatementHandler对应于JDBC的statement，

RoutingStatementHandler作为一个代理者，为PreparedSatementHandler,CallableStatementHandler和SimpleStatementHanlder做代理。

![image-20190418141721741](images/mybatis-sql1/image-20190418141721741.png)

RoutingStatementHandler的构造方法，进行选择代理对象，这里选择了prepared类型的。

接下来构建statement

![image-20190418141949528](images/mybatis-sql1/image-20190418141949528.png)

交给handler进行构建

![image-20190418142211503](/Users/yangsong/learn-note/images/mybatis-sql1/image-20190418142211503.png)

RoutingStatementHandler交给PreparedStatementHandler处理

同样和执行起Executor，使用了模版模式，进入BaseStatementHandler的prepare方法

![image-20190418142523500](images/mybatis-sql1/image-20190418142523500.png)

进入instantiateStatement（）构建statement，具体的instantiateStatement由子类实现，这里是prepareStatementHandler，所以进入![image-20190418142823894](images/mybatis-sql1/image-20190418142823894.png)通过jdbc基本操作构建PreparedStatement对象

设置参数

![image-20190418143138866](images/mybatis-sql1/image-20190418143138866.png)

![image-20190418143108530](images/mybatis-sql1/image-20190418143108530.png)

依然通过RoutingStatementHandler代理到PreparedStatementHandler

![image-20190418143253077](images/mybatis-sql1/image-20190418143253077.png)

PreparedStatementHandler交给ParameterHandler处理，使用默认实现DefaultParameterHandler处理。

![image-20190418143702505](images/mybatis-sql1/image-20190418143702505.png)

开始处理参数，

![image-20190418144357131](images/mybatis-sql1/image-20190418144357131.png)

jdbc的赋值操作 是交给TypeHandler处理的

![image-20190418144553211](images/mybatis-sql1/image-20190418144553211.png)

依然是模版模式，具体的赋操作进入

![image-20190418144700013](images/mybatis-sql1/image-20190418144700013.png)

这样赋完参数值。statement就构建完毕了。



代码再往下走

![image-20190418144939627](images/mybatis-sql1/image-20190418144939627.png)

构建完jdbc的statement就很容易进行相关的查询操作。交给handler处理，同样通过RoutingStatementHandler代理到PreparedStatementHandler，

![image-20190418145134871](images/mybatis-sql1/image-20190418145134871.png)

进入到PreparedStatementHandler，操作非常简单，最后处理结果集。

![image-20190418145243857](images/mybatis-sql1/image-20190418145243857.png)



最后封装结果集。完毕

### 总结

学习到模版模式，mybatis几乎每个组件都用到了模版模式。里面还有很多细节没仔细分析，缓存，BoundSql，对应到Mapper解析的处理等等。sql执行过程过了一遍，