## 常规命令
### mv
在 Linux 中，mv 命令用于移动文件/目录或重命名它们，是日常文件管理中非常常用的工具。
```bash
# 将文件 file1 移动到 /home/user 目录
mv file1 /home/user/
# 将文件夹 old_name 重命名为 new_name
mv old_name new_name
```
>常用选项
>-i：覆盖前询问确认
>-f：强制覆盖，不提示
>-n：不覆盖已存在文件
>-u：仅在源文件较新或目标不存在时移动

### unzip
- 解压到当前目录
```bash
unzip test.zip
```
- -d：解压到指定目录
```bash
unzip -d /temp test.zip
```
- -n：避免覆盖现有文件
- -o：覆盖现有文件

### ln
创建软链接
```bash
ln -s file1.txt link_to_file.txt
```

### wget
- -c：当下载中断时可以从中断处恢复，适合大文件（以及垃圾网络）
```bash
wget -P /home/user/downloads https://example.com/file.zip
```
- -O：指定保存文件名
```bash
wget -O myfile.zip https://example.com/file.zip
```
- -P：指定保存目录
```bash
wget -P /home/user/downloads https://example.com/file.zip
```
## conda
1. 创建虚拟环境
```bash
conda create -n xxx python=3.x
```
2. 进入虚拟环境
```bash
source activate xxx
# or
conda activate xxx
```
3. 退出虚拟环境
```bash
conda deactivate
```
4. 删除虚拟环境
```bash
conda remove -n xxx --all
```
5. 查看本地虚拟环境
```bash
conda info --env
```

## tmux
作用：
1. 即使 SSH 连接断开，正在终端里运行的程序也不会中断。
2. 窗口管理。
### 日常高频操作命令大全
注：prefix为Ctrl+b

`tmux` 层级关系是：**Session (会话) -> Window (窗口) -> Pane (面板)**。

#### 1. Session (会话) 管理：用于保护任务

当你需要跑一个长时间任务时，通常会新建一个 Session。

| 目标                  | 终端命令行输入                      | 快捷键操作 (在 tmux 内部) |
| ------------------- | ---------------------------- | ----------------- |
| **新建会话**并命名         | `tmux new -s <会话名>`          | 无                 |
| **暂时离开** (任务继续后台运行) | `tmux detach`                | `Prefix` + `d`    |
| **查看**所有后台会话        | `tmux ls`                    | `Prefix` + `s`    |
| **重新连接**到某个会话       | `tmux a -t <会话名>`            | 无                 |
| **彻底关闭**某个会话        | `tmux kill-session -t <会话名>` | 无                 |

#### 2. Pane (面板) 管理：用于屏幕分屏

这是在日常敲代码或看日志时最常用的功能。

| 目标 | 快捷键操作 (在 tmux 内部) |
| --- | --- |
| **左右**分屏 (垂直) | `Prefix` + `%` |
| **上下**分屏 (水平) | `Prefix` + `"` (双引号) |
| **切换**当前激活的面板 | `Prefix` + `↑ ↓ ← →` (方向键) |
| **放大/缩小**当前面板 (全屏查看) | `Prefix` + `z` (再次按恢复原状) |
| **关闭**当前面板 | `Prefix` + `x` (或直接输入 `exit` 回车) |
| 显示面板编号并跳转 | `Prefix` + `q` (然后输入屏幕上出现的数字) |

#### 3. Window (窗口) 管理：类似浏览器的标签页

如果面板分得太多屏幕太挤，你可以新建窗口。

| 目标 | 快捷键操作 (在 tmux 内部) |
| --- | --- |
| **新建**一个窗口 | `Prefix` + `c` |
| **切换**到上/下一个窗口 | `Prefix` + `p` (上一个) / `n` (下一个) |
| **跳转**到指定编号的窗口 | `Prefix` + `0~9` 的数字 |
| **重命名**当前窗口 | `Prefix` + `,` (逗号) |
| **关闭**当前窗口 | `Prefix` + `&` |

---

### 💡 新手强推：开启鼠标支持

默认情况下 `tmux` 只能用键盘操作。开启鼠标支持后，你可以直接用鼠标点击切换面板、拖拽调整面板大小、以及使用滚轮查看历史输出。

1. 在终端输入 `nano ~/.tmux.conf` 创建或编辑配置文件。
2. 将以下内容粘贴进去：
```text
set -g mouse on
```
3. 保存退出（`Ctrl+O` 回车，`Ctrl+X`）。
4. 在终端运行以下命令让配置生效：
```bash
tmux source-file ~/.tmux.conf
```

## 其他
1. 获取dsh网址：
```bash
journalctl --user -u dsh-web.service -n 30 --no-pager | grep -o 'http://127.0.0.1:3080/?token=<省略>' | tail -1
```