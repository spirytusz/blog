---
title: Gson原理分析
date: 2021-10-12 22:04:58
header-img: "cover.jpeg"
author:     "spirytusz"
tags:
- json
---

# 1. 前言
[Gson](https://github.com/google/gson)是一个Java平台的Json库，用于json的序列化和反序列化。

实践发现对于大的json字符串转换成实例的时间性能并不是很好。

本文将从源码的角度探索Gson反序列化json耗时的原因，并给出大致的解决方案。

<!--more-->

# 2. 太长不看
Gson反序列化耗时的原因有三（影响大小递减）：

1. 默认的`TypeAdapter`会反射遍历类及其超类的所有字段，并生成一个Map表；
2. 默认的`TypeAdapter`在setValue时使用到了反射；
3. 默认的`TypeAdapter`使用到了反射来创建实例。

解决方案就是书写`TypeAdapter`。

但书写TypeAdapter十分麻烦，可以考虑以下两种方法（或者其他更好的）方法来生成代码：

1. 注解处理器(kapt)生成代码。

    优点：易于接入
        
    缺点：kapt期间无法获取一些kotlin代码特性（如泛型可空）；对编译期有侵入。

2. 静态分析代码语法树并生成代码。

    优点：对kotlin语言特性**完全**支持；对项目代码和编译期零侵入。
    
    缺点：每种语言都要适配一份（不同语言的语法树不同）。

# 3. 基本用法

工欲善其事，必先利其器。首先需要了解一下如何使用Gson。

对下面这个类：

![Foo.jpeg](Foo.jpeg)

从json字符串生成一个`Foo`实例，可以调用`Gson.fromJson`方法

![common_use.jpeg](common_use.jpeg)

同时，Gson也支持泛型：

![parameterized_use.jpeg](parameterized_use.jpeg)

# 4. 源码解读
在分析Gson源码前，我们可以对Gson反序列化流程和序列化流程进行一个大致的猜想。

Gson反序列化时也许会是这样的：

1. 根据type来反射创建一个实例；
2. 以key-value的形式，读取json字符串，并反射的设置值；
3. 返回这个实例。

![简陋流程.png](简陋流程.png)

这只是大致的猜想，具体还是要看看源码，验证or推翻我们的猜想。

## 4.1. 前置知识
在阅读Gson源码之前，需要知道一些前置知识，这样在我们读完源码之后，对Gson源码能有一个更加清晰的、立体的理解。

### 4.1.1. TypeAdapter
TypeAdapter是Gson内部fromJson和toJson的通用接口，Gson内部调用read方法来将json字符串转换为实例。

返回类型`T`，代表fromJson的返回；

`JsonReader`能够以key-value的形式，流式读取json。

![TypeAdapter代码.jpeg](TypeAdapter代码.jpeg)

Gson内置了许多类型的TypeAdapter，Gson默认使用这些TypeAdapter来进行fromJson和toJson：

![内置TypeAdapter.jpeg](内置TypeAdapter.jpeg)

### 4.1.2. TypeAdapterFactory
`TypeAdapterFactory`是`TypeAdapter`的工厂类，负责创建指定类型的`TypeAdapter`。

![TypeAdapterFactory代码.jpeg](TypeAdapterFactory代码.jpeg)

在Gson实例中，`TypeAdapterFactory`被保存在一个叫做factories的List中：

![GsonTypeAdapterFactory线性列表.jpeg](GsonTypeAdapterFactory线性列表.jpeg)

## 4.2. 解析Json
`TypeAdapterFactory`和`TypeAdapter`都是fromJson的重要角色。不妨我们从入口开始分析代码：

![entry.jpeg](entry.jpeg)

通常我们都是这么调用Gson来解析Json的。

点进源码，一路追踪到fromJson的重载方法：

![fromJson.jpeg](fromJson.jpeg)

可以看到fromJson大致分为两步走：

1. 根据TypeToken获取`TypeAdapter`;
2. 调用TypeAdapter的read方法生成一个实例并返回。

首先咱们看看getAdapter是如何获取`TypeAdapter `的。


### 4.2.1. 解析Json-获取TypeAdapter

![getAdapter.jpeg](getAdapter.jpeg)

可以看到逻辑大致分为两个部分：

1. 尝试从缓存typeTokenCache内取；
2. 根据typeToken来在fatories里面线性查找。

首先会尝试从缓存中取；如果缓存中没有，getAdapter会遍历factories的每一个元素：如果这个`TypeAdapterFactory`能够创建这个类型（typeToken）的`TypeAdapter`，就会返回一个非空的`TypeAdapter`，否则就返回空。

因为这里使用到的是默认的Gson实例，并没有`Foo`这个类对应的`TypeAdapter `，所以最终getAdapter会返回一个`ReflectiveTypeAdapterFactory`实例（factories的最后一个元素），使用它来创建一个`TypeAdapter`。

`ReflectiveTypeAdapterFactory`的create方法返回了一个内部类（`ReflectiveTypeAdapterFactory.Adapter`）的实例，就是`TypeAdapter`。

![ReflectiveTypeAdapterFactory.jpeg](ReflectiveTypeAdapterFactory.jpeg)

可以看到Adapter构造方法的第二个参数调用了getBoundFields方法，这就是耗时所在：

![遍历字段.png](遍历字段.png)

这里的type是带有泛型的类型，raw就是不带泛型的类型。

比如:

| 类型 | type | raw |
| --- | --- | --- |
| Int | Int | Int |
| List\<Int\> | List\<Int\> | List |
| List\<List\<Int\>\> | List\<List\<Int\>\> | List |

阅读这段代码后，不难看出getBoundFileds做了下面几件事：

1. 遍历当前类（raw）及其超类的所有字段；
2. 对于每个字段，都会创建一个BoundFiled；

BoundFiled可以看做是对字段的封装，提供了对字段的读写能力。

由于这个方法有大量的反射逻辑，因此这个方法在首次调用时十分的耗时。

接下来看看`ReflectiveTypeAdapterFactory.Adapter`的read方法：

![Reflective.read.jpeg](Reflective.read.jpeg)

可以很明显的看到，read方法大致分为两步

1. 创建实例；
2. 遍历json字符串的key-value并设置值。

### 4.2.2. 解析Json-创建实例

先来看看Gson是如何构造一个Foo实例。

constructor能够生成对应类型的实例，它是实例化`ReflectiveTypeAdapterFactory.Adapter`的时候传入的。

在`ReflectiveTypeAdapterFactory`中，是通过`ConstructorConstructor.get`来创建的constructor的。

看看`ConstructorConstructor.get`内部干了啥：

情况1:
![constructor_1.jpeg](constructor_1.jpeg)

情况2:
![constructor_2.jpeg](constructor_2.jpeg)

情况3:
![constructor_3.jpeg](constructor_3.jpeg)

情况4:
![constructor_4.jpeg](constructor_4.jpeg)

情况5:
![constructor_5.jpeg](constructor_5.jpeg)

Gson内部在实例化一个对象时，大致分为了四种方式，这四种方式能覆盖所有的情况:

1. 使用事先设置好的实例构造器去构造实例，对应情况1和情况2；
2. 使用类的无参来构造实例，对应情况3；
3. 构造集合类型的实例，对应情况4；
4. 上面三种情况都不可行的话，使用兜底策略，使用unsafe直接构造实例，对应情况5。

具体到这个例子上，创建`Foo`实例时对应的是情况3（`Foo`的所有字段都有默认值编译成Java代码时会有一个无参构造方法）。

### 4.2.3. 解析Json-设置字段值
再回到`ReflectiveTypeAdapterFactory.Adapter`的read方法。

Gson使用`JsonReader`，从json中读取key-value，并调用BoundField.read方法来把读到的值设置进去。

此后，Gson只需不断的调用`JsonReader`的next系列方法来读取json字符串的key-value值，然后把值通过BoundField设置进对应的字段。

至此，设置值的流程结束。

### 4.2.4. 总结
Gson解析json的逻辑看起来并不复杂，主要分为三大步：

1. 获取TypeAdapter；
2. 反射创建实例；
3. 反射设置值。

在获取TypeAdapter的时候，由于`Foo`没有事先把写好的TypeAdapter给设置到Gson实例内，Gson内部在获取TypeAdapter时，最终会获取到`ReflectiveTypeAdapterFactory.Adapter`这个TypeAdapter；在创建`ReflectiveTypeAdapterFactory.Adapter`时，需要反射遍历类及其超类的所有字段；

在反射创建实例的时候，由于没有把实现写好的InstanceCreator给设置到Gson实例内，Gson内部就会反射`Foo`的无参构造方法来创建实例(`Foo`的所有字段都有默认值，编译成Java时会有一个无参构造方法)；

在反射设置值时，通过事先创建好的BoundField，调用其read方法将值设置到这个字段中。

最终只需要返回这个实例即可。

# 5. 耗时点
从上文中不难看出，裸Gson对json字符串的解析时存在性能瓶颈的(按影响的大小排序，由大到小)：

1. 默认的TypeAdapter会反射遍历类及其超类的所有字段，并生成一个Map表；
2. 默认的TypeAdapter在setValue时使用到了反射；
3. 默认的TypeAdapter使用到了反射来创建实例。

火焰图能够印证我的结论。

![火焰图.png](火焰图.png)

# 6. 解决方法

## 6.1. 手写TypeAdapter

书写TypeAdapter是一个重复性劳动的体力活。主要分为两部分，toJson部分（对应write方法）和fromJson部分（对应read方法）。这里只讨论反fromJson部分，以上面的类`Foo`为例。

![manually_write_type_adapter.jpeg](manually_write_type_adapter.jpeg)

这是对json字符串进行fromJson的手写TypeAdapter，要分为三步：

1. 先定义好临时变量；
2. 不断地从json字符串中读取key-value；
3. 把所有临时变量组装成一个实例返回。

可以看到我仅仅是为了读取4个字段并生成实例，就写了很多行代码，并且这段代码还没考虑异常的场景，十分麻烦。

## 6.2. 使用Kapt生成代码
不难看出上面手写的代码具有一定的规律，可以使用代码生成技术来生成TypeAdapter。

对于read方法的生成，无外乎三步

1. 生成临时变量，以保存读取到的值；
2. 生成一个while - when表达式，不断的调用`JsonReader`的方法读取key-value，存储到这些临时变量中；
3. 把所有的临时变量装配成返回类型并返回。

需要注意的是，第二步需要对不同的类型抽象出一个接口，每个实现专门读取每种类型的数据。

但Kapt有它的局限性：

1. kapt阶段代码已经被编译成了java代码，许多kotlin特性均已丢失；
2. 对编译阶段有侵入，需要处理的类越多，越耗时。

第一个局限性好解决，一是读取kotlin metadata，二是直接使用KSP。

第二个局限性不好解决，Kapt是编译期的工具，注定是侵入编译期的。

## 6.3. 编写IDEA Plugin
使用IDEA Plugin，能够直接解决Kapt的第二个局限性：对编译期有侵入；
并且，IDEA 提供了一个PSI (Project Structure Interface)，它是对抽象语法树AST(Abstract Syntax Tree)的封装。使用PSI，能够静态分析代码，以及生成代码。

但IDEA Plugin也有局限性：

因为是分析代码语法树，不同的语言语法树不同，所以要分别适配。

# 7. 总结
Gson的默认fromJson逻辑耗时，是因为使用到了大量的反射。

因此我们可以重新书写一个没有反射的逻辑，即TypeAdapter，来提速。

写TypeAdapter有不同的方案，不同的方案各有优劣，可以根据实际情况按需使用。