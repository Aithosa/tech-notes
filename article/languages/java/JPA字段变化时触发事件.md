# JPA 字段变化时触发事件

## JPA 部分

表的定义和方法就是 JPA 平时的用法。

```Java
// UserRepository
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
}
```

表和字段的定义并无不同，如下:

```Java
// User
@Data
@RequiredArgsConstructor
@Entity
@Table(name = "t_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Long id;

    private String firstName;

    private String lastName;

    private Integer age;

    private boolean status;

    /**
     * 监听event有变化时处理事件
     */
    @Transient
    @Getter(onMethod_={@DomainEvents})
    //@Getter(onMethod = @__(@DomainEvents))
    private List<Object> events = new ArrayList<>();

//    //该方法会在userRepository.save()调用时被触发调用
//    @DomainEvents
//    Collection<UserSaveEvent> domainEvents() {
//        return Arrays.asList(new UserSaveEvent(this.id));
//    }

    public void setStatus(boolean status) {
        events.add(new UserSaveEvent(this.id));
        this.status = status;
    }

    /**
     * 事件发布后清除
     */
    @AfterDomainEventPublication
    public void clearEvents() {
        this.events.clear();
    }
}
```

## 事件的监听

这里的例子是监听某个字段的变化，当有变化时会触发某个时间，所以需要拦截这个字段的变化。在这个例子里监听的是`status`字段的变化，因此需要重写`status`字段的`set`方法，当调用时顺便记录一个事件。

事件列表定义为表里的一个字段(`List<Object>`)，每次修改`status`都往里边加上一个事件。需要注意的是这个字段需要加上`@Getter(onMethod_={@DomainEvents})`注解。

事件定义为`Object`理论上可以支持针对监听字段的多种事件。上面定义的是`UserSaveEvent`事件。

```Java
@Data
@AllArgsConstructor
public class UserSaveEvent {
    private Long id;
}
```

同时，事件发布后需要清除该事件，定义一个`clearEvents()`方法，把列表清空，需要加上`@AfterDomainEventPublication`注解。

## 事件的执行

某个类事件可以放在 Service 中，唯一需要注意的是在该事件的执行方法加上`@TransactionalEventListener`注解。

```Java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    //接受User发出的类型为UserSaveEvent的DomainEvents事件
     // @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @TransactionalEventListener
    public void event(UserSaveEvent event){
        System.out.println("Event has been created.");
        System.out.println(userRepository.getOne(event.getId()));
    }
}
```

## 其他问题

以上监听的是字段调用`set`后触发的事件，但是可能调用了 set 后数据保存不成功，就需要撤销该事件，或者批量保存的时候某一个数据有问题，需要撤销该事件。

关注点：

- 能否之撤销队列里的某个事件
- 事件队列什么时候执行？

经验证，时间会在执行`save`或者`saveAll`时被触发。

## 参考

https://blog.csdn.net/f4761/article/details/84622317
