## 1. 文件与目录

- `ls`、`cd`、`pwd`、`mkdir`、`cp`、`mv`、`rm`
- `find`、`locate`、`du`、`df`

## 2. 文本与日志

- `cat`、`less`、`head`、`tail -f`
- `grep`、`awk`、`sed`

## 3. 进程与系统

- `ps`、`top`、`htop`、`kill`
- `free`、`uname`、`uptime`
- `systemctl`、`journalctl`

## 4. 网络与诊断

- `ping`、`traceroute`
- `curl`、`wget`
- `ss`、`netstat`、`lsof -i`

## 5. 常见追问/易错点

- 追问：`grep`、`awk`、`sed` 的适用场景分别是什么？
  - 答：核心结论是：常用命令主要覆盖文件与目录、文本处理、进程与系统、网络与诊断四类，熟悉这些命令即可满足日常开发与排障。展开时可按“文件与目录、文本与日志、进程与系统”组织，先概述再逐点展开，保证结构完整。其中文件与目录侧重`ls`、`cd`、`pwd`、`mkdir`、`cp`、`mv`、`rm`，文本与日志侧重`cat`、`less`、`head`、`tail -f`。回答时要体现步骤、关键点与适用场景，必要时补充示例或对比。
- 易错点：误用 `rm -rf` 或不理解管道导致数据不可恢复。
  - 答：常见易错点是只给结论不讲依据、边界条件与前提不清。比如文件与目录中提到：`ls`、`cd`、`pwd`、`mkdir`、`cp`、`mv`、`rm`。文本与日志中还提到：`cat`、`less`、`head`、`tail -f`。这些细节很容易被忽视。回答时应明确边界、关键步骤与适用场景，并用实例或对比验证。
