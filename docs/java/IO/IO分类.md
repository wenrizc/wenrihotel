
#### 一、从传输方式分类

从数据传输或运输方式的角度，IO 类分为：

1. **字节流**：
    - 以字节（8 bit）为单位传输数据。
    - 适合处理二进制文件（如图片、MP3、视频）。
2. **字符流**：
    - 以字符为单位传输数据（字符大小依编码而定）。
    - 适合处理文本文件。

**核心区别**：

- **字节**：计算机视角的最小单位，给机器处理。
- **字符**：人类可读的单位，给人理解。

#### 二、字节流

- **定义**：操作原始字节数据。
- **类结构**：
    - **输入**：InputStream
        - 子类：FileInputStream、ByteArrayInputStream 等。
    - **输出**：OutputStream
        - 子类：FileOutputStream、ByteArrayOutputStream 等。
- **特点**：
    - 读取/写入单个字节。
    - 用于二进制数据处理。

#### 三、字符流

- **定义**：操作字符数据（依赖编码）。
- **类结构**（部分派生类略）：
    - **输入**：Reader
        - 子类：FileReader、CharArrayReader 等。
    - **输出**：Writer
        - 子类：FileWriter、CharArrayWriter 等。
- **特点**：
    - 读取/写入单个字符。
    - 用于文本数据处理。
、
#### 四、字节流与字符流的区别

1. **操作单位**：
    - 字节流：单个字节。
    - 字符流：单个字符（依编码，1 字符 = 1~3 字节）。
        - UTF-8：中文 3 字节，英文 1 字节。
        - GBK：中文 2 字节，英文 1 字节。
2. **适用场景**：
    - 字节流：二进制文件（图片、视频等）。
    - 字符流：文本文件（编码后的特殊二进制，人可读）。
3. **本质**：
    - 字节流给计算机用，字符流给人用。

#### 五、字节与字符的转换

1. **编码与解码**：
    - **编码**：字符 → 字节。
    - **解码**：字节 → 字符。
    - 编码/解码不一致 → 乱码。
2. **常见编码**：
    - **GBK**：中文 2 字节，英文 1 字节。
    - **UTF-8**：中文 3 字节，英文 1 字节。
    - **UTF-16be**：中文/英文均 2 字节（大端序）。
    - **UTF-16le**：小端序。
3. **Java 特性**：
    - char 类型使用 UTF-16be 编码，占 2 字节。
    - 一个 char 可存储 1 个中文或英文。
4. **转换类**：
    - **输入**：InputStreamReader（字节流 → 字符流）。
    - **输出**：OutputStreamWriter（字符流 → 字节流）。

#### 六、从数据操作分类

从数据来源或操作对象角度，IO 类分为：

1. **文件 (File)**：
    - 字节流：FileInputStream、FileOutputStream。
    - 字符流：FileReader、FileWriter。
    - 用途：读写文件。
2. **数组 (Array)**：
    - 字节数组：ByteArrayInputStream、ByteArrayOutputStream。
    - 字符数组：CharArrayReader、CharArrayWriter。
    - 用途：操作内存中的数组数据。
3. **管道 (Piped)**：
    - 字节流：PipedInputStream、PipedOutputStream。
    - 字符流：PipedReader、PipedWriter。
    - 用途：线程间通信。
4. **基本数据类型**：
    - DataInputStream、DataOutputStream。
    - 用途：读写基本类型（如 int、double）。
5. **缓冲操作 (Buffered)**：
    - 字节流：BufferedInputStream、BufferedOutputStream。
    - 字符流：BufferedReader、BufferedWriter。
    - 用途：通过缓冲减少底层 IO 操作，提高效率。
6. **打印 (Print)**：
    - PrintStream、PrintWriter。
    - 用途：格式化输出（如 System.out）。
7. **对象序列化反序列化**：
    - ObjectInputStream、ObjectOutputStream。
    - 用途：对象持久化或网络传输。
8. **转换**：
    - InputStreamReader、OutputStreamWriter。
    - 用途：字节流与字符流互转。
