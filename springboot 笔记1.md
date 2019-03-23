#springboot 笔记1

### 1 开始

springboot版本2

@springBootApplicatoin核心三个注解

1. @Configuration

2. @EnableAutoConfigurarion

3. @ComponentScan

   ![image-springboot注解](/Users/yangsong/mymd/images/springboot-note1/image-springboot注解.png)

### 2 @Configuration

​	javaConfig形式的配置

![springbootConfiguration注解](/Users/yangsong/mymd/images/springboot-note1/springbootConfiguration注解.jpg)

@SpringBootConfiguration本质就是一个Configuration

###3 @ComponentScan

​	@Controller,@Service,@Component,@Responsity注解扫描路径

### 4 @EnableAutoConfiguration

![1553247005359](/Users/yangsong/mymd/images/springboot-note1/1553247005359.jpg)

#### 4.1 @AutoConfigurationPackage

这个注解的主要功能自动配置包，它会获取主程序类所在的包路径，并将包路径（包括子包）下的所有组件注册到 SpringIOC 容器中。

#### 4.2 @Import({AutoConfigurationImportSelector.class})

@EnableAutoConfiguration 的关键功能也是这个 @Import 导入的配置功能，使用 SpringFactoriesLoader.loadFactoryNames 方法来扫描具有 META-INF/spring.factories 文件的jar包