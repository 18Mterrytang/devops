## 程序功能
分析Linux服务器 IO 进程，分别按 `读` 和 `写` 的进程排序，`默认显示前5行`。功能类似 Linux Shell `pidstat`命令。

## 程序输出结果
![](/img/io.png)

 ## 程序环境
 - Python3+
 - 安装 Python prettytable 插件

 ## 运行示例
 ```bash
 # 如果不接参数，默认是等待5秒，打印前6个进程，脚本运行一次
$ io_difference_analysis3.py 4 5 3
 ```
- 第一个数位每次收集读写数据的间隔秒数
- 第二个数是打印出读写最多的n个进程
- 第三个为运行脚本的次数

## 程序部分代码
下面是程序部分代码，获取完整代码请关注微信公众号 `YP小站` ,并回复 `获取IO分析代码`

```python
#!/usr/bin/python3
# -*- coding:utf-8 -*-

import argparse
import os
import re
import sys
import time

from prettytable import PrettyTable


SYS_PROC_PATH = "/proc/"
RE_FIND_PROCESS_NUMBER = r"^\d+$"
DEFAULT_SLEEP_TIME = 1
DEFAULT_LIST_NUM = 5


def collect_info():
    process_info = {}
    pid_pattern = re.compile(RE_FIND_PROCESS_NUMBER)

    for entry in os.listdir(SYS_PROC_PATH):
        if not pid_pattern.match(entry):
            continue

        pid = entry
        stat_path = os.path.join(SYS_PROC_PATH, pid, "stat")
        io_path = os.path.join(SYS_PROC_PATH, pid, "io")

        try:
            with open(stat_path, "r", encoding="utf-8") as stat_file:
                process_name = stat_file.read().split(" ")[1]

            read_bytes = 0
            write_bytes = 0
            with open(io_path, "r", encoding="utf-8") as io_file:
                for line in io_file:
                    key, value = [item.strip() for item in line.split(":", 1)]
                    if key == "read_bytes":
                        read_bytes = int(value)
                    elif key == "write_bytes":
                        write_bytes = int(value)

            process_info[pid] = {
                "name": process_name,
                "read_bytes": read_bytes,
                "write_bytes": write_bytes,
            }
        except (FileNotFoundError, PermissionError, ProcessLookupError, IndexError, ValueError):
            continue

    return process_info


def build_diff(first_snapshot, second_snapshot):
    read_diff = {}
    write_diff = {}

    for pid, second_data in second_snapshot.items():
        first_data = first_snapshot.get(pid, {})
        read_diff[pid] = second_data["read_bytes"] - first_data.get("read_bytes", 0)
        write_diff[pid] = second_data["write_bytes"] - first_data.get("write_bytes", 0)

    return read_diff, write_diff


def format_process_name(process_name):
    return process_name.strip("()")


def build_table(first_snapshot, second_snapshot, top_n):
    read_diff, write_diff = build_diff(first_snapshot, second_snapshot)
    sorted_read = sorted(read_diff.items(), key=lambda item: item[1], reverse=True)[:top_n]
    sorted_write = sorted(write_diff.items(), key=lambda item: item[1], reverse=True)[:top_n]

    table = PrettyTable(
        ["r-pid", "r-process", "read(bytes)", "w-pid", "w-process", "write(bytes)"]
    )
    table.align = "l"
    table.padding_width = 1

    max_rows = max(len(sorted_read), len(sorted_write))
    for index in range(max_rows):
        if index < len(sorted_read):
            r_pid, r_value = sorted_read[index]
            r_name = format_process_name(second_snapshot[r_pid]["name"])
            read_row = [r_pid, r_name, r_value]
        else:
            read_row = ["", "", ""]

        if index < len(sorted_write):
            w_pid, w_value = sorted_write[index]
            w_name = format_process_name(second_snapshot[w_pid]["name"])
            write_row = [w_pid, w_name, w_value]
        else:
            write_row = ["", "", ""]

        table.add_row(read_row + write_row)

    return table


def parse_args():
    parser = argparse.ArgumentParser(
        description="Analyze Linux process IO and display top read/write processes."
    )
    parser.add_argument(
        "-s",
        "--sleep",
        type=float,
        default=DEFAULT_SLEEP_TIME,
        help="Sampling interval in seconds, default: %(default)s",
    )
    parser.add_argument(
        "-n",
        "--top",
        type=int,
        default=DEFAULT_LIST_NUM,
        help="Number of rows to display, default: %(default)s",
    )
    return parser.parse_args()


def main(sleep_time, list_num):
    if sleep_time <= 0:
        raise ValueError("sleep time must be greater than 0")
    if list_num <= 0:
        raise ValueError("top number must be greater than 0")

    first_snapshot = collect_info()
    time.sleep(sleep_time)
    second_snapshot = collect_info()

    print(time.strftime("%F %T"))
    print("sample_interval: %.2fs, top_n: %s" % (sleep_time, list_num))
    print(build_table(first_snapshot, second_snapshot, list_num))


if __name__ == "__main__":
    try:
        args = parse_args()
        main(args.sleep, args.top)
    except KeyboardInterrupt:
        print("\ninterrupted by user")
        sys.exit(130)
    except Exception as exc:
        print("error: %s" % exc, file=sys.stderr)
        sys.exit(1)
```
