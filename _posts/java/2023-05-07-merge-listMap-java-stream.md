---
title: List<Map>合并(Java Stream)
author: Oriental Ming
date: 2023-05-07 13:02:00 +0800
categories: [Java, Stream]
tags: [Java]
render_with_liquid: false
---

# Welcome

借助 `JDK(version>7) Stream` 的便利性，对两个 `List<Map>` 依据业务要求进行合并。
依据业务主键(`id 或 code 或 xxx`)对值对象进行合并。

## 1.样例数据

```java
 // ============== new三条源数据，value值均为一个字，加入list ==================
        Map<String, Object> map1 = new HashMap<>();
        map1.put("id", "1");
        map1.put("ab", "甲");
        map1.put("ac", "乙");
        Map<String, Object> map2 = new HashMap<>();
        map2.put("id", "2");
        map2.put("ab", "丙");
        map2.put("ac", "丁");
        Map<String, Object> map3 = new HashMap<>();
        map3.put("id", "3");
        map3.put("ab", "小果");
        map3.put("ac", "相机");

        List<Map<String, Object>> sourceList = new ArrayList<>();
        sourceList.add(map1);
        sourceList.add(map2);
        sourceList.add(map3);

        // ============== new三条新数据，value值均为两个字，加入list ==================
        Map<String, Object> newMap1 = new HashMap<>();
        newMap1.put("id", "1");
        newMap1.put("ww", "小强");
        newMap1.put("nn", "小张");
        Map<String, Object> newMap2 = new HashMap<>();
        newMap2.put("id", "2");
        newMap2.put("ww", "王红");
        newMap2.put("nn", "王亮");
        Map<String, Object> newMap3 = new HashMap<>();
        newMap3.put("id", "3");
        newMap3.put("ww", "朱大");
        newMap3.put("nn", "朱二");

        List<Map<String, Object>> newList = new ArrayList<>();
        newList.add(newMap1);
        newList.add(newMap2);
        newList.add(newMap3);

        // ============ 把newList的所有内容添加到sourceList中 ==============
        sourceList.addAll(newList);

        sourceList.forEach(System.out::println);
        System.out.println();

        /*
         输出结果：
            {ab=甲, ac=乙, id=1}
            {ab=丙, ac=丁, id=2}
            {ab=小果, ac=相机, id=3}
            {ww=小强, nn=小张, id=1}
            {ww=王红, nn=王亮, id=2}
            {ww=朱大, nn=朱二, id=3}
         */
```

## 2.目标结果

```java
        /*
        合并相同id下的value集合，我们要求合并后的结果：
            {nn=小张, ww=小强, ab=甲, ac=乙, id=1}
            {nn=王亮, ww=王红, ab=丙, ac=丁, id=2}
            {nn=朱二, ww=朱大, ab=小果, ac=相机, id=3}
         */
```

## 3.策略

```java
        // ================ 利用Java8的Stream流实现合并 =========================
        List<Map<String,Object>> combine = sourceList.stream()
                // 根据map中id的value值进行分组, 这一步的返回结果Map<String,List<Map<String, Object>>>，目的是将相同id下的value归类到一个value下
                .collect(Collectors.groupingBy(group -> group.get("id").toString()))
                .entrySet() // 得到Set<Map.Entry<String, List<Map<String, Object>>>
                .stream() // 使用Java8的流
                .map(m -> { // 进入映射环境
                    // m.getValue()的结果是 List<Map<String, Object>>
                    Map<String, Object> collect = m.getValue().stream()
                            // 核心重点！o.entrySet() 的结果是 Set<Map.Entry<String, Object>>，通过flatMap将value合并成一个stream
                            .flatMap(o -> o.entrySet().stream())
                            // (m1, m2) -> m2 的意思是如果 m1 == m2 则使用m2
                            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue, (m1, m2) -> m2));
                    return collect;
                }).collect(Collectors.toList());

        // 输出测试，
        combine.forEach(System.err::println);

        /*
          测试结果：
             {nn=小张, ww=小强, ab=甲, ac=乙, id=1}
             {nn=王亮, ww=王红, ab=丙, ac=丁, id=2}
             {nn=朱二, ww=朱大, ab=小果, ac=相机, id=3}

          达到目标要求(●'◡'●)
         */

```
