# Spring Boot 自动配置原理（Spring Boot 3.2+）

> ✅ 适用于 Spring Boot 3.2 及以上版本（包括 3.5.7）
>  📌 核心思想：**约定优于配置 + 条件化装配**

------

## 一、触发入口

- 由 `@SpringBootApplication` 触发：

  ```
  @SpringBootApplication
  public class MyApp { ... }
  ```

- 等价于：

  ```
  @Configuration
  @EnableAutoConfiguration   // ← 关键注解
  @ComponentScan
  ```

- `@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector`，启动自动配置流程。

------

## 二、自动配置加载流程（4 步）

### 1️⃣ 发现候选自动配置类

- **机制**：`ImportCandidates.load(AutoConfiguration.class, classLoader)`
- 数据来源：
  - 编译期生成的内部索引（JAR 内部优化结构）
  - **不再依赖** `spring.factories` 或 `.imports` 文件
- **输出**：全量自动配置类名列表（如 `DataSourceAutoConfiguration`）

>  此阶段 **不使用** `AnnotationMetadata` 和 `AnnotationAttributes`，因此 `getCandidateConfigurations()` 方法参数未被使用。

------

### 2️⃣ 应用排除规则（exclude）

- 检查主类上的排除配置： 

  ```
  @SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
  @SpringBootApplication(excludeName = "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration")
  ```

- 使用 `AnnotationAttributes attributes` 获取排除列表

- 从候选列表中移除被排除的类

------

### 3️⃣ 条件过滤（Conditional Filtering）

对每个候选类评估条件注解，例如：

```
@ConditionalOnClass(JdbcTemplate.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(prefix = "my.feature", name = "enabled", havingValue = "true")
```

#### ⚡ 性能优化：`spring-autoconfigure-metadata.properties`

- 路径：`META-INF/spring-autoconfigure-metadata.properties`

- 作用： 

  - 提供类存在性、依赖顺序等元数据
  - 避免反射加载类，提升启动速度

- 示例： 

  ```
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration=\
    ConditionalOnClass:javax.sql.DataSource,com.zaxxer.hikari.HikariDataSource
  ```

> ✅ 该文件 **不用于注册自动配置类**，仅用于条件评估和 IDE 提示。

------

### 4️⃣ 排序与注册

- 根据 `@AutoConfigureBefore` / `@AutoConfigureAfter` 排序
- 将最终列表作为 `@Configuration` 类注册到 Spring 容器

------

## 三、关键文件演进

| Spring Boot 版本 | 注册方式                  | 文件位置                                                     |
| ---------------- | ------------------------- | ------------------------------------------------------------ |
| 2.x              | `spring.factories`        | `META-INF/spring.factories`                                  |
| 3.0 – 3.1        | `.imports` 文件           | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| **3.2+**         | **编译期生成 + 内部索引** | ❌ 无显式注册文件 ✅ 有 `META-INF/spring-autoconfigure-metadata.properties`（仅元数据） |

> 🔔 注意：源码中仍保留 `"*.imports"` 字符串（用于错误提示或兼容第三方），但 **Spring Boot 自身已不再读取该文件**。

------

## 四、核心组件

| 组件                                       | 作用                                              |
| ------------------------------------------ | ------------------------------------------------- |
| `@EnableAutoConfiguration`                 | 启用自动配置的入口                                |
| `AutoConfigurationImportSelector`          | 核心选择器，负责加载、过滤、排序                  |
| `ImportCandidates`                         | Spring Boot 3.2+ 新增，封装自动配置发现逻辑       |
| `spring-autoconfigure-metadata.properties` | 提供条件元数据，优化启动性能                      |
| `@AutoConfiguration`                       | 推荐用于自定义自动配置类（替代 `@Configuration`） |

------

## 五、验证自动配置是否生效

### 方法 1：启用调试模式

```
# application.yml
debug: true
```

启动时输出 **Conditions Evaluation Report**，显示匹配/不匹配的自动配置。

### 方法 2：打印容器中的自动配置 Bean

```
@Autowired
private ApplicationContext ctx;

@PostConstruct
public void printAutoConfigBeans() {
    Arrays.stream(ctx.getBeanDefinitionNames())
          .filter(name -> name.contains("AutoConfiguration"))
          .forEach(System.out::println);
}
```

------

## 六、开发自定义 Starter 的最佳实践

1. 使用 `@AutoConfiguration` 注解（而非 `@Configuration`）

2. 添加条件注解（如 `@ConditionalOnClass`）

3. 引入注解处理器以生成元数据： 

   ```
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-configuration-processor</artifactId>
       <optional>true</optional>
   </dependency>
   ```

4. 构建时自动生成 `spring-autoconfigure-metadata.properties`

5. **无需手动维护** `spring.factories` 或 `.imports` 文件

------

## 七、流程图

```
@SpringBootApplication
        ↓
@EnableAutoConfiguration
        ↓
AutoConfigurationImportSelector
        ↓
ImportCandidates → 加载全量自动配置类（无参数）
        ↓
排除（exclude） → 使用 AnnotationAttributes
        ↓
条件过滤 → 使用 spring-autoconfigure-metadata.properties + 反射兜底
        ↓
排序（@AutoConfigureBefore/After）
        ↓
注册到 Spring 容器
```

------

## 八、常见误区

- ❌ “没有 `.imports` 文件说明自动配置坏了”
   → ✅ Spring Boot 3.2+ **本就不需要它**
- ❌ “必须手动写 `spring.factories`”
   → ✅ 已废弃，新项目不要用
- ✅ `spring-autoconfigure-metadata.properties` 是 **性能关键**，不是可有可无

------

>  **建议**：遇到自动配置不生效时，优先查看 `debug: true` 输出的条件报告，而不是检查文件是否存在。