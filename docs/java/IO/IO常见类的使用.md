
#### 一、IO 分类

Java IO 按操作类型可分为：

1. **磁盘操作**：File。
2. **字节操作**：InputStream 和 OutputStream。
3. **字符操作**：Reader 和 Writer。
4. **对象操作**：Serializable。
5. **网络操作**：Socket。

#### 二、File 相关

1. **定义**：
    - File 表示文件或目录的信息（路径、名称等），不涉及内容。
2. **常见用法**：
    - **递归列出目录下所有文件**：
        
        `public static void listAllFiles(File dir) { if (dir == null || !dir.exists()) return; if (dir.isFile()) { System.out.println(dir.getName()); return; } for (File file : dir.listFiles()) { listAllFiles(file); } }`
        

#### 三、字节流相关

1. **核心类**：
    - InputStream：字节输入流。
    - OutputStream：字节输出流。
2. **文件复制示例**：
    
    `public static void copyFile(String src, String dist) throws IOException { FileInputStream in = new FileInputStream(src); FileOutputStream out = new FileOutputStream(dist); byte[] buffer = new byte[20 * 1024]; int len; while ((len = in.read(buffer, 0, buffer.length)) != -1) { out.write(buffer, 0, len); } in.close(); out.close(); }`
    
    - read() 返回实际读取字节数，-1 表示文件末尾。
    - 缓冲区大小（如 20KB）提高效率。

#### 四、字符流相关

1. **核心类**：
    - Reader：字符输入流。
    - Writer：字符输出流。
2. **逐行读取文件示例**：
    `public static void readFileContent(String filePath) throws IOException { FileReader fileReader = new FileReader(filePath); BufferedReader bufferedReader = new BufferedReader(fileReader); String line; while ((line = bufferedReader.readLine()) != null) { System.out.println(line); } bufferedReader.close(); // 自动关闭 fileReader }`
    
    - BufferedReader 通过装饰者模式组合 Reader，只需关闭外层即可。

#### 五、对象操作（序列化）

1. **定义**：
    - 序列化：将对象转为字节序列，便于存储或传输。
    - 反序列化：从字节序列恢复对象。
2. **核心类**：
    - ObjectOutputStream：writeObject()。
    - ObjectInputStream：readObject()。
3. **特点**：
    - 不序列化静态变量（类状态）。
    - 类需实现 Serializable 接口（标记接口，无方法）。
4. **示例**：
    `public static void main(String[] args) throws IOException, ClassNotFoundException { A a1 = new A(123, "abc"); String objectFile = "file/a1"; ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(objectFile)); out.writeObject(a1); out.close(); ObjectInputStream in = new ObjectInputStream(new FileInputStream(objectFile)); A a2 = (A) in.readObject(); in.close(); System.out.println(a2); // x = 123 y = abc } private static class A implements Serializable { private int x; private String y; A(int x, String y) { this.x = x; this.y = y; } @Override public String toString() { return "x = " + x + " y = " + y; } }`
    
5. **transient**：
    - 修饰属性，阻止序列化。
    - 示例：ArrayList 的 transient Object[] elementData，通过自定义序列化方法只保存有效数据。

#### 六、网络操作

1. **核心类**：
    - **InetAddress**：表示 IP 地址。
    - **URL**：统一资源定位符。
    - **Socket**：TCP 通信。
    - **Datagram**：UDP 通信。
2. **InetAddress**：
    - 无公共构造方法，通过静态方法创建：
        - InetAddress.getByName(String host)。
        - InetAddress.getByAddress(byte[] address)。
3. **URL**：
    - 从 URL 读取数据：
        
        `public static void main(String[] args) throws IOException { URL url = new URL("http://www.baidu.com"); InputStream is = url.openStream(); InputStreamReader isr = new InputStreamReader(is, "utf-8"); BufferedReader br = new BufferedReader(isr); String line; while ((line = br.readLine()) != null) { System.out.println(line); } br.close(); }`
        
4. **Socket**：
    - **ServerSocket**：服务器端。
    - **Socket**：客户端。
    - 通过 InputStream 和 OutputStream 通信。
5. **Datagram**：
    - **DatagramSocket**：通信类。
    - **DatagramPacket**：数据包类。
    - 用于 UDP 无连接通信。
