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
