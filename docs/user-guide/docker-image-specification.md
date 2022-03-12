# Docker 镜像构建指引

## 评测过程简介

在学生上传提交自己的代码文件后，Autograder 将启动作业所指定的 Docker 镜像，并将学生提交的文件挂载到 `/autograder/submission` 目录下。之后，Autograder 将在 `/autograder` 目录下运行 `/autograder/run` 文件，这个文件可以是 Bash 脚本，也可以是 Python 脚本，还可以是二进制程序，只要是一个可执行文件即可。如果是脚本文件，需要使用 `#!` 语法来指定脚本解释器的目录。在 `/autograder/run` 程序退出后，Autograder 将读取解析 `/autograder/results/results.json` 文件，并将结果以及程序返回码写回数据库。

## 评测结果格式

其中，`results.json` 文件格式以及样例如下：

```json
{
    "tests": [ // 测试结果，由多个测试用例构成
        {
            "name": "测试用例 1", // 测试用例的名字
            "score": 10, // 测试用例的分数
            "max_score": 20, // 测试用例的满分
            "output": "Any output message", // 希望学生看到的输出信息（可选）
            "output_path": "path/to/output/file" // 希望学生下载的输出文件，需要在 /autograder/results/outputs 目录下（可选）
        },
        {
            "name": "测试用例 2", // 测试用例的名字
            "score": 20, // 测试用例的分数
            "max_score": 20, // 测试用例的满分
            "output": "Any output message", // 希望学生看到的输出信息（可选）
            "output_path": "path/to/output/file" // 希望学生下载的输出文件，需要在 /autograder/results/outputs 目录下（可选）
        },
    ],
    "leaderboard": [ // 排行榜信息，将用于排行榜的显示（可选）
        {
            "name": "准确率", // 排名分数项的名字
            "value": 23.33, // 排名分数项的值，可以为数字或字符串，字符串按字典序比较
            "order": 1, // 排名分数项的顺序，为整数，数字越小优先级越高，默认以及最小值为 0，多个顺序重复时，排序结果未定义
            "is_desc": true, // 排名分数项的排序规则，true 为降序，false 为升序，默认为 false
            "suffix": "%", // 排名分数项的单位后缀，为字符串（可选）
        },
        {
            "name": "运行时间",
            "value": 126,
            "order": 2,
            "suffix": "ms"
        }
    ]
    // 排行榜如果为空，则该提交无法获得排行榜排名
    // 对于所有提交而言，如果排行榜不为空，则排行榜分数项的名字、顺序和排序规则应当相同
    // 如果有分数项定义不一致，则排序结果未定义
}
```

无论测试是否出错，都应该生成这样的报告文件，否则将视为发生了内部错误。

Docker 镜像需要您在本地构建好之后上传至 Docker Hub，并确保 Autograder 有权限访问该镜像。镜像可以指定 tag，以方便复用镜像。如果您的测试需要使用额外的数据，也需要打包在 Docker 镜像中一并上传。Docker 镜像本身有复用机制，通过恰当编写 Dockerfile 文件，即使不同作业使用不同的镜像，也不会带来额外的存储开销。

## 运行脚本参考样例

!!!warning "注意换行符"
    如果您在 Windows 平台上编写脚本，需要注意保存的时候要**使用 `LF` 换行符**而不是 `CRLF` 换行符，否则会导致脚本无法运行。

```python
#!/usr/bin/env python3

import json
import subprocess
import shutil

class TestAborted:
    pass

if __name__ == "__main__":
    report = {}
    report["tests"] = []
    report["leaderboard"] = []

    # Copy files from the submission to working directory
    shutil.copy2("/autograder/submission/hello.cpp", "./hello.cpp")
    shutil.copy2("/autograder/submission/world.cpp", "./world.cpp")
    try:
        # Compile
        result = subprocess.run(["g++", "-o", "hello_world", "hello.cpp", "world.cpp"], capture_output=True)
        compile_case = {"name": "编译", "max_score": 20, "score": 0}
        if result.returncode != 0:
            compile_case["output"] = result.stdout
            report["tests"].append(compile_case)
            raise TestAborted
        compile_case["score"] = 20
        report["tests"].append(compile_case)

        # Run
        run_case = {"name": "运行测试", "max_score": 80, "score": 0}
        result = subprocess.run(["./hello_world"], capture_output=True)
        if result.stdout == "Hello, world!":
            run_case["score"] = 80
        else:
            run_case["output"] = result.stdout
        report["tests"].append(run_case)

        # Leaderboard
        statinfo = os.stat("hello.cpp")
        filesize = statinfo.st_size
        leaderboard_item = {"name": "代码大小", "value": filesize, "suffix": "B"}
        report["leaderboard"].append(leaderboard_item)
    except:
        pass

    # Write report
    with open("/autograder/results/results.json", "w") as f:
        json.dump(report, f)
```

## Dockerfile 参考样例

```dockerfile
# 指定基础镜像
FROM ubuntu:20.04
# 安装需要的包
RUN apt update && apt install -y g++ python3
WORKDIR /autograder
# 添加评测所需要的文件
ADD run.py /autograder/run
```

将需要放入的文件和 `Dockerfile` 保存在一起，参考以下构建指令：

```bash
docker build -t howardlau1999/my-assignment:1.0 .
```

构建完成后，推送到 DockerHub：

```bash
docker push howardlau1999/my-assignment:1.0
```

之后创建作业的时候填入同样的镜像名即可。