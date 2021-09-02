---
layout: post
title: "记一道 Java 笔试题"
date: 2017-12-24
tags: 文件
categories: 算法
---

# 题目
100+ 文件，1W+ 单词，设计用 5 个线程统计单词出现次数,输出次数最多的 100 个单词和次数

当时做的很烂，写了快 200 行，一开始的想法是 5 个线程 5 个 map 分开统计，然后归并，最后遍历插入一个 100 大小的有序数组。

后来想想，其实好像有个简单的解法，毕竟这个单词总数又不是很多， 30 行代码就搞定了
```
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

public class Main {

    public static void main(String[] args) throws IOException, InterruptedException {
        String filesDir = args[0];
        ExecutorService pool = Executors.newFixedThreadPool(5);
        Map<String, AtomicInteger> countMap = new ConcurrentHashMap<>();
        List<Path> paths = Files.list(Paths.get(filesDir)).filter(path -> path.toFile().isFile()).collect(Collectors.toList());
        System.out.println(paths.size());
        CountDownLatch countDownLatch = new CountDownLatch(paths.size());
        paths.forEach(path -> pool.execute(() -> {
            try {
                Files.lines(path).forEach(line -> {
                    String[] words = line.split(" ");
                    for (String word : words) {
                        String w = word.trim();
                        if (!w.isEmpty()) {
                            if (countMap.get(w) == null) {
                                countMap.putIfAbsent(w, new AtomicInteger());
                            }
                            countMap.get(w).incrementAndGet();
                        }
                    }
                });
            } catch (IOException e) {
                e.printStackTrace();
            }
            countDownLatch.countDown();
        }));
        countDownLatch.await();
        pool.shutdown();
        List<Map.Entry<String, AtomicInteger>> top100Wrods = countMap.entrySet().stream()
                .sorted((a, b) -> b.getValue().get() - a.getValue().get()).limit(100).collect(Collectors.toList());
        System.out.println(top100Wrods);
    }
}
```