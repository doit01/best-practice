public class User {
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)  // 只写不读
    private String password;  // 可接收但不能返回
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)  // 只读不写
    private Date createTime;  // 可返回但不能接收
} 
entity直接当做入参出参，省去了Vo，Dto

JPA返回部分实体字段的映射可以用record，方便返回给controller

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


N+1 问题

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
