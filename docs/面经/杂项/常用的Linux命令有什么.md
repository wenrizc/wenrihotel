
1.  文件和目录操作命令

*   `ls`: 列出目录内容。常用参数：
    *   `-l`: 显示详细信息（权限、所有者、大小、修改时间等）。
    *   `-a`: 显示所有文件，包括隐藏文件（以`.`开头）。
    *   `-h`: 以人类可读的格式显示文件大小（如K, M, G）。
    *   `-t`: 按修改时间排序。
*   `cd`: 切换目录。`cd ..`返回上一级目录，`cd ~`或`cd`回到当前用户的主目录。
*   `pwd`: 显示当前工作目录的绝对路径。
*   `mkdir`: 创建新目录。`-p`可以递归创建多级目录。
*   `rm`: 删除文件或目录。
    *   `-r`: 递归删除目录及其内容。
    *   `-f`: 强制删除，不进行提示。
    *   使用`rm -rf`时需要格外小心。
*   `cp`: 复制文件或目录。`-r`用于复制目录。
*   `mv`: 移动或重命名文件/目录。
*   `touch`: 创建一个空文件，或者更新一个已存在文件的时间戳。
*   `find`: 在指定目录下查找文件。功能强大，可以按名称、大小、类型、修改时间等多种条件查找。
    *   例如：`find /path/to/search -name "*.log" -mtime -7` (查找7天内修改过的log文件)。
*   `ln`: 创建链接。`-s`创建软链接（符号链接）。

2.  文件内容查看与编辑命令

*   `cat`: 查看并连接文件内容，一次性显示整个文件。
*   `more`: 分页显示文件内容，只能向下翻页。
*   `less`: 更强大的分页显示工具，可以向上/下翻页，支持搜索。
*   `head`: 显示文件的开头部分（默认前10行）。`-n`指定行数。
*   `tail`: 显示文件的结尾部分（默认后10行）。
    *   `-n`: 指定行数。
    *   `-f`: 实时跟踪文件的末尾内容，非常适合查看实时日志。
*   `grep`: 在文件中搜索包含指定模式的行。功能强大，常与正则表达式结合使用。
    *   `-i`: 忽略大小写。
    *   `-v`: 显示不匹配的行。
    *   `-r`或`-R`: 递归搜索目录。
    *   `-C <num>`, `-B <num>`, `-A <num>`: 显示匹配行及其上下文。
*   `vi`/`vim`: 功能强大的文本编辑器，是Linux下必备的编辑工具。
*   `wc`: 统计文件的行数、单词数、字节数。`-l`只统计行数。

3.  系统信息与性能监控命令

*   `top`: 实时动态地查看系统的整体运行情况，包括进程信息、CPU使用率、内存使用率等。
*   `htop`: `top`的增强版，交互更友好，信息更丰富。
*   `ps`: 查看当前系统的进程状态。
    *   `ps aux`或`ps -ef`: 显示所有进程的详细信息。常与`grep`结合使用来查找特定进程。
*   `df`: 查看磁盘空间使用情况。`-h`以可读格式显示。
*   `du`: 查看文件或目录占用的磁盘空间大小。
    *   `-h`: 以可读格式显示。
    *   `-s`: 只显示总计大小。
*   `free`: 查看内存和交换空间的使用情况。`-m`或`-g`以MB或GB为单位显示。
*   `vmstat`: 报告虚拟内存统计信息，也可以报告进程、I/O、CPU活动等。
*   `iostat`: 报告CPU统计信息和输入/输出统计信息。
*   `netstat`: 显示网络连接、路由表、接口统计等信息。
    *   `-anp`: 显示所有连接、以数字形式显示地址和端口、并显示进程ID和名称。
*   `ss`: `netstat`的替代品，功能更强大，速度更快。`ss -tuln`是常用组合。
*   `uptime`: 显示系统已经运行了多长时间、当前用户数以及系统平均负载。
*   `uname`: 显示系统内核信息。`-a`显示所有信息。
*   `lsof`: 列出当前系统打开的文件。`-i`可以查看网络连接。

4.  网络命令

*   `ping`: 测试与目标主机的网络连通性。
*   `telnet`/`nc` (netcat): 用于测试远程主机的端口是否开放。
*   `curl`/`wget`: 用于从网络下载文件或测试HTTP/HTTPS接口。`curl`功能更强大，常用于API调试。
*   `ifconfig`/`ip addr`: 查看和配置网络接口信息。`ip`命令是新一代的网络配置工具。
*   `route`/`ip route`: 查看和管理路由表。

5.  压缩与解压命令

*   `tar`: 打包和解包文件。常与压缩命令结合使用。
    *   `-cvf`: 创建一个tar包。
    *   `-xvf`: 解开一个tar包。
    *   `-z`: 使用gzip进行压缩/解压（.tar.gz）。
    *   `-j`: 使用bzip2进行压缩/解压（.tar.bz2）。
*   `gzip`/`gunzip`: .gz文件的压缩与解压。
*   `zip`/`unzip`: .zip文件的压缩与解压。

6.  权限管理命令

*   `chmod`: 修改文件或目录的权限。
*   `chown`: 修改文件或目录的所有者和所属组。

7.  管道与重定向

*   `|` (管道): 将一个命令的输出作为另一个命令的输入。这是组合命令实现复杂功能的关键。
    *   例如：`ps aux | grep java` (查找Java进程)。
*   `>` (输出重定向): 将命令的输出写入到文件中（覆盖）。
*   `>>` (输出重定向): 将命令的输出追加到文件中。
*   `<` (输入重定向): 将文件的内容作为命令的输入。

8.  其他常用命令

*   `ssh`: 安全地远程登录到另一台主机。
*   `scp`: 在本地和远程主机之间安全地复制文件。
*   `systemctl`/`service`: 管理系统服务（启动、停止、重启、查看状态），适用于使用systemd或SysVinit的系统。
*   `crontab`: 设置定时任务。
*   `history`: 查看命令历史记录。

