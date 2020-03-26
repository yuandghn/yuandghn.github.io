---
layout:     post
title:      "How to convert number to spell-out format in Java"
subtitle:   ""
date:       2020-03-26 16:25:00
author:     "Echo Yuan"
tags:
    - Number
    - Spell-out
    - Convert
---

有时候工作中需要将数字转换成单词拼写出的形式，如果有现成的肯定不去自己写啦~  [ICU Project](http://site.icu-project.org/)就可以帮你完成。

因为项目用的是Java，所以你可以从[这里](http://site.icu-project.org/download/66#TOC-ICU4J-Download)下载Jar包来用，当然更通用的方式是把它当作依赖[引用](https://mvnrepository.com/artifact/com.ibm.icu/icu4j/66.1)进来啦~

Maven
```xml
<dependency>

  <groupId>com.ibm.icu</groupId>

  <artifactId>icu4j</artifactId>

  <version>66.1</version>

</dependency>
```

Gradle
```xml
compile group: 'com.ibm.icu', name: 'icu4j', version: '66.1'
```

以下是个Groovy写的例子

```groovy
RuleBasedNumberFormat ruleBasedNumberFormat = new RuleBasedNumberFormat(new Locale("EN", "US"), RuleBasedNumberFormat.SPELLOUT)
[
        12345,
        300,
        1,
        12,
        98,
        101
].each {
    System.out.println(ruleBasedNumberFormat.format(it))
}
```

输出结果
```text
twelve thousand three hundred forty-five
three hundred
one
twelve
ninety-eight
one hundred one
```

Enjoy it.