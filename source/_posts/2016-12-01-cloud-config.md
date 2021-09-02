---
layout: post
title: "Spring Cloud Config"
date: 2016-12-01
tags: Cloud Config
categories: Java
---

## 背景
现今这个时候，微服务大行其道，互联网应用遍地都是，随便开发个什么应用首要考虑的都是要可伸缩，扩展性要好。   
当我们的后台服务一点点增多，各个服务的配置也越来越多，随之而来的就是一个配置管理问题，各自管各自的开发时没什么问题，到了线上之后管理就会很头疼，到了要大规模更新的时候就更烦了。   
我们的后台服务就是如此，各种语言开发的都有，在慢慢的迭代过程的我们发现配置中心是一个比较好的解决方案，作为 Spring 的拥趸我们自然就看中了 Spring Cloud Config。     
一般服务器的应用都有以下几种类型:
 ![图片](https://dn-coding-net-production-pp.qbox.me/b6e1ba88-36a7-4fa6-ba93-2b0e501e6c1b.png) 
其中当属业务部分最多也最繁杂。
当应用变得越来越庞大和复杂时，单机就肯定不能满足需求了，然后就要考虑分布式了，接下去还可能会应用不同的语言来开发应用。   

 ![图片](https://dn-coding-net-production-pp.qbox.me/c2b3c8e9-4054-4eaa-9071-1300a731340e.png) 
比如 nginx 毫无疑问的是用的最多的反向代理组件，使用 OpenResty 便要用到 lua，再比如前端要 seo ，一个解决办法就是使用 nodejs，到了后端分布式，那就更繁多了，可能会需要把业务一个个拆分成不同的服务单独运行...   
然后就发现散落一地的配置文件，**properties**、**yml**、**json** …

为了解决这个问题，我们采取了这么一种方案，通过 etcd 来收集所有的配置，然后统一的写到一个文件中去，任何应用都来访问这一个文件来找到自己的配置并应用。
 ![图片](https://dn-coding-net-production-pp.qbox.me/3905cf7f-9fde-4460-b7d6-1f8724a7dda9.png) 

这很好的解决了配置散落的问题，可以集中管理了，但是又一个问题来了，各种类型的配置在一个文件里看起来好乱，而且在初始化时必须要完成这个文件才能启动应用。

## 解决方案
所以下一个解决方案便想到用一个配置中心来搞定。
  ![图片](https://dn-coding-net-production-pp.qbox.me/9393a355-cd47-432a-85bf-05e6e8766b7c.png) 

### Spring Cloud Config 项目
- 提供 **服务端** 和 **客户端** 支持
- **集中式** 管理分布式环境下的应用配置
- 基于 **Spring** 环境，无缝 与 Spring 应用集成
- 可用于 **任何** 语言开发的程序
- 默认实现基于 **git** 仓库，可以进行 **版本管理**
- **可替换** 自定义实现

### Spring Cloud Config Server 
作为配置中心服务端
* 拉取配置时更新 git 仓库副本，保证是最新结果
* 支持数据结构丰富，yml, json, properties 等
* 配合 eureke 可实现服务发现，配合 cloud bus 可实现配置推送更新
* 配置存储基于 git 仓库，可进行版本管理
* 简单可靠，有丰富的配套方案

### Spring Cloud Config Client 
默认客户端实现
- SpringBoot 项目不需要改动任何代码，加入一个启动配置文件指明使用 ConfigServer 上哪个配置文件即可


## 简单使用示例
### 新建仓库 
新建一个 git 仓库，添加一个配置文件。   
例如想要一个 billing的服务，性质是开发，运行环境是测试环境。   
那么就新建一个 testing 的分支，然后提交一个 billing-dev.properties 的文件
```
devMode = true
spring.application.name = billing
spring.jdbc.host = localhost
spring.jdbc.port = 3306
spring.jdbc.user = root
spring.jdbc.password = 123qwe
loging.file = demo
```

### 新建一个标准 maven 项目
![图片](https://dn-coding-net-production-pp.qbox.me/fb006d98-a00b-40d6-930a-64e14a3ea9dc.png) 
**ConfigServer.java**

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServer {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServer.class, args);
    }
}
```

**application.yml**
```
server:
  port: 8888
spring:
  cloud:
    config:
      server:
        git:
          uri:https://git.coding.net/tiangao/demo-config-server.git
          clone-on-start: true
```

**pom.xml** 
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>config-server</artifactId>
    <version>0.1.0</version>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.1.RELEASE</version>
    </parent>
    <dependencies>
        <!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-config-server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
            <version>1.2.1.RELEASE</version>
        </dependency>
    </dependencies>
</project>
```
### 启动效果
 配置中心已经可以启动,最简单的访问方式可以访问浏览器。
 ![图片](https://dn-coding-net-production-pp.qbox.me/8af714a1-4df6-4d5c-90ec-54da2512fb0a.png) 
想要 json 格式的怎么办呢
 ![图片](https://dn-coding-net-production-pp.qbox.me/5b34de0d-05df-441a-9533-00ad502718f3.png) 
还有 yml, properties
 ![图片](https://dn-coding-net-production-pp.qbox.me/4c86bfd4-8662-47a4-95b0-6201d2caf38e.png) 
 ![图片](https://dn-coding-net-production-pp.qbox.me/628bcb17-0bde-4864-99ad-f9a69e189302.png) 

### 安全配置 
 添加 Security 依赖，配置用户名和密码访问，毕竟直接通过 url 就可以访问太不安全了。
```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
```
然后在 application.yml 中配置用户名密码。
```
security:
  user:
    password: db1ab002-1fba-476d-a421-22113355
```
再访问就能看到如下的效果
 ![图片](https://dn-coding-net-production-pp.qbox.me/b7d1349b-4605-43c4-879b-d4a87c2432d6.png)
**以上就是 ConfigSever 默认实现的基本的 HttpBasic 的认证**


### 客户端配置   
SpringBoot 客户端配置，仅仅只需要添加一个启动配置文件。
```
spring:
  cloud:
    config:
      uri: http://127.0.0.1:8888
      profile: dev
      label: testing
      name: billing
      password: db1ab002-1fba-476d-a421-22113355
      username: user
```
当启动时看到下面标示的内容是即表示加载成功
 ![图片](https://dn-coding-net-production-pp.qbox.me/82d8c322-3bbd-4654-9112-75142b6ebef4.png) 
 
### 流程示意图
 ![图片](https://dn-coding-net-production-pp.qbox.me/7fa57f51-cb10-4f2b-9f54-abe35a287d0b.png) 

### 配置加载源 
所有的配置都通过仓库来管理，应用启动时访问配置中心能拿到自己的配置即可加载，接下来再进入各自的初始化过程。
仓库也不仅仅只能用 git, 还有 svn 和本地目录可以选择
- **svn://${user.home}/config-repo**
- **file://${user.home}/config-repo**

### 疑问 
从上面的简单使用里可以看出，我们只要提交一种格式的配置文件，通过不同的方式访问即可得到不同格式的配置文件，真棒。那么这在ConfigServer里是如何实现的呢？

## 原理探究
### 访问方式
**首先看一共有多少种访问方式**

- /{name}/{profiles:.*[^-].*}
- /{name}/{profiles}/{label:.*}
- /{name}-{profiles}.properties
- /{label}/{name}
- /{profiles}.properties
- /{name}-{profiles}.json
- /{label}/{name}-{profiles}.json

**name** 可以代表应用名，**profiles** 是属性，可以用来甄别是什么用途，**label** 就是 git 的标签了，大约可以用来指明环境。
接下来再看就转到这么一个包含各种资源类型的转换操作的集合类上面，同时暴露资源接口。其中方法如下图所示：

 ![图片](https://dn-coding-net-production-pp.qbox.me/42731a32-f6ae-4171-80dd-cac05b4441bc.png) 

原来内部只存在一种资源抽象：
 ![图片](https://dn-coding-net-production-pp.qbox.me/38573fcc-7fd0-4c7e-b453-b7ce62212f3f.png) 

### 资源抽象实现
一个应用的配置只需要应用名、属性和标签就可以定位到。嗯，原来这么简单，那 EnvironmentReposiroty 又是哪里实现呢？

扒扒类图：
 ![图片](https://dn-coding-net-production-pp.qbox.me/030ff2b9-dc9e-4b53-b428-da74e5d63c6c.png) 
 
**findOne()** 方法在 AbstractScmEnvironmentRepository 中可以找到
```
@Override
public synchronized Environment findOne(String application, String profile, String label) {
	NativeEnvironmentRepository delegate = new NativeEnvironmentRepository(getEnvironment());
	Locations locations = getLocations(application, profile, label);
	delegate.setSearchLocations(locations.getLocations());
	Environment result = delegate.findOne(application, profile, "");
	result.setVersion(locations.getVersion());
	result.setLabel(label);
	return this.cleaner.clean(result, getWorkingDirectory().toURI().toString(), getUri());
}
```
这其中又由 SearcPathLocator 接口的 getLocations() 来实际加载内容,这个方法就由具体的实现类来完成。
Spring Cloud Config Server 内部除了 JGitEnvironmentRepository 实现外还有另外三种实现。

- SvnKitEnvironmentRepository
- NativeEnvironmentRepository
- MultipleJGitEnvironmentRepository

见名知其意，分别是 svn 本地目录以及多 git 仓库的实现，具体内容就不细说，大家有兴趣可自行研究。

### 客户端加载原理 

项目只需要加载依赖就可以使用远程配置，现有项目迁移过去也不用改动任何代码，刚使用的时候感觉真是爽快。
达到这样的效果背后依赖于 SpringMVC 自身的优秀的抽象实现以及 Springboot 的实用的自动配置类加载。
```
@Configuration
public class ConfigClientAutoConfiguration {

	@Bean
	public ConfigClientProperties configClientProperties(
			Environment environment,
			ApplicationContext context) {...}
	...
	@Bean
	public ConfigServerHealthIndicator configServerHealthIndicator(
			ConfigServicePropertySourceLocator locator,
			ConfigClientHealthProperties properties, Environment environment) {
		return new ConfigServerHealthIndicator(locator, environment, properties);
	}
	...
}
```
无关的逻辑部分在这里就省去了，接着看。   
**ConfigClientAutoConfiguration** 这个是 ConfigClient 的自动配置类,加载时触发新建一个 **ConfigServicePropertySourceLocator** 的 bean。
```
@Order(0)
public class ConfigServicePropertySourceLocator 
		implements PropertySourceLocator {
			
	private RestTemplate restTemplate;
	private ConfigClientProperties defaultProperties;
	...
	@Override
	@Retryable(interceptor = "configServerRetryInterceptor")
	public org.springframework.core.env.PropertySource<?> locate(
			org.springframework.core.env.Environment environment) {
		...
			// Try all the labels until one works
			for (String label : labels) {
				Environment result = getRemoteEnvironment(restTemplate,
						properties, label.trim(), state);
						...
}
```

然后在 ApplicationContextInitializer 的 initialize() 方法中被调用 locate() 方法从 ConfigServer 拉取配置文件，注入到 **ConfigurableEnvironment** 里。

```
@Configuration
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {
	...
	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		...
		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		for (PropertySourceLocator locator : this.propertySourceLocators) {
			PropertySource<?> source = null;
			source = locator.locate(environment);
			if (source == null) {
				continue;
			}
			...
		}
		...
	}
}
```

接下来才进入应用的初始化过程。
从以上过程可以看到，其他任何应用需要使用 ConfigServer 时也只需要在应用初始化之前通过  HTTP 请求拉取到配置文件即可。


## 小结 
了解了大致原理，便可以捋一捋集中式管理的优点。
 ![图片](https://dn-coding-net-production-pp.qbox.me/9e2bc064-cf1a-4204-aa7b-6af434916f40.png) 

 - 没有记忆负担：大家都使用一个仓库保存所有配置文件，方便查看。

 - 降低冲突沟通成本：开发新功能是凡是有配置变动均向仓库的提交，在合并后其他人碰到配置问题时也不用到处找人问，看看仓库历史就知道什么发生了变化。   

- 方便测试：测试环境下，测试人员也不用老是问开发环境配置的问题了，所有配置一目了然，更改记录也都有提交说明，有问题向开发反馈也能帮助快速定位问题范围。   

- 方便管理： 在模拟环境和生产环境下，由运维人员管理线上配置文件，只要设置好分支保护和权限分配。



