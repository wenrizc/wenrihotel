## 1. 核心概念

### 1.1 什么是 Maven 生命周期

`Maven` 的核心思想是把一次构建过程抽象成一条**标准化流水线**。官方称它为 `build lifecycle`，也就是从校验、编译、测试、打包到发布的一组固定阶段。

这样做的价值在于：开发者不需要为每个项目重新发明构建脚本，只需要记住少量命令，再由 `pom.xml` 告诉 `Maven` 当前项目在每个阶段该执行什么任务。

### 1.2 生命周期、阶段、目标三者的区别

| 概念 | 粒度 | 作用 | 典型例子 |
| --- | --- | --- | --- |
| `lifecycle` | 最大 | 定义一整条构建流程 | `default`、`clean`、`site` |
| `phase` | 中等 | 生命周期中的一个阶段 | `compile`、`test`、`package` |
| `goal` | 最小 | 插件真正执行的具体任务 | `compiler:compile`、`jar:jar` |

可以把它理解成：

- `lifecycle` 是生产线。
- `phase` 是生产线上的工位。
- `goal` 是工位里真正干活的动作。

很多人把 `package` 当成“命令”，但从 `Maven` 语义上说，它首先是一个 `phase`。真正执行打包动作的，通常是某个插件绑定到这个阶段上的 `goal`，例如 `jar:jar` 或 `war:war`。

## 2. Maven 内置的三套生命周期

### 2.1 `default` 生命周期

`default` 是最重要的一套生命周期，负责项目从源码到最终制品的构建与发布。平时执行的 `compile`、`test`、`package`、`install`、`deploy`，本质上都属于这条生命周期。

绝大多数面试问题中的“`Maven` 生命周期”，默认都是在问 `default` 生命周期。

### 2.2 `clean` 生命周期

`clean` 生命周期的职责很单一：**清理上一次构建产生的输出物**，例如 `target` 目录。它只有三步：

1. `pre-clean`
2. `clean`
3. `post-clean`

因此，`mvn clean package` 不是在一条生命周期里执行两个阶段，而是**先执行 `clean` 生命周期，再执行 `default` 生命周期直到 `package` 阶段**。

### 2.3 `site` 生命周期

`site` 生命周期用于生成项目站点文档，例如测试报告、插件报告、项目描述页等。它在现代业务项目里使用频率不如 `default` 和 `clean` 高，但仍是 `Maven` 官方内置生命周期之一。

它的常见阶段包括：

1. `pre-site`
2. `site`
3. `post-site`
4. `site-deploy`

## 3. `default` 生命周期的执行链路

### 3.1 关键阶段概览

下面这些阶段是面试中最常被问到的主干阶段：

| 阶段 | 作用 | 面试要点 |
| --- | --- | --- |
| `validate` | 校验项目是否完整、配置是否有效 | 还不做编译 |
| `compile` | 编译主源码 | 产物进入 `target/classes` |
| `test` | 执行单元测试 | 通常不依赖最终包 |
| `package` | 打成可分发制品 | 常见是 `jar`、`war` |
| `verify` | 做集成测试结果校验或质量校验 | 比 `package` 更完整 |
| `install` | 安装到本地仓库 | 供本机其他项目依赖 |
| `deploy` | 发布到远程仓库 | 供团队或环境共享 |

### 3.2 完整阶段顺序

`default` 生命周期的完整顺序如下：

1. `validate`
2. `initialize`
3. `generate-sources`
4. `process-sources`
5. `generate-resources`
6. `process-resources`
7. `compile`
8. `process-classes`
9. `generate-test-sources`
10. `process-test-sources`
11. `generate-test-resources`
12. `process-test-resources`
13. `test-compile`
14. `process-test-classes`
15. `test`
16. `prepare-package`
17. `package`
18. `pre-integration-test`
19. `integration-test`
20. `post-integration-test`
21. `verify`
22. `install`
23. `deploy`

面试里最容易答错的一点是：**执行某个阶段时，`Maven` 会把当前生命周期中它之前的所有阶段一起执行掉**。

例如下面这条命令：

```bash
mvn package
```

它不是只做“打包”这一件事，而是会顺序执行到 `package` 为止，也就是先校验、编译、处理资源、跑单测，最后才打包。

### 3.3 为什么很多场景推荐 `mvn verify`

`Maven` 官方文档明确提到，如果你不确定应该调用哪个阶段，通常更推荐调用 `mvn verify`。

原因是 `verify` 比 `package` 更靠后，它会在完成打包之后，再继续执行集成测试校验、覆盖率校验、代码规范校验等附加质量检查。对于持续集成环境，它往往比 `package` 更符合“构建完成”的语义。

## 4. `phase`、`packaging` 与 `goal` 如何协作

### 4.1 为什么同样是 `package`，不同项目打包结果不同

因为 `phase` 只是阶段名称，真正执行的任务取决于：

1. 项目的 `packaging`
2. 该 `packaging` 对默认 `goal` 的绑定
3. 你在 `pom.xml` 里额外配置的插件执行

例如，官方文档给出的默认绑定里：

| `packaging` | `package` 阶段的典型 `goal` | 结果 |
| --- | --- | --- |
| `jar` | `jar:jar` | 生成 `jar` 包 |
| `war` | `war:war` | 生成 `war` 包 |
| `maven-plugin` | `jar:jar`、`plugin:addPluginArtifactMetadata` | 生成 Maven 插件制品 |
| `pom` | 无实际打包动作 | 通常只做聚合或父 POM |

这也是为什么聚合工程通常声明：

```xml
<packaging>pom</packaging>
```

此时它本身一般不产出业务可执行包，而是负责统一模块、依赖版本或插件配置。

### 4.2 `jar` 工程的默认绑定是什么

以最常见的 `jar` 工程为例，官方文档给出的默认绑定大致如下：

| 阶段 | 默认执行的 `goal` |
| --- | --- |
| `process-resources` | `resources:resources` |
| `compile` | `compiler:compile` |
| `process-test-resources` | `resources:testResources` |
| `test-compile` | `compiler:testCompile` |
| `test` | `surefire:test` |
| `package` | `jar:jar` |
| `install` | `install:install` |
| `deploy` | `deploy:deploy` |

所以从源码角度看，`phase` 本身不直接“干活”，`goal` 才是实际执行单元。`Maven` 生命周期本质上是在驱动一组插件目标按顺序运行。

### 4.3 如何把自定义任务挂到生命周期里

如果你希望某个插件在固定阶段自动执行，就需要在 `pom.xml` 中通过 `<executions>` 把 `goal` 绑定到某个 `phase`。

```xml
<build>
  <plugins>
    <plugin>
      <groupId>com.example</groupId>
      <artifactId>demo-plugin</artifactId>
      <executions>
        <execution>
          <id>run-on-verify</id>
          <phase>verify</phase>
          <goals>
            <goal>check</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

上面的配置表示：当构建执行到 `verify` 阶段时，额外执行一次 `demo-plugin:check`。

要注意两点：

- **只声明插件，不一定会执行你想要的 `goal`**。很多场景必须显式写 `<executions>`。
- 如果同一个阶段绑定了多个 `goal`，官方文档说明它们会按声明顺序执行；而 `packaging` 自带的默认绑定会先执行，再执行你在 `POM` 中新增的绑定。

## 5. 常见命令的差异与使用场景

### 5.1 `test`、`package`、`verify`、`install`、`deploy` 的区别

| 命令 | 会执行到哪里 | 适用场景 |
| --- | --- | --- |
| `mvn test` | 执行到 `test` | 只想编译并跑单元测试 |
| `mvn package` | 执行到 `package` | 本地快速产出 `jar` 或 `war` |
| `mvn verify` | 执行到 `verify` | 希望连质量校验、集成测试链路一起跑完 |
| `mvn install` | 执行到 `install` | 需要把产物安装到本地仓库给别的本地项目使用 |
| `mvn deploy` | 执行到 `deploy` | 需要把产物发布到远程仓库 |

三者最常被混淆的是 `package`、`install`、`deploy`：

- `package`：只把当前项目打成包，产物通常还只在当前工程的 `target` 目录里。
- `install`：在 `package` 基础上，把产物放进本地仓库，路径通常位于 `~/.m2/repository`。
- `deploy`：在 `install` 之后，再把产物推送到远程仓库，例如公司私服或制品仓库。

### 5.2 `clean package` 到底做了什么

```bash
mvn clean package
```

这条命令可以拆成两段理解：

1. 执行 `clean` 生命周期，把上一次构建残留删掉。
2. 执行 `default` 生命周期直到 `package`。

因此，它非常适合作为本地“干净构建”的通用命令。

### 5.3 为什么不建议直接调用 `integration-test`

官方文档特别强调，像 `pre-*`、`post-*`、`process-*` 这类中间阶段通常不应该直接从命令行调用。

以 `integration-test` 为例，很多插件会在 `pre-integration-test` 阶段拉起容器环境，在 `post-integration-test` 阶段清理环境，在 `verify` 阶段汇总报告。你如果只手工执行：

```bash
mvn integration-test
```

就可能出现测试容器没被正确清理、报告没生成完整、进程悬挂等问题。更稳妥的方式通常是执行：

```bash
mvn verify
```

## 6. 多模块项目中的生命周期

### 6.1 根工程为什么常用 `pom` 打包

多模块项目的根工程通常承担两类职责：

1. 作为父 `POM`，统一依赖版本、插件版本、公共属性。
2. 作为聚合工程，统一组织多个子模块一起构建。

这类工程通常配置为：

```xml
<packaging>pom</packaging>
```

因为它的重点不是产出一个业务包，而是管理整个模块集合。

### 6.2 在根目录执行命令会发生什么

官方文档说明，在多模块场景下执行：

```bash
mvn clean deploy
```

`Maven` 会遍历每个子项目，先执行 `clean`，再执行 `deploy` 以及其之前的所有前置阶段。也就是说，根目录的一条命令，实际会驱动整棵模块树按生命周期逐步构建。
