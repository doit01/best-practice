public class User {
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)  // 只写不读
    private String password;  // 可接收但不能返回
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)  // 只读不写
    private Date createTime;  // 可返回但不能接收
} 
entity直接当做入参出参，省去了Vo，Dto

JPA返回部分实体字段的映射可以用record，方便返回给controller
=合理使用@ManyToMany @OneToMany====================================================================================================================
@Entity
@Table(name = "sys_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    
    // ✅ 使用Set避免重复
    // ✅ 延迟加载
    // ✅ 单向关联（Role不引用User）
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_role_rel",  // 明确指定中间表名
        joinColumns = @JoinColumn(name = "user_id", 
                       foreignKey = @ForeignKey(name = "fk_user_role_user")),
        inverseJoinColumns = @JoinColumn(name = "role_id",
                               foreignKey = @ForeignKey(name = "fk_user_role_role"))
    )
    private Set<Role> roles = new HashSet<>();  // 用Set！

    或者简化写法
    @ManyToManyy(fetch = FetchType.LAZY)  // 必须懒加载
private Set<Role> roles = new HashSet<>();
自动生成中间表名：user_roles（实体名_属性名）
自动生成外键列名：user_id、roles_id
自动生成外键约束名（随机字符串）


    
    // ✅ 添加移除的辅助方法
    public void addRole(Role role) {
        roles.add(role);
    }
    
    public void removeRole(Role role) {
        roles.remove(role);
    }
    
    // ❌ 不要提供setter，用上面方法控制
    // public void setRoles(Set<Role> roles) { ... }
}

@Entity
@Table(name = "sys_role")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String code;
    
    // ✅ 单向，Role不维护关系
    // 不需要@ManyToMany映射到User
}

===========================================================================================================================================================
N+1 问题
List<User> users = userRepository.findAll(); // 执行 1 条 SQL: select * from user  10条记录

for (User user : users) {
    // 每次循环，因为 orders 是懒加载且未被初始化，JPA 会执行一条 SQL 去查该用户的订单
    System.out.println(user.getOrders().size()); 
}
循环出来10次的懒加载查询。所以一共11次查询。

解决
使用JOIN FETCH（最常用）
java
下载
复制
// Repository中定义
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 方法1：使用@Query
    @Query("SELECT DISTINCT u FROM User u " +
           "LEFT JOIN FETCH u.orders o")
    List<User> findAllWithOrders();
    
    // 方法2：使用@EntityGraph
    @EntityGraph(attributePaths = {"orders"})
    @Query("SELECT u FROM User u")
    List<User> findAllWithOrdersGraph();
}

执行SQL：

sql
复制
-- 只有1次查询！
SELECT u.*, o.* 
FROM user u 
LEFT JOIN order o ON u.id = o.user_id



使用DTO投影
java
下载
复制
// 只查询需要的字段
public interface UserWithOrderCount {
    Long getId();
    String getName();
    Long getOrderCount();  // 使用子查询
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u.id as id, u.name as name, " +
           "(SELECT COUNT(o.id) FROM Order o WHERE o.user = u) as orderCount " +
           "FROM User u")
    List<UserWithOrderCount> findAllWithOrderCount();
}


手册的原话是“不得使用外键与级联”，这里的“外键”特指在数据库 DDL 中通过 FOREIGN KEY ... REFERENCES定义的物理约束。
维度
数据库物理外键 (被禁)
JPA @OneToMany (可用)
定义位置​
数据库表结构 DDL
Java 实体类代码
作用​
强制数据一致性 (DB 级)
描述对象间的关系 (ORM 级)
性能影响​
写操作有锁和检查开销
仅影响 ORM 查询策略
本质​

数据库引擎的强制规则
应用程序的映射元数据

手册的初衷：在高并发、分布式（分库分表）场景下，数据库层面的外键检查会带来严重的锁竞争和性能损耗，且无法跨库生效。因此要求将“数据一致性”的逻辑上移到应用层代码控制，而不是依赖数据库。

✅ @OneToMany 的正确使用姿势（不违反规约）

你完全可以在不创建数据库外键的情况下，正常使用 @OneToMany。这才是符合阿里规约的高级玩法。

1. 代码定义关联，数据库不建约束
java
下载
复制
@Entity
public class Order {
    @Id
    private Long id;
    
    // 定义业务逻辑上的“外键字段”，但数据库不建约束
    @Column(name = "user_id")
    private Long userId;  // 逻辑外键，非物理外键
}

@Entity
public class User {
    @Id
    private Long id;
    
    // JPA 依然可以通过 userId 字段建立对象映射
    @OneToMany
    @JoinColumn(name = "user_id", referencedColumnName = "id", 
                foreignKey = @ForeignKey(ConstraintMode.NO_CONSTRAINT)) // 关键：禁止生成外键约束
    private List<Order> orders;
}

效果：Java 代码知道 User和 Order是一对多关系，方便你写 JOIN FETCH查询；但数据库层面 user_id只是一个普通字段，没有 FOREIGN KEY约束，完全符合手册要求。



spring-boot-starter-validation=============================================================================================================================
用implementation 'org.springframework.boot:spring-boot-starter-validation'
不要直接使用Hibernate Validator
// Spring 自动配置，开箱即用
@RestController
public class UserController {
    @PostMapping
    public User create(@Valid @RequestBody User user) {
        // 校验自动触发，无需手动调用
        return userService.save(user);
    }
}
✅ 自动配置：Spring Boot 自动创建 Validator Bean
✅ 无缝集成：与 Spring MVC、Spring Data、Spring Security 深度集成
✅ 声明式校验：通过 @Valid 自动触发，无需手动调用
✅ 统一异常处理：Spring 自动抛出 MethodArgumentNotValidException
✅ 方法级别校验：支持 @Validated 注解的 AOP 切面校验
// 直接注入使用
@Autowired
private Validator validator; // 已经是配置好的 Hibernate Validator
public void someMethod(Object obj) {
    Set<ConstraintViolation<Object>> violations = validator.validate(obj);// 如果需要手动校验，也可以随时使用
}
Spring MVC 的 RequestResponseBodyMethodProcessor 会自动识别 @Valid 注解并执行校验，失败时自动抛出异常
方法级别校验（AOP）
java
@Service
@Validated // Spring 会为所有 public 方法创建 AOP 代理进行校验
public class UserService {
    public User createUser(@Valid User user) {
        // 方法参数校验自动生效
        return userRepository.save(user);
    }
}
消息国际化支持
Spring Boot 自动配置了 MessageSource，可以无缝集成国际化错误消息：
yaml
# application.yml
spring:
  messages:
    basename: ValidationMessages
properties
# ValidationMessages_zh_CN.properties
javax.validation.constraints.NotBlank.message=用户姓名不能为空


@RestController
@Validated
public class UserController {
    
    @PostMapping("/users")
    public User createUser(@Valid @RequestBody User user) {
        // Spring Boot starter 自动处理：
        // 1. 参数绑定
        // 2. 参数校验（使用 Hibernate Validator）
        // 3. 校验失败自动返回 400 错误
        return userService.create(user);
    }
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable @Min(1) Long id) {
        // 方法级别校验（需要 @Validated）
        return userService.getById(id);
    }
}
功能对比表
使用场景	@Valid	@Validated
@RequestBody 校验	✅ 推荐	❌ 不支持嵌套
@RequestParam 校验	❌ 不支持	✅ 需要类上加
@PathVariable 校验	❌ 不支持	✅ 需要类上加
嵌套对象校验	✅ 支持	❌ 不支持
分组校验	❌ 不支持	✅ 支持
Service 层方法参数校验	❌ 不支持	✅ 需要类上加
Service 层返回值校验	❌ 不支持	✅ 需要类上加
标准规范	✅ Jakarta	❌ Spring 特有
6. 实际开发中的选择建议
java
// 场景1: Controller 层接收 JSON 请求体（最常见）
@PostMapping("/users")
public User create(@Valid @RequestBody UserDTO dto) {  // 用 @Valid
    // 如果 UserDTO 有嵌套对象，@Valid 会级联校验
    return userService.create(dto);
}

// 场景2: Controller 层有路径变量或查询参数
@RestController
@Validated  // 类上加 @Validated
public class UserController {
    
    @GetMapping("/users/{id}")
    public User getById(@PathVariable @Min(1) Long id) {  // 方法参数校验
        return userService.getById(id);
    }
    
    @GetMapping("/users")
    public List<User> list(@RequestParam @Min(0) Integer page,
                          @RequestParam @Min(1) Integer size) {
        return userService.list(page, size);
    }
}

// 场景3: 同一个 DTO 不同操作不同校验规则
@PostMapping("/users")
public User create(@Validated(Create.class) @RequestBody UserDTO dto) {  // 用 @Validated 分组
    return userService.create(dto);
}

@PutMapping("/users/{id}")
public User update(@Validated(Update.class) @RequestBody UserDTO dto) {
    return userService.update(dto);
}

// 场景4: Service 层方法保护
@Service
@Validated  // 类上加 @Validated
public class UserService {
    
    public User create(@Valid UserDTO dto) {  // 参数自动校验
        return userRepository.save(dto);
    }
    
    @NotNull  // 返回值校验
    public User findById(@Min(1) Long id) {  // 参数校验
        return userRepository.findById(id).orElse(null);
    }
}
总结
记忆口诀：

@Valid：标准规范，用于嵌套校验，推荐在 Controller 的 @RequestBody 使用

@Validated：Spring 增强，用于分组校验和方法级别校验（参数、返回值），需要在类上声明

最佳实践：

Controller 的 @RequestBody 参数：用 @Valid

Controller 的 @RequestParam/@PathVariable：配合类上的 @Validated 使用

同一个 DTO 不同场景不同规则：用 @Validated 分组

Service 层方法保护：类上加 @Validated，方法参数加 @Valid

嵌套对象：必须在父对象字段上加 @Valid（无论外层用哪个注解）
