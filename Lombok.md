# Lombok

~~~java
Project Lombok is a java library that automatically plugs into your editor and build tools, spicing up your java.
Never write another getter or equals method again, with one annotation your class has a fully featured builder, Automate your logging variables, and much more.
~~~

- java library
- plugins
- build tools
- one annotation

~~~java
@Getter and @Setter：可以放在类上或字段上，分别为所有属性和单个属性生成对应方法
@FieldNameConstants
@ToString：生成toString方法
@EqualsAndHashCode
@AllArgsConstructor, @RequiredArgsConstructor and @NoArgsConstructor
@Log, @Log4j, @Log4j2, @Slf4j, @XSlf4j, @CommonsLog, @JBossLog, @Flogger, @CustomLog
@Data(应用在类上)：生成无参构造、get、set、toString、hashcode、equals方法
@Builder
@SuperBuilder
@Singular
@Jacksonized
@Delegate
@Value
@Accessors
@Tolerate
@Wither
@With
@SneakyThrows
@StandardException
@val
@var
experimental @var
@UtilityClass
~~~

缺点：

- 不便于判断设置属性和获取属性的逻辑（额外的校验）
- 可读性差