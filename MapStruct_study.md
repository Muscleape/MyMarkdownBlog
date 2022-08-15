# MapStruct分享

## java注解介绍

### 注解的编译时和运行时处理

#### 编译时处理

- 原理：APT技术
- 处理对象：@Retention=Source的注解
- 编译时处理需要使用到APT技术，该技术提供了一套编译期的注解处理流程。
- 在编译期扫描.java文件的注解，并传递到注解处理器，注解处理器可根据注解生成新的.java文件，这些新的.java问和原来的.java一起被javac编译。

##### 注解处理器？

注解处理器是（Annotation Processor）是javac的一个工具，用来在编译时扫描和编译和处理注解（Annotation）。你可以自己定义注解和注解处理器去搞一些事情。一个注解处理器它以Java代码或者（编译过的字节码）作为输入，生成文件（通常是java文件）。这些生成的java文件不能修改，并且会与其他手动编写的java代码一样会被javac编译，其实，就是把标记了注解的类，变量等作为输入内容，经过注解处理器处理，生成想要生成的java代码。

**注意**：注解处理器不能修改已经存在的Java类（即不能向已有的类中添加方法）。只能生成新的Java类。

jdk7之前访问和处理Annotation的工具统称APT（Annotation Processing Tool)(jdk7后就被废除了），jdk7及之后采用了JSR 269 API【**Pluggable Annotation Processing API(插件式注解处理器)**】。

JSR-269提供一套标准API来处理Annotations，具体来说，我们只需要继承AbstractProcessor类，重写process方法实现自己的注解处理逻辑，并且在META-INF/services目录下创建javax.annotation.processing.Processor文件注册自己实现的Annotation Processor，在javac编译过程中编译器便会调用我们实现的Annotation Processor，从而使得我们有机会对java编译过程中生产的抽象语法树进行修改。

##### Java代码编译过程

Java代码编译和执行的整个过程包含了以下三个重要的机制：

- 1）Java源码编译机制；
- 2）类加载机制；
- 3）类执行机制

其中，Java源码编译由以下三个过程组成：
- 1）分析和输入到符号表；
- 2）注解处理；
- 3）语义分析和生成class文件

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152003997.png?token=AD7IYN3NXD5BOGAS4WR5X33C7I3HE)

其中的annotation processing就是代码的注解处理


#### 运行时处理

- 原理： 通过反射机制获取注解。
- 处理对象：@Retention=RUNTIME的注解
    - 判断某个注解是否存在于Class、Field、Method或Constructor
    ```java
    Class.isAnnotationPresent(Class)
    Field.isAnnotationPresent(Class)
    Method.isAnnotationPresent(Class)
    Constructor.isAnnotationPresent(Class)
    ```
    - 直接从Class、Field、Method或Constructor读取到Annotation
    ```java
    Class.getAnnotation(Class)
    Field.getAnnotation(Class)
    Method.getAnnotation(Class)
    Constructor.getAnnotation(Class)
    ```
    - 读取方法参数的Annotation
    ```java
    // 要读取方法参数的注解，我们先用反射获取Method实例，然后读取方法参数的所有注解
    // 获取Method实例:
    Method m = ...
    // 获取所有参数的Annotation:
    Annotation[][] annos = m.getParameterAnnotations();
    // 第一个参数（索引为0）的所有Annotation:
    Annotation[] annosOfName = annos[0];
    for (Annotation anno : annosOfName) {
        if (anno instanceof Range) { // @Range注解
            Range r = (Range) anno;
        }
        if (anno instanceof NotNull) { // @NotNull注解
            NotNull n = (NotNull) anno;
        }
    }
    ```

**运行时处理注解的缺点**
- 1：通过反射会影响运行效率
- 2：如果注解无法保存到运行时的话，是无法使用运行时处理的

## MapStruct原理

> MapStruct是一个Java 注释处理器，用于生成类型安全的bean映射类。
> 


### 代码结构

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152005016.png?token=AD7IYN7CGAJIF6DPWVP2CZLC7I3MC)

#### 框架代码的组成部分，主要分为两个包: 
- org.mapstruct:mapstruct：包含了必要的注解，例如@Mapping; 
- org.mapstruct:mapstruct-processor：包含生成映射器实现的注解处理器。这个就是整个mapstruct框架的入口，继承了注解处理器，在java compile时将会调用process做操作；

在使用过程中需要只需要配置完成后运行 mvn compile就会发现 target文件夹中生成了一个mapper接口的实现类。打开实现类会发现实体类中自动生成了字段一一对应的get、set方法的文件。
这就是**为什么mapstruct的效率比较高的原因**，相比于反射获取对象进行拷贝的方法，这种更贴近于原生get、set方法的框架显得更为高效。

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152007439.png?token=AD7IYNZE5N5RMGOLDXVLTN3C7I3UK)

### MapStruct对JSR 269 API的实现

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152009242.png?token=AD7IYN6W5VBGYMZZXV6A2PTC7I33O)

MappingProcessor在process方法中，通过SPI机制加载MapStruct自定义的ModelElementProcessor实现的责任链，每一个我们在代码中定义的@Mapping接口，经过链路上的ModelElementProcessor处理后，生成目标java文件；

- MappingProcessor继承抽象类AbstractProcessor，重写process()实现mapstruct的逻辑


![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152011224.png?token=AD7IYNYUFJG3VA746NTOIM3C7I4CI)

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152014997.png?token=AD7IYN6LIYTOCEGN5ZNBBRTC7I4MS)

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152018860.png?token=AD7IYN23I7ESDFOW5H3X3KLC7I45M)

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152021836.png?token=AD7IYN3PSH3QL47R4IVTPBLC7I5HU)

![](https://raw.githubusercontent.com/Muscleape/MyMarkdownBlog/main/images/202208152024308.png?token=AD7IYNZMMAY3AYRRKQA57XLC7I5T4)
