# Spring Boot 如何打包成可执行 JAR

## 1. 原理与产物结构
Spring Boot 通过 `spring-boot-maven-plugin` 或 `bootJar` 将依赖打包到可执行 JAR 中，核心是内置启动器与分层打包结构。最终产物包含 `BOOT-INF/classes` 与 `BOOT-INF/lib`，运行时由 `JarLauncher` 加载依赖。这样可以做到单文件部署与 `java -jar` 直接启动。

## 2. Maven 打包方式
Maven 项目常用如下插件配置，`repackage` 目标会把依赖重新封装进产物。

```xml
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>repackage</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

```shell
mvn -DskipTests package
java -jar target/app.jar
```

## 3. Gradle 打包方式
Gradle 项目通常使用 `bootJar` 任务生成可执行 JAR。

```gradle
bootJar {
  archiveFileName = "app.jar"
}
```

```shell
gradle bootJar
java -jar build/libs/app.jar
```

## 4. 常见坑与注意事项
打包失败或运行报错时通常与依赖或打包方式有关。

- `provided` 或 `optional` 依赖不会被打入产物，导致运行时报类缺失。
- 多模块项目只在启动模块应用插件，避免重复打包或依赖冲突。
- 需要外置容器部署时改用 `war`，并继承 `SpringBootServletInitializer`。
- 依赖冲突可通过 `dependencyManagement` 与排除规则解决，再重新打包验证。
