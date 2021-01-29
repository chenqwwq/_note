## 以Consul作为配置中心



1. PropertySourceLocator

   用于定位具体的资源位置，每个配置中心都可以继承该类，来获取自己的PropertySourceLocator

2. ConsulPropertySourceLocator

   consul-config用来定位consul中的配置信息的实现类，使用的目录如下：

   - ${spring.cloud.config.consul.prefix}/application
   - ${spring.cloud.config.consul.prefix}/${spring.application.name}

   以上两种再加上分别遍历profiles的结果类似:

   ${spring.cloud.config.consul.prefix}/application,dev/

3. PropertySourceBootstrapConfiguration

   继承ApplicationContextInitializer，在SpringBoot的启动过程中准备ApplicationContext的时候调用，通过SPI的方式引入。

   

