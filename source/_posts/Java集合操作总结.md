---
title: Java集合操作总结
date: 2016-12-27 20:13:03
tags:
---
* List和Array之间转换

```java
String [] arr = new String[]{"test", "test1"};
List list = Arrays.asList(arr);

List<String> list = new ArrayList<String>();
list.add("str1");
list.add("str2");
int size = list.size();
String[] arr = (String[])list.toArray(new String[size]);
```