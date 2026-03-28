public class User {
    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)  // 只写不读
    private String password;  // 可接收但不能返回
    @JsonProperty(access = JsonProperty.Access.READ_ONLY)  // 只读不写
    private Date createTime;  // 可返回但不能接收
} 
entity直接当做入参出参，省去了Vo，Dto

JPA返回部分实体字段的映射可以用record，方便返回给controller

