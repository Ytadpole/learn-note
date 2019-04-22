## mybatis 动态sql分析1

```xml
分析这条sql执行 select * from Blog where id = #{id} 如何转为 select * from Blog where id = ？
```

#### 分析

xml文件

![image-20190422164823500](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422164823500.png)



先进入mapper解析

![image-20190422164128805](images/mybatis-dy-sql1/image-20190422163929092.png)



进入语句解析

![image-20190422164240711](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422164240711.png)



转交XMLStatmentBuilder解析

![image-20190422164429027](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422164429027.png)



parseStatementNode方法核心点，转给LanguageDirver，这里是XMLLanguageDirver。

第三个参数parameterTypeClass是前面读取参数，在typeAliase注册中心拿到的class
![image-20190422164600871](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422164600871.png)



XMLLanguageDirver做一个透传，交给XMLScriptBuilder处理生成SqlSource

![image-20190422164913965](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422164913965.png)



进入XMLScriptBulder解析方法

![image-20190422165118955](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422165118955.png)

主要生成MixedSqlNode对象，再将MixSqlNode封装到SqlSource里面返回



MixSqlNode内部包含的是SqlNode的集合，<font color=red>这个目前不太清楚</font>



实例话TextSqlNode，再解析

![image-20190422170122124](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422170122124.png)

GenericTokenParser作为一个解析xml中的占位符的解析器

对于${}占位符的解析


![image-20190422170418092](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422170418092.png)



因为语句中不包含${}占位符,所以执行到这里结束。

![image-20190422170601265](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422170601265.png)



dynamic就为false，后续代码进入到下图，实例话StaticTextSqlNode

![image-20190422170800068](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422170800068.png)



因为仅仅包含一条sql，contents就包含一个StaticTextSqlNode，生成MiexedSqlNode对象返回

![image-20190422170934309](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422170934309.png)



在XMLScriptBuilder中可以看到

![image-20190422171201212](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422171201212.png)



执行过程中，未将dynamic属性更新为true，所以实例化RawSqlSource

![image-20190422171443344](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422171443344.png)



实例化RowSqlSource， 先由sqlNode生成sql语句进入getSql()方法，

![image-20190422171700105](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422171700105.png)



通过MixedSqlNode去遍历其内部的SqlNode，并执行对应的apply方法。

![image-20190422172029624](/Users/yangsong/Library/Application Support/typora-user-images/image-20190422172029624.png)

![image-20190422172001295](/Users/yangsong/Library/Application Support/typora-user-images/image-20190422172001295.png)

这里contents里面就一个StaticTextSqlNode

![image-20190422172240279](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422172240279.png)



![image-20190422172404222](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422172404222.png)

做了一个简单处理，用到了StringJoiner，处理比较简单。



建立SqlSourceBuilder，让他处理

![image-20190422172813503](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422172813503.png)

![image-20190422173034926](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422173034926.png)

和前面解析${}占位符一样， 这次解析#{}占位符，处理者为ParameterMappingTokenHandler实例



前面就是一些取占位符开始，结束的计算， 读到后交给ParameterMappingTokenHandler处理。

![image-20190422173504799](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422173504799.png)



ParameterMappingTokenHandler作为参数映射处理者，将参数转为？并存储参数映射信息存到内存

![image-20190422173650398](/Users/yangsong/learn-note/images/mybatis-dy-sql1/image-20190422173650398.png)

最终完毕。