# Netflix Archaius

- GitHub 仓库：https://github.com/Netflix/archaius
- wiki：https://github.com/Netflix/archaius/wiki

```xml
<dependency>
  <groupId>com.netflix.archaius</groupId>
  <artifactId>archaius-core</artifactId>
  <version>0.6.0</version>
 </dependency>
```

## 概述

Netflix Archaius 是 Apache 公共配置库的扩展，是一个功能强大的配置管理库，添加了一些很好的功能，如动态源的轮询框架，具有高吞吐量和线程安全的实现。Archaius 可用于从许多不同动态源收集配置属性的框架，提供对配置信息的快速及线程安全访问。除此之外，Archaius 还允许属性在运行时动态更改，使系统无需重新启动应用程序即可获得这些变化。

Archaius 提供了一些其他任何配置框架都没有考虑过的方便有趣的功能。其中的一些关键点是：

- 动态和类型属性
- 在属性改变时调用的回调机制
- 动态配置源（如 URL，JDBC 和 Amazon DynamoDB）的实现
- Spring Boot Actuator 或 JConsole 可以访问的 JMX MBean，用于检查和操作属性
- 动态属性验证

## 原理

Archaius 定义了一个复合配置，可以从不同来源获得的各种配置的集合。此外，其中一些配置源可以支持在运行时轮询更改。Archaius 提供接口和一些预定义的实现来配置这些类型的源。

由于源集合是分层的，因此如果属性存在于多个配置中，则最终值将是最顶部插槽中的值。

最后， `ConfigurationManager` 处理系统范围的配置和部署上下文。它可以安装最终的复合配置，或检索已安装的复合配置进行修改。

Spring Cloud 开发了一个库，可以轻松配置`Spring Environment Bridge`，以便 Archaius 可以从`Spring Environment`中读取属性。

Spring Cloud Archaius 库的主要任务是将所有不同的配置源合并为 `ConcurrentCompositeConfiguration`，并使用`ConfigurationManager`进行安装 。

库定义源的优先顺序是：

- 上下文中定义的任何 Apache 公共配置 `AbstractConfiguration` Bean
- `Autowired Spring ConfigurableEnvironment` 中定义的所有源代码
- 默认的 Archaius 源，我们在上面的例子中看到过
- Apache 的 `SystemConfiguration` 和 `EnvironmentConfiguration` 源

Spring Cloud 库提供的另一个有用功能是定义一个 `Actuator Endpoint` 来监控属性并与之交互。

## 简单使用

默认情况下，它动态管理应用程序类路径中名为`config.properties`的文件中定义的所有属性。请注意，不需要告诉 Archaius 在哪里找到您的属性文件，因为他要查找的默认名称是`config.properties`。

```Java
public class ApplicationConfig {

  public String getStringProperty(String key, String defaultValue) {
    final DynamicStringProperty property = DynamicPropertyFactory.getInstance().getStringProperty(key,
        defaultValue);
    return property.get();
  }

  public String getStringPropertyWithCallback(String key, String defaultValue) {
    final DynamicStringProperty property = DynamicPropertyFactory.getInstance().getStringProperty(key,
        defaultValue);

    // 创建callback
    prop.addCallback(new Runnable() {
      public void run() {
        // ...
    }

    return property.get();
  }
}
```

此时更改类路径文件中属性的值，无需重新启动服务。在一分钟左右之后，对参数的查询应检索出新值。

如果需要指定不同配置文件名，可以通过配置变量设置，告诉 Archaius 在哪里查找此文件。
更改系统属性`archaius.configurationSource.defaultFileName`，在启动应用程序时将其作为参数传递给 vm。

如设置启动参数：

```bash
java ... -Darchaius.configurationSource.defaultFileName=customName.properties
```

或者在代码里设置：

```Java
public class ApplicationConfig {
  static {
    System.setProperty("archaius.configurationSource.defaultFileName", "customConfig.properties");
  }

  public String getStringProperty(String key, String defaultValue) {
    final DynamicStringProperty property = DynamicPropertyFactory.getInstance().getStringProperty(key,
        defaultValue);
    return property.get();
  }
}
```

如果需要读几个属性文件，可以从首先加载的默认文件开始，轻松定义属性文件链及其加载顺序。可以使用键`@next=nextFile.properties`指定一个特殊属性来告诉 Archaius 哪个是应该加载的下一个文件，并将相应的`secondConfig.properties`添加到我们的 `resources` 文件夹中。
。
如在`customConfig.properties`文件中添加以下行：

```properties
@next=secondConfig.properties
```

## 调整和扩展 Archaius 配置

### Archaius 支持的配置属性

如果我们希望 Archaius 考虑类似于`config.properties`的其他配置文件 ，我们可以定义 `archaius.configurationSource.additionalUrls`系统属性。

该值被解析为由逗号分隔的 URL 列表，例如，我们可以在启动应用程序时添加此系统属性：

```bash
-Darchaius.configurationSource.additionalUrls="classpath:other-dir/extra.properties,file:///home/user/other-extra.properties"
```

Archaius 将首先读取`config.properties`文件，然后按指定的顺序读取其他文件。因此，后面文件中定义的属性将优先于先前的属性。

我们可以使用几个其他系统属性来配置 Archaius 默认配置的各个方面：

- `archaius.configurationSource.defaultFileName`：类路径中的默认配置文件名，默认值 config.properties
- `archaius.fixedDelayPollingScheduler.initialDelayMills`：读取配置源之前的初始延迟，默认 30000
- `archaius.fixedDelayPollingScheduler.delayMills`：两次读取源之间的延迟; 默认值为 1 分钟（60000）

### 使用 Spring 添加其他配置源

#### 来自配置文件

TODO: 这里解释的不清楚

如何添加一个不同的配置源交由框架管理？我们如何管理优先级高于 Spring 环境中定义的动态属性？

回顾之前提到的内容，我们可以发现 Spring 定义的`Composite Configuration`中的最高配置是在上下文中定义的 `AbstractConfiguration` bean。

因此，我们需要做的就是使用 Archaius 提供的一些功能将这个 Apache 的抽象类的实现添加到我们的`Spring Context`中，Spring 的自动配置会自动将它添加到托管配置属性中。

简单起见，我们将看到一个示例，我们配置一个类似于默认`config.properties`的属性文件，但其优先级高于 Spring 环境和应用程序属性的其余部分：

```Java
@Configuration
public class ApplicationPropertiesConfigurations {
    @Bean
    public AbstractConfiguration addApplicationPropertiesSource() {
        // 添加该该配置源给Spring管理，并且优先级最高，支持刷新
        PolledConfigurationSource source = new URLConfigurationSource("classpath:other-config.properties");
        AbstractPollingScheduler scheduler = new FixedDelayPollingScheduler()
        return new DynamicConfiguration(source, scheduler);
    }
}
```

这里其实可以自定义配置并注册您的配置：

```Java
PolledConfigurationSource source = createMyOwnSource();
AbstractPollingScheduler scheduler = createMyOwnScheduler();
DynamicConfiguration configuration = new DynamicConfiguration(source, scheduler);
ConfigurationManager.install(configuration);
```

多配置源管理：

```Java
import com.netflix.config.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MultipleConfigSourcesConfiguration {

    @Bean
    public AbstractConfiguration configurationSource() {
        // 创建一个组合配置
        ConcurrentCompositeConfiguration config = new ConcurrentCompositeConfiguration();

        // 添加第一个配置源：属性文件
        DynamicPropertyFactory.getInstance();
        DynamicURLConfiguration urlConfig = new DynamicURLConfiguration();
        config.addConfiguration(urlConfig, "urlConfig");

        // 添加第二个配置源：系统属性
        SystemConfiguration sysConfig = new SystemConfiguration();
        config.addConfiguration(sysConfig, "systemConfig");

        // 添加第三个配置源：环境变量
        EnvironmentConfiguration envConfig = new EnvironmentConfiguration();
        config.addConfiguration(envConfig, "environmentConfig");

        // 添加第四个配置源：自定义属性文件
        ClasspathPropertiesConfiguration customConfig = new ClasspathPropertiesConfiguration("custom-config.properties");
        config.addConfiguration(customConfig, "customConfig");

        return config;
    }
}
```

这个例子中，我们添加了四个不同的配置源：

1. `DynamicURLConfiguration`: 这是默认的配置源，通常用于加载 `config.properties` 文件。
2. `SystemConfiguration`: 这个配置源使用系统属性。
3. `EnvironmentConfiguration`: 这个配置源使用环境变量。
4. `ClasspathPropertiesConfiguration`: 这个配置源从 `classpath` 中加载一个自定义的属性文件。

这些配置源被添加到一个 `ConcurrentCompositeConfiguration` 对象中，它会按照添加的顺序来查找属性。
这意味着如果多个配置源中存在同名的属性，先添加的配置源中的属性会优先使用。

要使用这个配置，你只需要在你的主应用类中导入这个配置类：

```Java
@SpringBootApplication
@Import(MultipleConfigSourcesConfiguration.class)
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

这样，Spring 就会使用你定义的多个配置源来管理应用程序的配置。

#### 来自数据库

TODO：https://www.jianshu.com/p/f9d7500920a9 后半部分
