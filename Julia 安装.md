## 初识 Julia Pluto 

**什么是Pluto**

https://github.com/fonsp/Pluto.jl

写笔记不仅仅是编写最后的文档，Pluto 还提供了过程中必需的实验和探索

Pluto 提供了 在笔记中  **探索模型** 和 **分享结果** 的能力

- **反应性**-更改功能或变量时，Pluto会自动更新所有受影响的单元格。 
- **轻量**-冥王星是用纯Julia编写的，易于安装。 
- **简单**-没有隐藏的工作空间状态；友好的用户界面。

**安装 Pluto ** 

命令行下按 `]` 进入REPL环境

```
add Pluto
Installing known registries into `C:\Users\liuyh\.julia`
```

一直卡在这里，没有其他日志

怀疑是 github 下载速度的问题

**离线安装**

```
git clone https://github.com/fonsp/Pluto.jl
add E:\GitHub\Julia\Pluto.jl
```

问题：

```
ERROR: SystemError: opening file "C:\\Users\\liuyh\\.julia\\registries\\General\\Registry.toml": No such file or directory
```

解决：

删除  C:\\Users\\liuyh\\.julia\\registries 文件夹，重新安装

原因：

add Pluto 远程安装时已经有写入一些内容，与当前安装产生冲突

**启动**

`Backspcae` 退出REPL

```
using Pluto
Pluto.run()
```

<img src="C:\Users\liuyh\AppData\Roaming\Typora\typora-user-images\image-20200921101858846.png" alt="image-20200921101858846" style="zoom:50%;" />

**HelloWorld**

<img src="C:\Users\liuyh\AppData\Roaming\Typora\typora-user-images\image-20200921105410732.png" alt="image-20200921105410732" style="zoom:50%;" />

- cell 是一个 代码+输出 的容器, 可以
  - 添加，
  - 删除，
  - 运行，所见即所得
- 多行 cell,  
  - 要么拆分成多个cell,
  - 要么使用begin..end 行为代码块