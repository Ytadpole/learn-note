## mybatis 动态sql

### 测试例子

java程序

![image-20190429141657869](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429141657869.png)

xml文件

![image-20190429141822905](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429141822905.png)



### 调试

直接进入到解析select语句的地方

![image-20190429142103850](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429142103850.png)



进入

![image-20190429142149442](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429142149442.png)



#### 解析xml

进入buildStatementFromContext,依然是交给XMLStatementBuilder解析

核心部分创建SqlSource

![image-20190429142510942](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429142510942.png)



交给XMLLanguageDriver后再转到XMLScriptBuilder

先读取xml配置， 读到一个MiedSqlNode中，主要是将xml节点的层次结构。

![image-20190429142928655](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429142928655.png)



##### 解析语句第一部分

先解析<#text>
    SELECT *
    FROM POST P
    WHERE id in
    </#text>

比较简单，直接封装成StaticTextSqlNode。

![image-20190429143258080](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429143258080.png)



##### 解析Foreach

 <foreach open="(" index="index" item="item" collection="list" close=")">
<if test="index != 0">,</if>
</foreach>

![image-20190429143431508](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429143431508.png)



进入ForeacHandler, 

1. 第一步继续对当前标签内部 进行解析
2. 将内部属性，解析返回封装成 ForEachSqlNode，可以理解为他自己是个ForEachSqlNode
3. 将它自己（ForEachSqlNode）加入到他的上级中，<font color=red>这部分代码本身是一个递归操作</font>

![image-20190429143736023](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429143736023.png)



进入内部标签解析，开始解析foreach内部，第一个元素是一个文本，仅仅含有一个\n换行，所以最后封装成一个StaticTextSqlNode

![image-20190429144242743](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429144242743.png)



第二次循环，解析<if test="index != 0">,</if>

根据标签名，选择ifHandler处理

![image-20190429144928267](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429144928267.png)



在IfHandler中，与ForeachHanlder相似，有3步

1. 解析内部元素
2. 处理自身需要的属性，并封装为IfSqlNode
3. 将自身加入到父级SqlNode

![image-20190429145303964](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429145303964.png)



###### 解析If

内部就是一个逗号，所以处理 就直接封装成一个StaticTextSqlNode

![image-20190429145555994](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429145555994.png)



###### 解析 #{item}

比较简单，也是一个StaticTextSqlNode

![image-20190429145933530](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429145933530.png)



##### 最后还有一个\n换行<font color=red>(后面继续调试，发现后有个循环)</font>

问题不大，就是一个StaticSqlTextNode



##### 解析完毕 查看MiexedSqlNode节点

是一个树形结构  *未包含最后一个换行*

![image-20190429150549158](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429150549158.png)



if节点展开

![image-20190429150625961](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429150625961.png)



##### 解析完毕 封装DynamicSqlSource

明显会封装成DynamicSqlSource

所有内部含有子标签，或者有${}未处理完毕的都是动态sql

![image-20190429151306144](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429151306144.png)



##### 封装成MappedStatement

通过一个简单的构建者模式，生成MappedStatement，保存到configuration，这部分代码不截图了，意义不大。



#### 执行sql

![image-20190429152546576](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429152546576.png)



直接进入到下图

1. 先通过id拿到前面通过xml解析封装成的MappedStatement
2. 封装查询参数
3. 交给执行器去执行

![image-20190429152655108](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429152655108.png)



封装查询参数比较简单

1. 集合，数组封装成map，key不一样而已
2. 对象原样返回

![image-20190429153000862](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429153000862.png)



##### 正式开始

先交给CacheExecutor执行，先拿取到BoundSql

通过MappedStatement去拿，MappedStatement转sqlSource处理，我们这里的sqlSource是DynamicSqlSource。

![image-20190429153338743](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429153338743.png)



进入到DynamicSqlSource

可以看到第一二行代码，进行动态处理。

与StaticSqlSource不同，StaticSqlSource只需要封装一个BoundSql返回即可，因为解析xml阶段就处理完毕了。DynamicSqlSource还需要在执行过程中进行处理。

![image-20190429153733567](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429153733567.png)



实例化一个DynamicContext对象

![image-20190429154342143](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429154342143.png)



核心第二步开始 对sqlNode树进行操作

```java
rootSqlNode.apply(context);
```



MiexedSqlNode转交内部结构自己去处理

![image-20190429154540095](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429154540095.png)



##### 处理第一个文本节点

  SELECT *
    FROM POST P
    WHERE id in 

这个StaticSqlNode先行处理，比较简单，直接拼接文本



##### 处理foreach节点

对foreach进行处理

![image-20190429155624344](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429155624344.png)



在保存对应参数处理中，将参数存储为自身定义的格式，后续进行相应的替换

![image-20190429155813718](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429155813718.png)





内部节点继续进行处理

同样通过MixedSqlNode交给具体SqlNode进行处理

_第一个换行节点跳过_

###### 解析if

![image-20190429160723980](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429160723980.png)

因为是第一个节点 test内容是 index!=0所以会返回false，

否则会进入判断内部，继续对内部的节点进行处理。

后续第二次第三次，会进入判断里面，处理的会将 内部的StaticSqlNode节点进行处理。

因为就是一个逗号文本， 所以返回文本逗号。

<font color=red>这里的处理就是逗号分隔，第一个不要逗号。</font>



###### 解析#{item}

交给Foreach节点下的FilteredDynamicContext处理

![image-20190429161244823](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429161244823.png)



通过GenerictokenParser去解析，token解析器将对应的标签专成 前面存储的类型。

再拼接到语句后面。

![image-20190429161924868](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429161924868.png)

![image-20190429162928020](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429162928020.png)



主要先通过FilteredDynamicContext对元素进行封装，封装格式

```
__frch_{item}_{index} 
```



PrefixedContext



##### DynamicContext生成完毕

![image-20190429163325792](/Users/yangsong/learn-note/images/mybatis-dynamic-sql/image-20190429163325792.png)



##### SqlSourceBuilder解析sql生成StaticSqlSource

和静态解析一样，对#{}占位符转？处理



##### 直接到SimpleExecutor的doQuery方法

先生成statementHandler

前面获取到了再和对应的静态执行一样。



完毕。

#### 总结

后续对静态sql再进行深入研究。

xml解析理解更加深刻了。对于树结构处理，其实就用到了先序遍历。

对他内部的抽象理解更近一步。



