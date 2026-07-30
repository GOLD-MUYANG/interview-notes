# Linux 常用命令学习手册

本手册面向刚开始使用 Linux 终端的学习者，主要素材来自 `.codex/` 中保留的真实项目排障记录。重点不是背诵命令，而是学会判断：当前要解决什么问题、用哪条命令收集证据、命令是否会修改系统状态。

除非特别说明，Sylar 项目示例假设当前目录是：

```text
/home/muyang/workspace/sylar
```

`rg` 是 ripgrep 提供的命令，不是 POSIX 或 Bash 内置命令；如果系统没有安装，可以在理解用法后改用 `grep` 和 `find`。网络请求、本地端口、编译和测试命令都依赖当前环境，不同机器的输出可能不同。

## 1. 怎样阅读一条 Linux 命令

### 1.1 命令的基本结构

一条命令通常可以拆成三部分：

```text
command [options] [arguments]
```

- `command`：要执行的命令，如 `rg`、`sed` 或 `git`。
- `options`：改变命令的行为，一般以 `-` 或 `--` 开头。
- `arguments`：命令要处理的对象，如文件、目录、搜索词或 Git 分支。

例如：

```bash
rg -n 'handleClient' sylar/http
```

| 部分 | 内容 | 含义 |
| --- | --- | --- |
| 命令 | `rg` | 在文件中搜索文本 |
| 选项 | `-n` | 显示匹配行的行号 |
| 参数 | `'handleClient'` | 搜索的文本 |
| 参数 | `sylar/http` | 搜索范围 |

短选项常用单个字母，如 `-n`；长选项使用完整单词，如 `--files` 和 `--help`。不同命令的选项含义可能不同，不能只靠字母猜测。

### 1.2 空格、引号与通配符

Shell 用空格分隔命令的各个部分。参数中包含空格或 Shell 特殊字符时，需要使用引号：

```bash
rg -n 'HTTP Server' README.md
```

- 单引号 `'...'` 中的内容基本原样传给命令。
- 双引号 `"..."` 会展开 `$VARIABLE` 等内容。
- 不加引号的 `*` 可能由 Shell 先展开成多个文件名。

初学阶段，搜索词和包含特殊字符的参数优先用单引号包围。

### 1.3 相对路径与绝对路径

- 绝对路径从 `/` 开始，例如 `/home/muyang/workspace/sylar/README.md`。
- 相对路径从当前目录开始解释，例如 `sylar/http/http_server.cc`。
- `.` 表示当前目录，`..` 表示上一级目录。
- `~` 通常由 Shell 展开为当前用户的主目录。

两条命令可能查看同一个文件：

```bash
sed -n '1,20p' README.md
sed -n '1,20p' /home/muyang/workspace/sylar/README.md
```

第一条只有在当前目录是仓库根目录时才成立；第二条不依赖当前目录，但可移植性较差。

### 1.4 退出状态

命令执行结束后会返回一个状态码。通常 `0` 表示成功，非 `0` 表示未匹配、参数错误、文件不存在或其他失败。`$?` 保存上一条命令的退出状态：

```bash
pwd
echo $?
```

必须立即检查 `$?`，因为执行 `echo $?` 后，“上一条命令”就已经变成了 `echo`。

## 2. 获取帮助与确认命令来源

不确定命令怎么用时，先查帮助，不要猜参数。

### 2.1 `--help`：快速查看选项

大多数外部命令都支持 `--help`：

```bash
rg --help
git diff --help
```

`rg --help` 直接在终端输出帮助。`git diff --help` 在部分环境中会打开 man 页；只想在终端看简短用法时可用 `git diff -h`。

### 2.2 `man`：阅读完整手册

```bash
man sed
man 1 printf
```

`man` 适合查看完整语法、选项和返回值。常用操作：

- `/word`：在 man 页中搜索。
- `n`：跳到下一个匹配。
- `q`：退出。

如果系统没有安装 man 页，`man` 可能提示找不到手册，这不代表命令本身不存在。

### 2.3 `type`：判断 Shell 将执行什么

```bash
type cd
type rg
type -a printf
```

`type` 能区分：

- Shell 内置命令，如 `cd`。
- 别名 alias。
- Shell 函数。
- 通过 `PATH` 找到的可执行文件。

`type -a` 会列出所有同名来源，适合排查“输入的命令不是自己以为的那一个”。

### 2.4 `which`：查找 `PATH` 中的可执行文件

```bash
which cmake
which rg
```

`which` 主要查找外部可执行文件，不能像 `type` 那样完整解释 alias、函数和内置命令。排查 Shell 实际行为时优先使用 `type`；只需要可执行文件路径时再用 `which`。

## 3. 路径、目录与文件管理

### 3.1 `pwd`、`cd`、`ls`：先确认自己在哪里

```bash
pwd
ls
ls -la
cd sylar/http
cd ../..
```

- `pwd`（print working directory）输出当前目录的绝对路径。
- `cd PATH` 切换当前 Shell 的工作目录。`cd` 必须由 Shell 自身处理，否则无法改变当前 Shell 的目录。
- `ls` 列出目录内容；`-l` 显示详细信息，`-a` 包含名称以 `.` 开头的隐藏项。

运行依赖相对路径的命令前，先用 `pwd` 确认位置，能避免很多“文件不存在”或“读错配置目录”的问题。

### 3.2 `find` 与 `rg --files`：按条件找文件

`find` 是常见的递归文件查找工具：

```bash
find sylar/http -type f -name '*.cc'
find .codex -maxdepth 1 -type f -name '*.md'
```

- `-type f`：只选择普通文件。
- `-type d`：只选择目录。
- `-name '*.cc'`：按文件名匹配；用引号防止 `*` 被 Shell 提前展开。
- `-maxdepth 1`：最多向下查找一层。

`rg --files` 专注于快速列出文件，并会遵守 `.gitignore` 等忽略规则：

```bash
rg --files sylar/http tests
rg --files .codex -g '*.md'
```

`-g '*.md'` 是 glob 过滤条件。需要按类型、时间或目录深度精细查找时用 `find`；在代码仓库中快速获取文件清单时优先 `rg --files`。

### 3.3 `mkdir`、`cp`、`mv`：创建、复制和移动

下面的练习只修改临时目录：

```bash
mkdir -p /tmp/linux-command-practice/source
cp README.md /tmp/linux-command-practice/source/
mv /tmp/linux-command-practice/source/README.md /tmp/linux-command-practice/source/README.copy.md
```

- `mkdir -p`：递归创建缺失的父目录；目录已存在时不报错。
- `cp SOURCE DEST`：复制文件。
- `mv SOURCE DEST`：移动或重命名文件。

`cp` 和 `mv` 的目标已存在时可能覆盖它。初学时可使用 `-i` 在覆盖前询问：

```bash
cp -i README.md /tmp/linux-command-practice/source/README.copy.md
mv -i /tmp/linux-command-practice/source/README.copy.md /tmp/linux-command-practice/README.copy.md
```

执行前先用 `ls -l` 检查源和目标，不要在不确定展开结果时对宽泛通配符执行移动。

### 3.4 `cmp`：判断两个文件是否逐字节相同

```bash
cp README.md /tmp/linux-command-practice/README.for-compare.md
cmp README.md /tmp/linux-command-practice/README.for-compare.md
cmp -s README.md /tmp/linux-command-practice/README.for-compare.md
echo $?
```

- 默认模式在发现第一处差异时报告位置。
- `-s` 不输出内容，只通过退出状态表示结果：`0` 相同，`1` 不同，其他值通常是读取错误。

`.codex` 中曾用 `cmp -s` 判断 worktree 与主仓库文件是否一致。它只告诉你“是否一样”，需要看具体差异时使用 `diff` 或 `git diff`。

### 3.5 `stat` 与 `file`：查看元数据和文件类型

```bash
stat README.md
stat -c '%y %n' modules/ai_gateway/real_provider_smoke.cc bin/ai_gateway_real_provider_smoke
file bin/sylar
file README.md
```

- `stat` 显示大小、权限、时间戳等元数据。
- `stat -c FORMAT` 自定义输出；`%y` 是内容修改时间，`%n` 是文件名。
- `file` 根据文件内容判断文本、ELF 可执行文件、共享库等类型，不只看扩展名。

对比源文件、目标文件和可执行文件的时间戳，可以排查“源码已修改，实际运行的二进制还是旧版”。

### 3.6 `readlink` 与 `realpath`：解析链接和规范路径

```bash
readlink /proc/self/exe
readlink -f /proc/self/exe
realpath sylar/http/../http/http_server.cc
```

- `readlink LINK` 输出符号链接直接保存的目标。
- `readlink -f` 递归跟随链接并规范化路径。
- `realpath` 将路径转换为规范化的绝对路径。

它们常用于确认“这个命令或库文件最终指向哪里”。

## 4. 查看文件与搜索文本

查文件时先问自己两个问题：是要看全部内容，还是只看一段？是已经知道文件名，还是只知道要找的文本？

### 4.1 `cat` 与 `less`：查看全文

```bash
cat CMakeLists.txt
less sylar/http/http_server.cc
```

- `cat FILE` 将文件全部写到标准输出，适合小文件，也常用于将内容传入管道。
- `less FILE` 分页显示，适合大文件。用方向键或 `PageUp/PageDown` 移动，`/word` 搜索，`q` 退出。

对几百行的源文件直接 `cat` 往往会把有用内容快速冲出屏幕，此时优先 `less`、`sed -n` 或文本搜索。

### 4.2 `head` 与 `tail`：查看开头和结尾

```bash
head -n 20 README.md
tail -n 30 README.md
tail -f /tmp/sylar.log
```

- `head -n 20` 查看前 20 行。
- `tail -n 30` 查看后 30 行。
- `tail -f` 持续跟踪新追加的内容，常用于日志；按 `Ctrl-C` 停止。

`tail -f` 会一直运行，不是命令卡住。示例中的日志文件必须存在，路径应替换为实际日志路径。

### 4.3 `nl` 与 `sed -n`：带行号查看指定范围

```bash
nl -ba sylar/http/http_session.cc | sed -n '1,120p'
sed -n '1,80p' sylar/http/http_server.cc
sed -n '/^## 3\./,/^## 4\./p' '文档/Linux常用命令学习手册.md'
```

- `nl -ba FILE` 为所有行加行号，包括空行。
- `sed -n '1,80p' FILE` 只打印第 1 到 80 行。
- `sed -n '/START/,/END/p' FILE` 打印从起始匹配到结束匹配的范围。

这里的 `sed` 不会修改文件：`-n` 是关闭默认输出，`p` 是打印匹配内容。使用 `sed -i` 才会就地修改文件，在理解替换规则前不要对重要文件使用它。

### 4.4 `rg`：在代码和文档中快速搜索

```bash
rg -n 'recvRequest|handleClient' sylar/http
rg -n -i 'todo|fixme' sylar tests
rg -n 'HttpServer' -g '*.cc' sylar
rg -n -F 'socket(2, 1, 0)' .codex
```

| 选项 | 用途 |
| --- | --- |
| `-n` | 显示行号 |
| `-i` | 忽略大小写 |
| `-g '*.cc'` | 用 glob 限制文件名 |
| `-F` | 把搜索词当作普通文本，不解释正则符号 |
| `--files` | 列出文件，而不搜索文件内容 |

`rg` 默认使用正则表达式，所以 `'recvRequest|handleClient'` 表示匹配任意一个单词。要搜索包含括号、星号等符号的完整日志文本时，`-F` 更不容易写错。

`rg` 在没有匹配时退出码通常是 `1`，这可能只表示“没找到”，不是工具损坏。

### 4.5 `grep`：通用文本过滤

```bash
grep -n 'HttpServer' sylar/http/http_server.cc
grep -Rni --include='*.cc' 'handleClient' sylar/http
```

- `-n`：显示行号。
- `-i`：忽略大小写。
- `-R`：递归遍历目录。
- `--include='*.cc'`：只搜索指定文件类型。

`grep` 在大多数 Linux 环境中更常见；`rg` 在代码仓库中通常更快，并默认跳过隐藏文件和忽略项。二者的正则语法和默认行为并不完全相同。

### 4.6 `wc`、`tr` 与 `sort`：统计、转换和排序

```bash
wc -l README.md
rg --files .codex -g '*.md' | wc -l
printf 'a,b,c\n' | tr ',' '\n'
```

- `wc -l` 统计行数，`-w` 统计单词数，`-c` 统计字节数。
- `tr SET1 SET2` 将输入中属于第一个字符集的字符映射为第二个字符集。
- `sort` 按行排序文本；如果不带文件参数，它从标准输入读取。

Linux 的 `/proc/<PID>/environ` 用空字节分隔环境变量，因此排障记录中使用过 `tr '\0' '\n'` 将它转成每行一项。第 7 章会讲如何在不打印密钥的前提下安全检查。

## 5. 管道、重定向与命令串联

### 5.1 标准输入、标准输出与标准错误

进程默认有三条基本数据通道：

| 编号 | 名称 | 默认来源或去向 |
| --- | --- | --- |
| `0` | 标准输入 stdin | 键盘 |
| `1` | 标准输出 stdout | 终端 |
| `2` | 标准错误 stderr | 终端 |

管道和重定向的本质，就是改变这些数据通道的连接方式。

### 5.2 管道 `|`：把前一条命令的输出交给后一条

```bash
rg -n 'HttpServer' sylar/http | head
rg --files sylar/http tests | sort
ps -ef | rg 'bin/sylar|mock_model_provider'
```

`A | B` 将 A 的标准输出连接到 B 的标准输入。B 看到的是文本流，而不是 A 这个进程本身。

管道默认不会把 A 的标准错误传给 B。这是为什么有时搜索结果被过滤了，错误信息仍然直接出现在终端。

### 5.3 `>` 与 `>>`：将输出写入文件

在临时练习目录中执行：

```bash
mkdir -p /tmp/linux-command-practice
printf 'first line\n' > /tmp/linux-command-practice/output.txt
printf 'second line\n' >> /tmp/linux-command-practice/output.txt
cat /tmp/linux-command-practice/output.txt
```

- `>` 创建或覆盖目标文件。这个覆盖在命令启动前就可能发生。
- `>>` 在文件末尾追加，目标不存在时创建它。

`printf 'first line\n'` 按格式输出文本，其中 `\n` 表示换行。在脚本中，`printf` 通常比依赖不同 Shell 行为的 `echo` 更可控。

高风险误区：

```bash
# 不要这样做：sort 还没读取时，Shell 可能已经清空了 file.txt
sort file.txt > file.txt
```

应写到一个新文件，检查后再决定是否替换原文件。

### 5.4 `<` 与 `2>&1`：改变输入和合并错误输出

```bash
wc -l < README.md
cmake --build build --target test_http_connection > /tmp/linux-command-practice/build.log 2>&1
```

- `< FILE` 让命令从文件获取标准输入。`wc -l < README.md` 只输出数字，而 `wc -l README.md` 通常还会输出文件名。
- `2>&1` 让文件描述符 `2`（stderr）指向当前的 `1`（stdout）。

重定向从左到右处理，所以顺序很重要：

```bash
# stdout 先指向文件，然后 stderr 跟随 stdout：两者都进文件
command > output.log 2>&1

# stderr 先跟随当时的 stdout，之后 stdout 才进文件：两者可能分开
command 2>&1 > output.log
```

### 5.5 `&&` 与 `||`：根据成功或失败决定是否继续

```bash
cmake --build build --target test_http_request_options && ./bin/test_http_request_options
rg -n 'handleClient' sylar/http || echo 'not found'
```

- `A && B`：只有 A 的退出状态是 `0` 时才执行 B。
- `A || B`：只有 A 的退出状态不是 `0` 时才执行 B。

这与管道不同：`|` 传递数据，`&&` 和 `||` 根据退出状态控制流程。

注意，`rg` 没有匹配时会返回非 `0`，所以第二个示例会输出 `not found`；这不一定是系统故障。

### 5.6 长命令的阅读方法

看到一条长命令时，按下面顺序拆解：

```bash
timeout 10 bash -lc '</dev/tcp/ark.cn-beijing.volces.com/443' && echo tcp_ok || echo tcp_failed
```

1. `timeout 10 ...`：最多运行 10 秒。
2. `bash -lc '...'`：让 Bash 执行引号中的命令字符串。
3. `</dev/tcp/HOST/PORT`：尝试打开 Bash 的特殊 TCP 连接。
4. `&& echo tcp_ok`：前面成功时输出成功标记。
5. `|| echo tcp_failed`：整个前缀结果为失败时输出失败标记。

不要在没有拆清数据流、退出状态和写入目标时盲目复制长命令。

## 6. 文件权限与 Shell 脚本

### 6.1 阅读 `ls -l` 中的权限

```bash
ls -l scripts/demo_ai_gateway.sh
```

权限串可能类似：

```text
-rwxr-xr-x
```

- 第 1 位：文件类型，`-` 是普通文件，`d` 是目录，`l` 是符号链接。
- 第 2–4 位：所有者 owner 的权限。
- 第 5–7 位：所属组 group 的权限。
- 第 8–10 位：其他用户 other 的权限。
- `r`：读，`w`：写，`x`：执行，`-`：没有该权限。

目录上的 `r/w/x` 语义与普通文件不完全相同：`x` 主要表示可以进入并访问其中已知名称的项。

### 6.2 `chmod`：修改权限

在临时目录练习：

```bash
cp scripts/demo_ai_gateway.sh /tmp/linux-command-practice/demo.sh
chmod u+x /tmp/linux-command-practice/demo.sh
ls -l /tmp/linux-command-practice/demo.sh
```

- `u+x`：为所有者 user 增加执行权限。
- `g-w`：移除所属组的写权限。
- `o=r`：将其他用户的权限精确设置为只读。

初学时优先使用这种符号形式，因为它更容易看出改了哪一组权限。不要为了解决 `Permission denied` 就直接 `chmod -R 777 ...`：它会对整棵目录放开读、写和执行权限，既宽泛又难以恢复原状。

### 6.3 直接执行脚本与 `bash script.sh`

```bash
./scripts/demo_ai_gateway.sh
bash scripts/demo_ai_gateway.sh
```

两者边界不同：

- `./script.sh` 要求文件有执行权限，内核根据首行 shebang 选择解释器。
- `bash script.sh` 明确让 Bash 读取文件，脚本本身不需要执行权限，但使用的语法必须与 Bash 兼容。

常见 shebang：

```bash
#!/usr/bin/env bash
```

它告诉系统从当前 `PATH` 中查找 `bash`。不要因为文件名以 `.sh` 结尾就默认它一定是 Bash 脚本，应检查 shebang 和实际语法。

### 6.4 `bash -n`：只检查 Bash 语法

```bash
bash -n scripts/demo_ai_gateway.sh
bash -n scripts/demo_ai_gateway_real_provider.sh
```

`-n` 读取脚本并检查语法，不执行脚本中的普通命令。它能发现未闭合引号、`if/fi` 不匹配等语法问题，但不能证明：

- 文件路径一定存在。
- 环境变量已设置。
- 端口未被占用。
- 脚本的业务逻辑正确。

因此 `bash -n` 是低成本的第一道检查，不是完整验证。

## 7. 进程、信号与端口排查

进程排查的基本顺序是：先查谁在运行，再查它监听什么端口，最后才考虑发送信号。

### 7.1 `ps`：查看进程快照

```bash
ps -ef
ps -ef | rg 'bin/sylar|mock_model_provider'
```

`ps -ef` 使用 Unix 风格选项列出当前能看到的全部进程：

- `UID`：运行进程的用户。
- `PID`：进程号。
- `PPID`：父进程号。
- `STIME`：启动时间。
- `CMD`：命令行。

`ps` 默认是一次性快照，不会持续刷新。用 `rg` 过滤时，结果里可能包含 `rg` 自身，应结合完整命令行判断。

### 7.2 `pgrep`：按名称或命令行找进程

```bash
pgrep -af 'bin/sylar'
pgrep -af 'mock_model_provider'
```

- `-f`：匹配完整命令行，而不是只匹配进程名。
- `-a`：连同命令行一起输出。

`pgrep` 比 `ps | rg` 更适合脚本化查找 PID，但它仍然可能匹配到多个进程。在将输出传给 `kill` 之前，必须先人工确认每个 PID 的完整命令行。

### 7.3 `ss`：查看 TCP 监听端口

```bash
ss -ltn
ss -ltnp
```

| 选项 | 含义 |
| --- | --- |
| `-l` | 只看 listening 监听套接字 |
| `-t` | 只看 TCP |
| `-n` | 直接显示数字地址和端口，不做名称解析 |
| `-p` | 尝试显示使用套接字的进程 |

`Local Address:Port` 中：

- `127.0.0.1:18080` 通常只接受本机 IPv4 访问。
- `0.0.0.0:18080` 表示绑定本机所有 IPv4 地址，是否能从外部访问还取决于防火墙和网络。
- `[::]:18080` 是 IPv6 的通配监听形式。

普通用户使用 `-p` 时可能看不到其他用户的进程信息；先用 `ss -ltn` 确认端口是否监听通常已经足够。

### 7.4 `kill`：向进程发送信号

```bash
kill -TERM <PID>
kill -KILL <PID>
```

`<PID>` 是教程占位符，执行时必须替换为经 `ps` 或 `pgrep -af` 确认的实际数字进程号。

- `SIGTERM`（`-TERM`）请求进程有序退出，进程可以执行清理工作。这是默认的首选。
- `SIGKILL`（`-KILL`）由内核立即终止进程，进程没有机会清理文件、日志或运行状态。它是最后手段。

发送信号前的安全步骤：

1. `pgrep -af PATTERN` 核对命令行。
2. `ps -ef` 核对 PID、PPID 和所有者。
3. 先发 `TERM`。
4. 再次检查进程和端口是否消失。
5. 只在进程无法正常退出且确认目标无误时考虑 `KILL`。

### 7.5 `timeout`：限制命令运行时间

```bash
timeout 10s bash -lc '</dev/tcp/ark.cn-beijing.volces.com/443'
timeout 3s tail -f /tmp/sylar.log
```

`timeout DURATION COMMAND...` 在命令超过指定时间后发送信号。`10s` 是 10 秒；在 GNU coreutils 中，不写单位时默认也是秒。

命令因超时被停止时，`timeout` 默认常返回 `124`。它只是外部时间上限，不能替代 HTTP 客户端的连接超时、读超时等内部配置。

### 7.6 `/proc/<PID>/environ`：安全确认进程是否拿到环境变量

Linux 的 `/proc` 是内核暴露运行状态的特殊文件系统。某个进程启动时获得的环境变量可在 `/proc/<PID>/environ` 中查看，前提是当前用户有权限读取。

不要直接 `cat` 该文件，它可能包含密钥和令牌。下面的形式只输出“已设置”：

```bash
tr '\0' '\n' < /proc/<PID>/environ | sed -n 's/^ARK_API_KEY=.*/ARK_API_KEY=SET/p'
```

数据流是：

1. `< /proc/<PID>/environ` 读取指定进程的环境。
2. `tr '\0' '\n'` 把空字节分隔改为换行。
3. `sed` 只匹配 `ARK_API_KEY=` 开头的行，并把整个值替换成 `SET`。

这条命令只能证明进程环境中有该变量，不能证明密钥有效或有访问目标服务的权限。

## 8. HTTP、网络与 TLS 检查

排查网络问题时，不要把所有失败都叫作“网络不通”。可以从下到上分成六层：

```text
DNS 解析 → TCP 连接 → TLS 握手 → HTTP 状态 → 身份认证 → 业务响应
```

下面的外网命令会实际访问网络，结果受 DNS、代理、防火墙、证书和目标服务影响。学习时可先阅读参数，不必对真实服务执行。

### 8.1 `getent hosts`：查看系统解析到的地址

```bash
getent hosts ark.cn-beijing.volces.com
getent hosts localhost
```

`getent` 通过系统配置的 Name Service Switch 查询数据，`hosts` 表示查主机名。它会结合 `/etc/hosts`、DNS 等当前系统的实际解析路径。

能解析出 IP 只证明 DNS 层有结果，不代表目标端口可以连接。同一域名在不同时间或不同网络下解析到不同 IP 也不一定是异常。

### 8.2 Bash `/dev/tcp`：只验证 TCP 端口

```bash
timeout 10 bash -lc '</dev/tcp/ark.cn-beijing.volces.com/443'
echo $?
```

`/dev/tcp/HOST/PORT` 是 Bash 的特殊重定向能力，不是磁盘上的普通文件，也不是所有 Shell 都支持。该命令能在超时内成功打开连接，只能证明到目标 TCP 端口的路径可用；它没有验证 TLS 证书、HTTP 协议或鉴权。

### 8.3 `curl`：观察 HTTP 请求和响应

本地状态接口示例：

```bash
curl -sS --max-time 3 http://127.0.0.1:18080/internal/status
```

- `-s`：静默模式，不显示进度条。
- `-S`：即使开启 `-s`，失败时仍显示错误。因此常组合为 `-sS`。
- `--max-time 3`：限制整个 curl 操作最多 3 秒。

如果本机没有服务监听 `18080`，连接失败是正常现象，应先用 `ss -ltn` 确认端口。

查看详细连接、TLS 和 HTTP 信息：

```bash
curl -v --noproxy '*' --connect-timeout 10 https://ark.cn-beijing.volces.com/api/v3/
```

- `-v`：输出 DNS/IP、连接、TLS 和 HTTP 头等详细调试信息。调试输出可能包含请求头，分享日志前要检查是否有秘密。
- `--noproxy '*'`：对所有目标绕过 curl 代理配置，用于对比直连与代理路径；它不应被无条件加到所有请求中。
- `--connect-timeout 10`：限制建立连接阶段的等待时间。

未带鉴权信息时收到 HTTP `401` 是有价值的证据：通常说明 DNS、TCP、TLS 和 HTTP 层已走通，失败点在鉴权。这不是业务请求成功。

### 8.4 `curl --fail`：让 HTTP 4xx/5xx 体现为命令失败

```bash
curl --fail -sS --max-time 3 http://127.0.0.1:18080/demo
echo $?
```

curl 默认认为“成功收到 HTTP 响应”就是命令执行成功，即使状态码是 `404`。`--fail`可以让 HTTP `400` 以上的响应返回非 `0` 退出状态，适合脚本判断。

`--fail-with-body` 在较新 curl 中可以在失败时保留响应体，但使用前应通过 `curl --help all` 确认当前版本支持。

### 8.5 发送 POST、请求头和 JSON 请求体

仅在本地测试服务已启动时执行：

```bash
curl -sS --max-time 5 \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"model":"demo","messages":[{"role":"user","content":"hello"}]}' \
  http://127.0.0.1:19101/v1/chat/completions
```

- `-X POST`：显式指定 HTTP 方法。实际上使用 `-d` 时 curl 通常已会选择 POST，这里保留是为了便于阅读。
- `-H 'Name: Value'`：添加请求头。
- `-d '...'`：发送请求体。用单引号包围 JSON，避免双引号被 Shell 解释。
- 行末的 `\` 表示下一行仍属于同一条 Shell 命令；`反斜杠` 后不要再放空格。

不要把真实 `Authorization` 头写入文档、Shell 历史或可共享日志。

### 8.6 `openssl s_client`：独立检查 TLS

```bash
openssl s_client \
  -connect ark.cn-beijing.volces.com:443 \
  -servername ark.cn-beijing.volces.com \
  </dev/null
```

- `-connect HOST:PORT`：指定 TLS 服务器地址。
- `-servername HOST`：发送 SNI 服务器名，对同一 IP 上托管多个 HTTPS 域名很重要。
- `</dev/null`：让标准输入立即结束，避免 `s_client` 等待交互输入。

输出中可以观察证书主题、签发者、有效期和协商的 TLS 版本。仅看到 TCP 连接成功不等于证书校验成功；应同时查看验证结果，并了解 `s_client` 的默认验证行为会因 OpenSSL 版本和参数而异。

### 8.7 六层证据如何组合

| 层次 | 常用命令 | 成功能证明什么 | 不能证明什么 |
| --- | --- | --- | --- |
| DNS | `getent hosts` | 系统可以解析主机名 | 端口可连 |
| TCP | `/dev/tcp`、`curl --connect-timeout` | 到目标端口可建立连接 | TLS/HTTP 正常 |
| TLS | `openssl s_client`、`curl -v` | 能观察握手和证书 | HTTP 路由和鉴权正确 |
| HTTP | `curl -v`、`curl --fail` | 服务器返回 HTTP 响应 | 业务响应成功 |
| 鉴权 | HTTP `401/403` 与脱敏日志 | 失败已进入身份/权限层 | 凭证应被打印检查 |
| 业务 | 响应体、状态接口、trace | 请求进入应用逻辑 | 所有后续请求都会成功 |

## 9. CMake 编译与 CTest 测试

CMake 不是编译器，而是构建系统生成工具。CTest 是 CMake 配套的测试运行器。一条完整的开发证据链通常包含：生成构建系统、编译目标、运行测试。

### 9.1 `cmake -S . -B build`：配置构建目录

```bash
cmake -S . -B build
```

- `-S .`：源码目录是当前目录。
- `-B build`：将构建系统和中间文件生成到 `build/`。

修改 `CMakeLists.txt` 或新增目标后，应重新运行配置。排障记录中出现过 `No rule to make target 'test_http_client'`，重新配置后才能区分“构建目录还不知道新目标”与“源码本身无法编译”。

### 9.2 `cmake --build`：构建全部或指定目标

```bash
cmake --build build
cmake --build build --target test_http_connection
cmake --build build --target test_ai_gateway_servlet ai_gateway_module
```

- `--build build`：使用 `build/` 中已生成的构建系统。
- `--target NAME...`：只构建指定目标；可以在后面列出多个目标。

构建单个测试目标反馈更快，适合开发中小步验证。完整构建能发现其他库或模块的编译、链接问题。二者不能相互完全替代。

### 9.3 直接运行测试可执行文件

```bash
./bin/test_http_request_options
./bin/test_http_connection
```

`./` 表示从当前目录找可执行文件，而不是从 `PATH` 查找。构建通过只证明编译和链接成功，不证明测试行为正确；必须实际运行测试。

本仓库中：

- `test_http_request_options` 使用本地 loopback socket，受沙箱或容器网络权限影响。
- `test_http_connection` 会访问外网，受 DNS 和公网可用性影响。

遇到 `socket(...) errno=1 Operation not permitted` 时，应先判断是否为运行环境限制，而不是立即把它归因为 HTTP 业务逻辑错误。

### 9.4 `ctest`：运行已注册测试

```bash
ctest --test-dir build -N
ctest --test-dir build -L unit --output-on-failure
ctest --test-dir build -R ai_gateway_real_provider --output-on-failure
```

| 选项 | 用途 |
| --- | --- |
| `--test-dir build` | 在 `build/` 中查找 CTest 配置 |
| `-N` | 只列出测试，不执行 |
| `-L unit` | 只运行带 `unit` 标签的测试 |
| `-R REGEX` | 只运行名称匹配正则的测试 |
| `--output-on-failure` | 测试失败时显示它的输出 |

“目标已编译”、“CTest 能找到测试”和“测试执行通过”是三件不同的事。例如，一个 `bin/test_xxx` 可执行文件可能存在，但没有通过 `add_test(...)` 注册给 CTest；此时 `ctest -N` 不会列出它。

### 9.5 一条可复用的构建测试链

```bash
cmake -S . -B build
cmake --build build --target test_http_request_options
./bin/test_http_request_options
```

需要确保前一步成功才执行后一步时，可以用 `&&` 串联：

```bash
cmake --build build --target test_http_request_options && ./bin/test_http_request_options
```

学习阶段更建议先分开运行，这样能看清失败发生在配置、编译、链接还是测试运行阶段。

### 9.6 `gdb -batch`：在崩溃时收集调用栈

`.codex` 排障记录中使用过：

```bash
gdb -batch -ex run -ex bt --args ./bin/test_http_connection
gdb -batch -ex run -ex bt --args ./bin/sylar -s -c examples/ai_gateway_conf
```

- `-batch`：非交互批处理模式，执行完命令后退出。
- `-ex run`：启动被调试程序。
- `-ex bt`：输出当前调用栈 backtrace。
- `--args PROGRAM ARGS...`：指定程序及它的参数。

这条命令会真正启动程序，因此程序的网络请求、端口占用和文件写入仍会发生。调用栈是定位崩溃的证据，不是自动的根因结论；如果二进制缺少调试符号，函数名和行号信息可能不完整。

## 10. Git 只读检查命令

对 Git 不熟悉时，先掌握只读检查命令。先看清状态、差异和历史，再决定是否执行会改变仓库的命令。

### 10.1 四个基本概念

```text
工作区 → 暂存区 → 提交历史
                  ↑
                当前分支
```

- 工作区 working tree：磁盘上当前看到和编辑的文件。
- 暂存区 index/staging area：下一次提交准备包含的快照。
- 提交 commit：已写入 Git 历史的快照。
- 分支 branch：指向某个提交的可移动名称。

`git diff` 查看哪两个状态，取决于它的参数，不能笼统地理解为“查看所有改动”。

### 10.2 `git status --short`：快速查看工作区状态

```bash
git status --short
git status --short -- '文档/Linux常用命令学习手册.md'
```

`--short` 用两列状态符号输出简洁结果：第一列主要表示暂存区相对 `HEAD` 的状态，第二列主要表示工作区相对暂存区的状态。常见标记：

| 标记 | 含义 |
| --- | --- |
| `M` | 已修改 |
| `A` | 已添加到暂存区 |
| `D` | 已删除 |
| `??` | 未跟踪文件 |

`-- PATH` 中的 `--` 表示选项到此结束，后面是路径。当路径可能以 `-` 开头时，这能防止它被当作选项。

### 10.3 `git diff`：查看具体差异

```bash
git diff
git diff --name-status
git diff --check
git diff -- sylar/http/http_server.cc sylar/http/http_session.cc
git diff --cached
```

- `git diff`：查看工作区中尚未暂存的差异。
- `--name-status`：只列出路径和修改/新增/删除状态，适合快速确认范围。
- `--check`：检查尾随空格和其他补丁空白错误，通过时通常没有输出。
- `-- PATH...`：只查指定路径。
- `--cached`：查看暂存区相对 `HEAD` 的差异。

如果新文件还是 `??`，普通 `git diff` 不会显示它的内容，因为它还不在 Git 跟踪集中。可用 `git status --short` 发现它，然后直接阅读文件。

### 10.4 `git log` 与 `git show`：查看历史和某个提交

```bash
git log --oneline -10
git log --oneline master..feature/ai-gateway-g3
git show --stat HEAD
git show HEAD -- '文档/Linux常用命令学习手册.md'
```

- `git log --oneline -10`：每个提交显示一行，只看最近 10 个。
- `A..B`：在这个场景中，查看 B 可达但 A 不可达的提交。交换 A/B 会得到反方向结果。
- `git show --stat HEAD`：查看当前提交的摘要和文件变化统计。
- `git show COMMIT -- PATH`：查看某个提交中指定路径的补丁。

### 10.5 `git branch --show-current`：确认当前分支

```bash
git branch --show-current
```

它输出当前分支名。如果当前是 detached HEAD，可能没有输出。在修改、提交或推送前先确认分支，可以避免把工作写到错误位置。

### 10.6 `git worktree list`：查看多工作区

```bash
git worktree list
git worktree list --porcelain
```

Git worktree 允许同一个仓库同时有多个独立工作目录。`--porcelain` 输出稳定、适合脚本解析的格式，常见字段包括 `worktree`、`HEAD` 和 `branch`。

当一份修改“在另一个目录里看得到，当前目录里看不到”时，先用它确认是否在不同 worktree。

### 10.7 `git check-ignore`：解释文件为什么被忽略

```bash
git check-ignore -v .worktrees
git check-ignore -v build/CMakeCache.txt
```

`-v` 会同时输出命中的忽略规则来自哪个文件、第几行。如果路径没有被忽略，命令通常没有输出并返回非 `0`。

### 10.8 高风险 Git 命令：先知道边界，不作为入门练习

下面的命令在 `.codex` 排障记录中出现过，但它们会修改状态或删除引用，不要在不理解目标时复制：

| 命令 | 影响 | 执行前的只读检查 |
| --- | --- | --- |
| `git restore -- PATH` | 可能丢弃指定路径的未提交改动 | `git status --short -- PATH`、`git diff -- PATH` |
| `git reset` | 根据模式改变分支、暂存区或工作区 | `git log --oneline`、`git status --short`、阅读对应模式的手册 |
| `git branch -D NAME` | 强制删除分支引用，未合并提交可能难以找回 | `git log --oneline BASE..NAME` |
| `git worktree remove --force PATH` | 强制移除 worktree，其未提交改动可能丢失 | `git worktree list --porcelain`、`git -C PATH status --short` |

安全原则是：先用精确路径的 `status`、`diff`、`log` 解析影响，再由了解上下文的人明确决定是否执行。

## 11. 环境变量与当前 Shell 会话

环境变量是进程启动时获得的键值对。它们常用于传递路径、运行模式和密钥所在的变量名。最容易混淆的边界是：修改配置文件、修改当前 Shell，以及修改已经运行的子进程，是三件不同的事。

### 11.1 Shell 变量与已导出的环境变量

```bash
PRACTICE_MODE=enabled
printf '%s\n' "$PRACTICE_MODE"
export PRACTICE_MODE
```

- `NAME=value` 在当前 Shell 中设置变量。等号两边不能随意加空格。
- `export NAME` 将它标记为环境变量，使之可以被之后启动的子进程继承。
- 也可以一步写成 `export PRACTICE_MODE=enabled`。

变量展开时优先加双引号：

```bash
printf '%s\n' "$PRACTICE_MODE"
```

这能避免值中的空格或通配符被 Shell 再次分词和展开。

### 11.2 `printenv` 与 `env`：检查已导出的环境

```bash
printenv PRACTICE_MODE
env | rg '^PRACTICE_MODE='
```

- `printenv NAME` 只输出指定环境变量的值；未设置时通常无输出并返回非 `0`。
- 不带参数的 `env` 输出当前进程环境，其中可能包含敏感信息，不要整段复制到日志或聊天中。

只检查密钥是否存在，不打印密钥：

```bash
if printenv ARK_API_KEY >/dev/null; then
    printf 'ARK_API_KEY=SET\n'
else
    printf 'ARK_API_KEY=NOT_SET\n'
fi
```

这是一段 Bash 条件语句：`>/dev/null` 丢弃真实值，`if` 只根据 `printenv` 的退出状态选择脱敏输出。

### 11.3 只为一条命令提供临时环境

```bash
PRACTICE_MODE=enabled printenv PRACTICE_MODE
env PRACTICE_MODE=enabled printenv PRACTICE_MODE
```

这两种形式都可以让后面启动的命令看到临时值，而不必持久改变当前 Shell 环境。它适合测试某个开关或单次运行参数。

不要把真实密钥直接写在命令行中，因为它可能出现在 Shell 历史、进程信息或调试日志中。

### 11.4 `source`：在当前 Shell 中执行文件

```bash
source ~/.bashrc
```

`source FILE` 不是启动一个独立子进程，而是在当前 Shell 中读取并执行文件。因此文件中的 `export`、`cd`、alias 和函数定义会直接影响当前会话。

只对自己信任并阅读过的文件使用 `source`。`source ~/.bashrc` 可以让刚写入 Bash 配置的变量在当前交互式 Bash 中生效，但 `.bashrc` 中的条件逻辑可能使它在非交互环境下表现不同。

### 11.5 父进程、子进程和“当前终端为什么没变”

环境变量一般由父进程复制给新启动的子进程：

```text
当前 Bash --启动--> 子进程
      |
      +-- 子进程可继承已 export 的变量
      +-- 子进程不能反向修改已运行的父 Bash 环境
```

因此：

- 在 `~/.bashrc` 新增 `export` 后，已经打开的 Shell 需要 `source ~/.bashrc` 或重新打开合适的 Bash 会话。
- 修改当前 Shell 的环境不会自动改变已经运行的 Sylar 进程；需要在正确环境下重新启动它。
- 查当前 Shell 用 `printenv`；查已运行进程的启动环境，用第 7.6 节的脱敏 `/proc/<PID>/environ` 方法。

### 11.6 Windows `setx` 与 WSL 的边界

Windows 中的 `setx NAME VALUE` 修改的是 Windows 用户或系统环境的持久值，它不会反向更新：

- 已打开的 Windows 终端进程。
- 已运行的 WSL 实例和 Bash 会话。
- 已经启动的 Linux 子进程。

在 WSL 里排查时，先直接检查当前 Linux Shell：

```bash
printenv ARK_API_KEY >/dev/null && printf 'ARK_API_KEY=SET\n' || printf 'ARK_API_KEY=NOT_SET\n'
```

如果本来就在 WSL 中运行 Linux 程序，更清晰的做法通常是在可控的 Linux Shell 配置或启动脚本中设置环境，而不是假设 Windows `setx` 会立即进入当前 WSL 会话。

## 12. 安全边界、排障流程与练习

### 12.1 三级风险分类

在执行陌生命令前，先判断它属于哪一类：

| 等级 | 特征 | 典型命令 | 行动原则 |
| --- | --- | --- | --- |
| 低风险：只读检查 | 只收集信息，不主动修改目标 | `pwd`、`ls`、`sed -n`、`rg`、`ps`、`ss`、`git status`、`git diff` | 先用它们建立证据 |
| 中风险：改变局部状态 | 创建、覆盖、移动文件或改变当前会话 | `mkdir`、`cp`、`mv`、`chmod`、`export`、`>` | 将练习限定在明确的 `/tmp/linux-command-practice` 中，操作前检查目标 |
| 高风险：可能中断或丢失数据 | 强制终止进程、丢弃改动或删除 Git 引用 | `kill -KILL`、`git reset`、`git restore`、`git branch -D`、`git worktree remove --force` | 不作为入门练习；只在目标、影响和恢复方案都明确时执行 |

“只读”不等于“没有任何影响”。例如 `curl` 会向外部服务发送请求，`gdb -ex run` 会真正启动程序，`cmake --build` 会写构建产物。所以还要考虑网络、计费、端口和磁盘写入边界。

### 12.2 执行任何写操作前的四个问题

1. 目标是哪个精确文件、目录、PID、端口或 Git 分支？
2. Shell 会不会展开 `*`、`$VARIABLE` 或命令替换，使实际目标比眼前看到的更广？
3. 命令失败或在中间停止后，现场会变成什么状态？
4. 是否能先用 `ls`、`stat`、`ps`、`ss`、`git status`、`git diff` 等只读命令确认范围？

不要将未解析的环境变量、宽泛通配符或用户主目录作为递归删除、覆盖或强制操作的目标。

### 12.3 文件与源码排查流程

目标：不从头通读整个仓库，而是从文件地图和关键符号快速进入相关实现。

#### 第一步：确认当前目录

```bash
pwd
```

预期观察：如果是手册默认环境，应看到 `/home/muyang/workspace/sylar`。不一致时，后续相对路径可能失效，应先切换到正确目录。

#### 第二步：建立文件地图

```bash
rg --files sylar/http tests | sort
```

预期观察：输出 HTTP 实现与测试文件列表。这一步回答“有哪些文件”，还没有回答“哪个是真正调用点”。

#### 第三步：用函数名定位声明、定义和调用点

```bash
rg -n 'recvRequest|handleClient' sylar/http
```

预期观察：看到文件路径、行号和匹配内容。要区分头文件声明、源文件定义和其他位置的调用。

#### 第四步：阅读最小相关范围

```bash
sed -n '1,180p' sylar/http/http_server.cc
```

预期观察：查看类实现、日志和请求处理主线。如果关键符号不在该范围，应根据 `rg -n` 的行号调整范围，而不是盲目扩大到整个仓库。

#### 第五步：查看当前未提交差异

```bash
git diff -- sylar/http/http_server.cc sylar/http/http_session.cc
```

预期观察：只显示这两个已跟踪文件的未暂存改动。没有输出只能说明该比较范围内没有这类差异，不证明整个仓库干净。

### 12.4 服务与网络排查流程

目标：从本地进程逐层检查到远程 HTTP，在第一个失败层停下来分析。

#### 第一步：确认服务进程

```bash
pgrep -af 'bin/sylar'
```

预期观察：已启动时能看到 PID 和完整命令行；没有输出时先检查启动步骤和日志，不要继续假设本地 HTTP 端口已经存在。

#### 第二步：确认端口监听

```bash
ss -ltn
```

预期观察：查找配置的本地端口及绑定地址。进程存在但端口不存在，应检查配置、bind/listen 错误和启动日志。

#### 第三步：请求本地状态接口

```bash
curl -sS --max-time 3 http://127.0.0.1:18080/internal/status
```

预期观察：收到 HTTP 响应表示本地 TCP、HTTP Server 和该路由至少走到了返回响应的阶段。`404` 说明服务可达但路由未挂载，不等于端口不通。

#### 第四步：确认远程域名解析

```bash
getent hosts ark.cn-beijing.volces.com
```

预期观察：得到一个或多个地址。无输出或报错时，先排查 DNS 和系统网络配置。

#### 第五步：观察远程 TLS 和 HTTP

```bash
curl -v --connect-timeout 10 https://ark.cn-beijing.volces.com/api/v3/
```

预期观察：分别查看连接地址、TLS 握手、证书和 HTTP 状态。未带鉴权时收到 `401` 可以证明前四层已经工作，但不应为了“请求成功”就把真实密钥写到命令历史中。

如果某一层已失败，优先解释该层证据。例如 DNS 都没有解析结果时，修改 JSON 请求体不会解决问题。

### 12.5 可恢复练习

练习以只读命令为主；需要写文件时只使用 `/tmp/linux-command-practice`。“预期观察”描述应看到什么类型的证据，不要强求与某台机器的具体行数、PID 或时间戳相同。

#### 练习 1：确认位置并列出排障记录

```bash
pwd
rg --files .codex -g '*.md' | sort
```

预期观察：先看到仓库根目录，再看到 `.codex` 下按名称排序的 Markdown 文件。

#### 练习 2：找到 `handleClient` 并阅读周边源码

```bash
rg -n 'handleClient' sylar/http
nl -ba sylar/http/http_server.cc | sed -n '1,180p'
```

预期观察：`rg` 先给出声明/定义/调用所在路径和行号；`nl | sed` 显示带行号的源码范围。如果实际行号超出 180，根据搜索结果调整 `sed` 范围。

#### 练习 3：区分构建与测试

```bash
cmake --build build --target test_http_request_options
./bin/test_http_request_options
```

预期观察：第一条产生编译/链接证据；第二条产生行为测试证据。如果本地 socket 被环境禁止，要单独记录这个环境限制。

#### 练习 4：只读检查 Git 改动

```bash
git status --short
git diff --name-status
git diff --check
```

预期观察：第一条包含未跟踪路径；第二条列出已跟踪文件的未暂存状态；第三条只在有空白错误时输出诊断。三条都不会暂存或恢复文件。

#### 练习 5：在临时目录练习写入、追加、复制和移动

```bash
mkdir -p /tmp/linux-command-practice
printf 'alpha\n' > /tmp/linux-command-practice/notes.txt
printf 'beta\n' >> /tmp/linux-command-practice/notes.txt
cp /tmp/linux-command-practice/notes.txt /tmp/linux-command-practice/notes.copy.txt
mv /tmp/linux-command-practice/notes.copy.txt /tmp/linux-command-practice/notes.renamed.txt
stat -c '%y %n' /tmp/linux-command-practice/notes.txt /tmp/linux-command-practice/notes.renamed.txt
```

预期观察：`notes.txt` 有两行，重命名后存在 `notes.renamed.txt`；`stat` 显示两个文件的修改时间和路径。所有写入都限定在练习目录。

#### 练习 6：服务已启动时检查端口和本地 HTTP

```bash
ss -ltn
curl -sS --max-time 3 http://127.0.0.1:18080/internal/status
```

预期观察：只有已知本机有服务监听 `18080` 时才运行第二条。端口存在但返回 `404`，应查路由装配；端口不存在，应先查服务启动。

### 12.6 命令速查表

| 命令/符号 | 主要用途 | 常用形式 | 风险 | 详解 |
| --- | --- | --- | --- | --- |
| `man`、`--help` | 查看帮助 | `man sed`、`rg --help` | 低 | 第 2 章 |
| `type`、`which` | 确认命令来源 | `type -a printf`、`which cmake` | 低 | 第 2 章 |
| `pwd`、`cd`、`ls` | 导航和列目录 | `pwd`、`cd ..`、`ls -la` | `pwd/ls` 低，`cd` 改当前会话 | 第 3.1 节 |
| `find` | 按条件查找路径 | `find sylar -type f -name '*.cc'` | 低（本手册只讲查找） | 第 3.2 节 |
| `rg --files` | 列出仓库文件 | `rg --files .codex -g '*.md'` | 低 | 第 3.2 节 |
| `mkdir`、`cp`、`mv` | 创建、复制和移动 | `mkdir -p DIR`、`cp -i SRC DST`、`mv -i SRC DST` | 中 | 第 3.3 节 |
| `cmp` | 判断文件是否逐字节相同 | `cmp -s A B` | 低 | 第 3.4 节 |
| `stat`、`file` | 查元数据和文件类型 | `stat -c '%y %n' FILE`、`file FILE` | 低 | 第 3.5 节 |
| `readlink`、`realpath` | 解析链接和绝对路径 | `readlink -f LINK`、`realpath PATH` | 低 | 第 3.6 节 |
| `cat`、`less` | 查看文件 | `cat FILE`、`less FILE` | 低 | 第 4.1 节 |
| `head`、`tail` | 查看开头/结尾或跟踪日志 | `head -n 20 FILE`、`tail -f LOG` | 低 | 第 4.2 节 |
| `nl`、`sed -n` | 带行号查看指定范围 | `nl -ba FILE`、`sed -n '1,80p' FILE` | 低 | 第 4.3 节 |
| `rg`、`grep` | 搜索文本 | `rg -n PATTERN PATH`、`grep -Rni PATTERN PATH` | 低 | 第 4.4–4.5 节 |
| `wc`、`tr`、`sort` | 统计、字符转换和排序 | `wc -l FILE`、`tr ',' '\n'`、`sort` | 低 | 第 4.6 节 |
| `printf`、`echo` | 输出文本或状态 | `printf '%s\n' "$VALUE"`、`echo $?` | 低；与 `>` 组合时会写文件 | 第 1.4、5.3 节 |
| `|` | 把 stdout 传给下一命令 | `rg PATTERN PATH \| head` | 取决于两端命令 | 第 5.2 节 |
| `>`、`>>`、`<`、`2>&1` | 改变输入输出 | `CMD > LOG 2>&1` | `>` 可覆盖文件，中 | 第 5.3–5.4 节 |
| `&&`、`||` | 根据退出状态串联 | `A && B`、`A || B` | 取决于 A/B | 第 5.5 节 |
| `chmod` | 修改文件权限 | `chmod u+x FILE` | 中；递归宽泛修改可高风险 | 第 6.2 节 |
| `bash`、`bash -n` | 执行脚本/只查语法 | `bash SCRIPT`、`bash -n SCRIPT` | 执行脚本取决于内容；`-n` 低 | 第 6.3–6.4 节 |
| `ps`、`pgrep` | 查进程 | `ps -ef`、`pgrep -af PATTERN` | 低 | 第 7.1–7.2 节 |
| `ss` | 查端口和套接字 | `ss -ltn` | 低 | 第 7.3 节 |
| `kill` | 向进程发信号 | `kill -TERM PID` | 高，`KILL` 更高 | 第 7.4 节 |
| `timeout` | 限制命令时间 | `timeout 10s COMMAND` | 中，会向超时进程发信号 | 第 7.5 节 |
| `/proc/<PID>/environ` | 查已运行进程的启动环境 | `tr '\0' '\n' < ...` 后立即脱敏 | 敏感读取，不应直接打印 | 第 7.6 节 |
| `getent hosts` | 按系统配置解析主机名 | `getent hosts HOST` | 低 | 第 8.1 节 |
| Bash `/dev/tcp` | 尝试 TCP 连接 | `timeout 10 bash -lc '</dev/tcp/HOST/PORT'` | 发起网络连接 | 第 8.2 节 |
| `curl` | 发 HTTP(S) 请求 | `curl -sS --max-time 3 URL` | 发起网络请求，可能含敏感头 | 第 8.3–8.5 节 |
| `openssl s_client` | 观察 TLS 握手和证书 | `openssl s_client -connect HOST:443 -servername HOST` | 发起网络连接 | 第 8.6 节 |
| `cmake` | 配置和构建 | `cmake -S . -B build`、`cmake --build build --target NAME` | 写构建目录 | 第 9.1–9.2 节 |
| `ctest`、`./bin/test_*` | 列出或运行测试 | `ctest --test-dir build -N`、`./bin/test_NAME` | 测试可能开端口或访问网络 | 第 9.3–9.4 节 |
| `gdb` | 调试并收集调用栈 | `gdb -batch -ex run -ex bt --args PROGRAM` | 会真正启动程序 | 第 9.6 节 |
| `git status/diff/log/show` | 只读检查状态、差异和历史 | `git status --short`、`git diff --check`、`git log --oneline` | 低 | 第 10.2–10.4 节 |
| `git branch/worktree/check-ignore` | 检查分支、worktree 和忽略规则 | `git branch --show-current`、`git worktree list --porcelain` | 表中的查询形式低 | 第 10.5–10.7 节 |
| `git restore/reset/branch -D/worktree remove --force` | 恢复、重置或强制删除引用 | 本手册不提供练习 | 高 | 第 10.8 节 |
| `export`、`env`、`printenv` | 设置或查看环境 | `export NAME=value`、`printenv NAME` | `printenv NAME` 低；全量 `env` 可泄露秘密 | 第 11.1–11.3 节 |
| `source` | 在当前 Shell 执行文件 | `source ~/.bashrc` | 取决于文件内容，可改当前会话 | 第 11.4 节 |

### 12.7 学习顺序建议

1. 先掌握 `pwd`、`ls`、`cd`、`man`、`type`，能确认当前位置和命令来源。
2. 再掌握 `rg --files`、`rg -n`、`sed -n`、`less`，能快速找到并阅读文件。
3. 接着练习 `|`、`>`、`>>`、`&&`、`||`，学会拆解数据流和退出状态。
4. 然后学习 `ps`、`pgrep`、`ss`、`curl`，按层收集运行时证据。
5. 最后再将 CMake/CTest、Git 和环境变量放进自己的项目工作流。

真正熟练命令的标志，不是能背出所有选项，而是能回答三个问题：这条命令收集什么证据？它会改变什么状态？它的成功与失败分别能证明什么？
