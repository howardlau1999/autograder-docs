# Docker 镜像构建指引

在学生上传提交自己的代码文件后，Autograder 将启动作业所指定的 Docker 镜像，并将学生提交的文件挂载到 `/autograder/submission` 目录下。之后，Autograder 将在 `/autograder` 目录下运行 `/autograder/run` 文件，这个文件可以是 Bash 脚本，也可以是 Python 脚本，只要是一个可执行文件即可，如果是脚本文件，需要使用 `#!` 语法来指定脚本解释器的目录。在 `/autograder/run` 程序退出后，Autograder 将读取解析 `/autograder/results/results.json` 文件，并将结果以及程序返回码写回数据库。

其中，`results.json` 文件样例如下：

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
            "isDesc": true, // 排名分数项的排序规则，true 为降序，false 为升序，默认为 false
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