---
layout: post
title: "file-leak-detector 检查Java文件句柄泄露"
date: 2017-01-02
tags: file leak
categories: Java
---

> 维护 WEB-IDE 免不了要管理很多的文件， 自从我们 线上的系统增加了资源回收功能，便一直受一个问题困扰，后台线程解绑目录时偶尔报错，看症状因为是某些文件被占用了，目录不能解绑。但是由于系统中很多地方都有打开文件，各种包也存在复杂的的引用关系，在搜查几遍代码后并没有发现什么明显的异常。

由于这个功能清理的是既没在线又没有在离线列表中的磁盘绑定目录，那么很可能是文件句柄泄露了，还有一种原因可能是 JVM 延迟释放文件句柄，不过实际是什么原因还需要用数据说话。

经过一番搜索，发一个工具叫 file-leak-detector， 可以监控什么线程在什么时候打开了哪儿的文件，看起来好酷，官网在这里：  
[http://file-leak-detector.kohsuke.org](http://file-leak-detector.kohsuke.org)

## 使用方式

### 监听 HTTP 端口方式启动   
以 javaagent 方式启动一个 jar 文件，输出在 http 19999 端口。
```
$java -javaagent:./file-leak-detector-1.8-jar-with-dependencies.jar=http=19999 -jar ide-backend.jar
```
然后直接在浏览器访问刚刚启动时配置的 http端口：
 ![图片](https://dn-coding-net-production-pp.qbox.me/fc6fdcc0-bf03-499a-bb46-2834e6aa9ba9.png) 
可以看到当前所有打开中的文件的堆栈信息。

### 配置参数启动

配置线程数量限制,在文件句柄持有数超过设定数值时输出所有文件打开时的堆栈信息到 System 的 err 日志中。
```
$ java -javaagent:path/to/file-leak-detector.jar=threshold=200 ...your usual Java args follows...
```

### Attach 方式启动

启动后直接被加载到运行中的 JAVA 进程里。
```
$ java -jar path/to/file-leak-detector.jar 1500 threshold=200,strong
```
strong 代表的含义是把记录信息的应用变成强引用，防止被 GC 回收掉，不设置在内存不足时文件记录会丢失。

## 实际使用体验

首先我们在测试服务器上部署端口来监控，然后进行各种测试，最后确实找到几处未关闭的代码。
```
$java -javaagent:./file-leak-detector-1.8-jar-with-dependencies.jar=http=19999 -jar xxx.jar
```
不过有一点比较不爽，绑定的地址是固定的 localhost, 远程的就不能访问。
```
╭─tiangao@tgmbp  ~/git/tiangao  ‹master*›
╰─$ curl 192.168.31.227:19999
curl: (7) Failed to connect to 192.168.31.227 port 19999: Connection refused
```
这个先放一边，官网说还可以 attach 到正在运行的进程中，这点才是我们到线上监控所需要的，有些问题只有在线上才会出现。

不过官网里并没有发现怎么挂到正在运行中的 java 程序并开启 http 端口输出，而且监听的端口只有 localhost。这就让我们感觉有点怪异，
也许有安全性的考量吧，只好去看看源码，才知道怎么个用法，为了更方便还改了下监听的 host，以便远程可以访问。

**AgentMain.java**
``` 
private static void runHttpServer(int port) throws IOException {
    final ServerSocket ss = new ServerSocket();
    ss.bind(new InetSocketAddress("0.0.0.0", port));
    System.err.println("Serving file leak stats on http://0.0.0.0:"+ss.getLocalPort()+"/ for stats");
    ...
}
```
改之后使用如下所示:
```
root@staging-1:~# java -jar file-leak-detector-1.8-jar-with-dependencies.jar 612 http=19999
Connecting to 612
Activating file leak detector at /root/file-leak-detector-1.8-jar-with-dependencies.jar
```
`612` 是 java 服务的进程号，19999 是监听的 `http` 端口号。  

执行后输出类似如下内容时即表示 `attach` 到进程成功。
```
╭─tiangao@tgmbp  ~/git/WebIDE-Backend/target  ‹master*›
╰─$ java -jar file-leak-detector-1.8-jar-with-dependencies.jar 93739
Connecting to 93739
Activating file leak detector at /Users/shitiangao/git/WebIDE-Backend/target/file-leak-detector-1.8-jar-with-dependencies.jar
```

然后通过地址加端口就可以访问,就可以显示进程在 `attach` 之后打开的文件以及相应堆栈信息。
```
3 descriptors are open
#1 /opt/coding-ide-home/ide-backend.jar by thread:qtp873134840-16 on Tue Nov 29 15:05:34 CST 2016
	at java.io.RandomAccessFile.<init>(RandomAccessFile.java:244)
	at org.springframework.boot.loader.data.RandomAccessDataFile$FilePool.acquire(RandomAccessDataFile.java:252)
	at org.springframework.boot.loader.data.RandomAccessDataFile$DataInputStream.doRead(RandomAccessDataFile.java:174)
	at org.springframework.boot.loader.data.RandomAccessDataFile$DataInputStream.read(RandomAccessDataFile.java:152)
```

如此改动测试后在本地好用，但是一到线上部署就报错了：
```
pid: 13546
Connecting to 13546
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:497)
	at org.kohsuke.file_leak_detector.Main.run(Main.java:54)
	at org.kohsuke.file_leak_detector.Main.main(Main.java:39)
Caused by: com.sun.tools.attach.AttachNotSupportedException: Unable to open socket file: target process not responding or HotSpot VM not loaded
	at sun.tools.attach.LinuxVirtualMachine.<init>(LinuxVirtualMachine.java:106)
	at sun.tools.attach.LinuxAttachProvider.attachVirtualMachine(LinuxAttachProvider.java:63)
	at com.sun.tools.attach.VirtualMachine.attach(VirtualMachine.java:208)
	... 6 more
```

目测原因是 JVM 运行时反射加载不到类。

第一感觉需要设置一下 JAVA_HOME, 然而结果证明并不是这个原因。

万能的 google & stackoverflow 找到了解法：  
[java - AttachNotSupportedException due to missing java_pid file in Attach API](http://stackoverflow.com/questions/5769877/attachnotsupportedexception-due-to-missing-java-pid-file-in-attach-api)

执行 attach 的用户需要和 Java 服务运行用户是同一个，另外 JAVA_HOME 环境变量还是需要的。

终于成功了，接下来就是等待错误的再次发生，然后分析堆栈信息了。

***如此好用的工具是让我们对其原理很好奇***。

## 工作原理

项目源码并不是太多，先看 main ：   
```
public static void main(String[] args) throws Exception {
        Main main = new Main();
        CmdLineParser p = new CmdLineParser(main);
        try {
            p.parseArgument(args);
            main.run();
        } catch (CmdLineException e) {
            System.err.println(e.getMessage());
            System.err.println("java -jar file-leak-detector.jar PID [OPTSTR]");
            p.printUsage(System.err);
            System.err.println("\nOptions:");
            AgentMain.printOptions();
            System.exit(1);
        }
    }
```

来到 run() 方法，
```
    public void run() throws Exception {
        Class api = loadAttachApi();

        System.out.println("Connecting to "+pid);
        Object vm = api.getMethod("attach",String.class).invoke(null,pid);

        try {
            File agentJar = whichJar(getClass());
            System.out.println("Activating file leak detector at "+agentJar);
            // load a specified agent onto the JVM
            api.getMethod("loadAgent",String.class,String.class).invoke(vm, agentJar.getPath(), options);
        } finally {
            api.getMethod("detach").invoke(vm);
        }
    }
```

通过 `loadAttachApi()` 得到 `VirtualMachine 类`，然后再用反射获取 `attach()` 方法，紧接着执行 `attach()` 到指定进程 id 上，得到 vm 的实例后执行 `loadAgent()` 方法，第一个参数为 **agentJar** 包的路径，第二个 **options** 是附加参数。

`loadAttachApi()` 方法如下：

```
private Class loadAttachApi() throws MalformedURLException, ClassNotFoundException {
    File toolsJar = locateToolsJar();

    ClassLoader cl = wrapIntoClassLoader(toolsJar);
    try {
        return cl.loadClass("com.sun.tools.attach.VirtualMachine");
    } catch (ClassNotFoundException e) {
        throw new IllegalStateException("Unable to find tools.jar at "+toolsJar+" --- you need to run this tool with a JDK",e);
    }
}
```

问题来了，`VirtualMachine` 是个什么功能的类？ `attach()` `loadAgent()` 又是什么作用呢？

这个就涉及到 JVM 层面提供的功能，在这之前也没有研究过，只好看看大拿的研究。  

[InfoQ JVM源码分析之javaagent原理完全解读](http://www.infoq.com/cn/articles/javaagent-illustrated)  

关键类 Instrument:

[Package java.lang.instrument](http://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html)

**简单总结，JVM 暴露了一些动态操作已加载类型的接口，javaagnet 就是利用这些接口的一个实现，通过 agent 类的固定方法可以执行一些操作，比如对已经加载的类注入字节码，最常用的是用来监控运行时，进行一些疑难 bug 追踪。**

此项目里 TransformerImpl 类就是字节码修改的实现类。   

关键源码：    
```
instrumentation.addTransformer(new TransformerImpl(createSpec()),true);
        
instrumentation.retransformClasses(
        FileInputStream.class,
        FileOutputStream.class,
        RandomAccessFile.class,
        Class.forName("java.net.PlainSocketImpl"),
        ZipFile.class);
```
可以看到注册的类有 FileInputStream、FileOutputStream、RandomAccessFile、ZipFile 和 PlainSocketImpl。
   
```
static List<ClassTransformSpec> createSpec() {
    return Arrays.asList(
        newSpec(FileOutputStream.class, "(Ljava/io/File;Z)V"),
        newSpec(FileInputStream.class, "(Ljava/io/File;)V"),
        newSpec(RandomAccessFile.class, "(Ljava/io/File;Ljava/lang/String;)V"),
        newSpec(ZipFile.class, "(Ljava/io/File;I)V"),

        /*
            java.net.Socket/ServerSocket uses SocketImpl, and this is where FileDescriptors
            are actually managed.

            SocketInputStream/SocketOutputStream does not maintain a separate FileDescritor.
            They just all piggy back on the same SocketImpl instance.
         */
        new ClassTransformSpec("java/net/PlainSocketImpl",
                // this is where a new file descriptor is allocated.
                // it'll occupy a socket even before it gets connected
                new OpenSocketInterceptor("create", "(Z)V"),

                // When a socket is accepted, it goes to "accept(SocketImpl s)"
                // where 's' is the new socket and 'this' is the server socket
                new AcceptInterceptor("accept","(Ljava/net/SocketImpl;)V"),

                // file descriptor actually get closed in socketClose()
                // socketPreClose() appears to do something similar, but if you read the source code
                // of the native socketClose0() method, then you see that it actually doesn't close
                // a file descriptor.
                new CloseInterceptor("socketClose")
        ),
        new ClassTransformSpec("sun/nio/ch/SocketChannelImpl",
                new OpenSocketInterceptor("<init>", "(Ljava/nio/channels/spi/SelectorProvider;)V"),
                new CloseInterceptor("kill")
        )
    );
}
```

`ClassTransformSpec` 定义：   
```
    /**
     * Creates {@link ClassTransformSpec} that intercepts
     * a constructor and the close method.
     */
    private static ClassTransformSpec newSpec(final Class c, String constructorDesc) {
        final String binName = c.getName().replace('.', '/');
        return new ClassTransformSpec(binName,
            new ConstructorOpenInterceptor(constructorDesc, binName),
            new CloseInterceptor()
        );
    }
```
关键真相在这里，实现了一个方法拦截适配器，在注册类的某些方法执行后运行 `Listener` 类的 `open()` 方法来记录信息。
```
    /**
     * Intercepts the this.open(...) call in the constructor.
     */
    private static class ConstructorOpenInterceptor extends MethodAppender {
        /**
         * Binary name of the class being transformed.
         */
        private final String binName;

        public ConstructorOpenInterceptor(String constructorDesc, String binName) {
            super("<init>", constructorDesc);
            this.binName = binName;
        }

        @Override
        public MethodVisitor newAdapter(MethodVisitor base, int access, String name, String desc, String signature, String[] exceptions) {
            final MethodVisitor b = super.newAdapter(base, access, name, desc, signature, exceptions);
            return new OpenInterceptionAdapter(b,access,desc) {
                @Override
                protected boolean toIntercept(String owner, String name) {
                    return owner.equals(binName) && name.startsWith("open");
                }

                @Override
                protected Class<? extends Exception> getExpectedException() {
                    return FileNotFoundException.class;
                }
            };
        }

        protected void append(CodeGenerator g) {
            g.invokeAppStatic(Listener.class,"open",
                    new Class[]{Object.class, File.class},
                    new int[]{0,1});
        }
    }
 ```
 
最后的 `append()` 方法可以说是整个流程中最核心的地方，`Listener#open()` 方法如下所示：
 ```
 public static synchronized void open(Object _this, File f) {
    put(_this, new Listener.FileRecord(f));
    Iterator i$ = ActivityListener.LIST.iterator();

    while(i$.hasNext()) {
        ActivityListener al = (ActivityListener)i$.next();
        al.open(_this, f);
    }
}
```
 最后说 一下 **`Listener`** 这个类，这也是这个工具的一个关键的类实现，有许多静态方法，所有监控的打开的文件相关内容都在 `Listener` 中保存，内容输出的操作也在其中。

这是类中的属性和方法:   
 ![图片](https://dn-coding-net-production-pp.qbox.me/b34f6f31-d3fb-42eb-a3d5-535fdd45edfb.png) 

TABLE 保存打开中的文件，默认是 weak 引用，内存不足时这个对象会被回收掉，以防止程序不会因为监控导致的内存不足而异常退出。   
当参数 strong 存在时会 new 一个 LinkedHashMap, 让监控内容的容器不会被回收掉。
 ```
 /**
 * Files that are currently open, keyed by the owner object (like {@link FileInputStream}.
 */
private static Map<Object,Record> TABLE = new WeakHashMap<Object,Record>();
 ```
 Record 中有三个字段，一个是用来保存堆栈信息的异常类型，一个是线程名，最后一个是时间。
 ```
/**
 * Remembers who/where/when opened a file.
 */
public static class Record {
    public final Exception stackTrace = new Exception();
    public final String threadName;
    public final long time;
    ...
}
 ```

到这里已经差不多了，其他细节实现也就不赘述了。

## 小结

**file-leak-detector** 查找文件句柄泄露问题，就是用 JVM 提供的接口，以 agent 方式 attach 进正在运行的 JAVA 进程，修改 `FileStream` 等类型的字节码，在 open & close 文件时加入拦截操作，记录线程和堆栈，然后在 http 或者 系统日志中输出记录。   
最后通过这些信息查找是哪里导致的问题，然后做针对性的修复。

