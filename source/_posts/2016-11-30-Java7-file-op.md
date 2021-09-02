---
layout: post
title: "Java7 文件夹操作"
date: 2016-11-30
tags: 文件夹操作
categories: Java
---


## 复制文件夹 &  删除文件夹
```java
public static void copy(String src, String dst) throws IOException {
    final Path srcPath = Paths.get(src);
    Path destPath = Paths.get(dst);
    if (Files.notExists(destPath)) {
        Files.createDirectory(destPath);
    }
    Files.walkFileTree(srcPath, new FileVisitor<Path>() {
        @Override
        public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
            String dirStr = dir.toString();
            String r = dirStr.substring(src.length(), dirStr.length());
            Path dstPath = Paths.get(dst, r);
            if (Files.notExists(dstPath)) {
                Files.createDirectory(dstPath);
            }
            return FileVisitResult.CONTINUE;
        }

        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            String pathStr = file.toString();
            String relative = pathStr.substring(src.length(), pathStr.length());
            String destFile = Paths.get(dst, relative).toString();

            Files.copy(file, new FileOutputStream(destFile));

            return FileVisitResult.CONTINUE;
        }

        @Override
        public FileVisitResult visitFileFailed(Path file, IOException exc) throws IOException {
            return FileVisitResult.CONTINUE;
        }

        @Override
        public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
            return FileVisitResult.CONTINUE;
        }
    });
}

public static void clean(Path path) throws IOException {
    if (!Files.exists(path)) {
        return;
    }
    if (!Files.isDirectory(path)) {
        Files.delete(path);
        return;
    }
    Files.walkFileTree(path, new FileVisitor<Path>() {
        @Override
        public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
            return FileVisitResult.CONTINUE;
        }

        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            Files.delete(file);
            return FileVisitResult.CONTINUE;
        }

        @Override
        public FileVisitResult visitFileFailed(Path file, IOException exc) throws IOException {
            throw exc;
        }

        @Override
        public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
            Files.delete(dir);
            return FileVisitResult.CONTINUE;
        }
    });
}
```
## 不过这个似乎有些问题，如果碰到有软链的情况怎么办？