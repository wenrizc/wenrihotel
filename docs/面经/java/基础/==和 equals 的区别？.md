
#### 区别
- **`==`**：
  - 比较运算符，判断基本类型的**值**是否相等，或引用类型的**地址**是否相同。
- **`equals`**：
  - 对象方法，比较两个对象的**内容**是否相等（默认比较地址，可重写）。

#### 核心点
- `==`：关注内存级别（值或引用）。
- `equals`：关注逻辑级别（内容）。

---

### 1. 区别详解
#### (1) 使用对象
- **`==`**：
  - 适用于基本类型（如 `int`、`double`）和引用类型（如 `String`、`Object`）。
- **`equals`**：
  - 只适用于对象（引用类型），基本类型无此方法。

#### (2) 比较内容
- **`==`**：
  - 基本类型：比较值。
  - 引用类型：比较内存地址（是否同一对象）。
- **`equals`**：
  - 默认（`Object` 类）：比较地址（与 `==` 相同）。
  - 重写后（如 `String`）：比较内容。

#### (3) 默认行为
- **`==`**：
  - 固定行为，无法修改。
- **`equals`**：
  - `Object` 类定义：
```java
public boolean equals(Object obj) {
    return (this == obj); // 默认比较地址
}
```
  - 可重写自定义逻辑。

#### 示例
```java
// 基本类型
int a = 5;
int b = 5;
System.out.println(a == b); // true（值相等）

// 引用类型
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);      // false（不同对象）
System.out.println(s1.equals(s2)); // true（内容相等）

String s3 = "hello"; // 字符串常量池
String s4 = "hello";
System.out.println(s3 == s4);      // true（同一对象）
System.out.println(s3.equals(s4)); // true（内容相等）
```

---

### 2. equals 的重写
#### String 类
- **重写逻辑**：
```java
public boolean equals(Object anObject) {
    if (this == anObject) return true; // 地址相同直接返回
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            for (int i = 0; i < n; i++) {
                if (v1[i] != v2[i]) return false;
            }
            return true;
        }
    }
    return false;
}
```
- **效果**：比较字符数组内容。

#### 自定义类
```java
class Person {
    String name;
    int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Person)) return false;
        Person p = (Person) obj;
        return name.equals(p.name) && age == p.age;
    }
}

Person p1 = new Person("Alice", 25);
Person p2 = new Person("Alice", 25);
System.out.println(p1 == p2);      // false（不同对象）
System.out.println(p1.equals(p2)); // true（内容相等）
```

---

### 3. 注意事项
#### (1) Null 处理
- **`==`**：
  - 可与 `null` 比较。
  - `obj == null` 返回 `true` 或 `false`。
- **`equals`**：
  - 调用者为 `null` 抛 `NullPointerException`。
  - 被比较对象为 `null` 返回 `false`。
```java
String s = null;
System.out.println(s == null);      // true
// System.out.println(s.equals(null)); // NullPointerException
```

#### (2) 自动装箱
- **`Integer` 等包装类**：
  - `==` 比较地址，受缓存影响（-128 到 127）。
  - `equals` 比较值。
```java
Integer a = 128;
Integer b = 128;
System.out.println(a == b);      // false（新对象）
System.out.println(a.equals(b)); // true（值相等）
```

---

### 4. 特点对比
| **特性**         | **`==`**          | **`equals`**      |
|------------------|-------------------|-------------------|
| **适用类型**     | 基本 + 引用       | 引用类型          |
| **比较内容**     | 值 / 地址         | 地址 / 内容       |
| **可定制**       | 否                | 是（重写）        |
| **null 处理**    | 支持              | 调用者不可 null   |
| **性能**         | 高（直接比较）    | 稍低（方法调用）  |

---

### 5. 延伸与面试角度
- **与 hashCode 关系**：
  - `equals` 相等，`hashCode` 必须相等。
  - 重写 `equals` 时需重写 `hashCode`。
- **适用场景**：
  - `==`：判断同一对象或基本值。
  - `equals`：判断逻辑相等。
- **实际应用**：
  - `HashMap`：用 `equals` 比较 key。
- **面试点**：
  - 问“区别”时，提地址 vs 内容。
  - 问“陷阱”时，提 null 和缓存。

---

### 总结
`==` 比较值或地址，`equals` 默认地址但可重写为内容。基本类型用 `==`，对象用 `equals` 更灵活。面试时，可提 `String` 重写或举自定义类，展示理解深度。