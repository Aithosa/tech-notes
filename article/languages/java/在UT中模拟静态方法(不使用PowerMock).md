# 在 UT 中模拟静态方法(不使用 PowerMock)

首先引入依赖,用`mockito-inline`替换`mockito-core`依赖(看`mockito-inline`的`pom.xml`文件可知，其内部依赖了`mockito-core`)

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>3.7.7</version>
    <scope>test</scope>
</dependency>
```

假如有个 SysDataService 类, 里面都是静态的工具方法

```java
public class SysDataService {
    /**
     * 校验配置中的机场是否都归属于对应城市
     */
    public static boolean airportsBelongToCity(String cityCode, List<String> airportCodeList) {
        return airportCodeList.stream()
            .allMatch(airportCode -> cityCode.equals(SysDataService.getAirportCity(airportCode)));
    }
}
```

在 UT 中如果想要 Mock，可以使用：

```java
MockedStatic<SysDataService> theMock = Mockito.mockStatic(SysDataService.class);
theMock.when(() -> SysDataService.airportsBelongToCity(anyString(), anyList())).thenReturn(true);
```

另外在使用结束后需要执行

```java
theMock.close();
```

因为 mock 的是整个类，里面的其他静态方法也会受到这次影响，其他用到这些静态方法的 ut 可能会报错，所以需要及时关闭。

关联报错

```text
static mocking is already registered in the current thread ，To create a new mock, the existing static mock registration must be deregistered
```

如果可能会在连续多个 ut 中使用可以设置为

```java
private static final MockedStatic<SysDataService> theMock = Mockito.mockStatic(SysDataService.class);
```

方便最后统一关闭。

注意：如果在同一个函数里还调用了模拟的静态方法所属的类里的其他方法处理过的数据，会导致模拟失败。
比如,模拟的类.模拟的静态方法(参数 1.1,参数 2);其中 参数 1.1 = 模拟的类.其他方法(参数 1)，这样模拟就会失效。
