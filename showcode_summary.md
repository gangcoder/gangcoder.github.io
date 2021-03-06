<!-- ---
title: showcode 方法与总结
date: 2020-03-13 14:31:32
category: showcode
--- -->

# showcode 方法与总结

## showcode 方法论

### showcode 内容组织

一个showcode 阅读项目需要完成的内容事项。

1. 序言，介绍内容组织形式，使用的资料，学习建议等
2. 软件使用指南，介绍软件功能
3. 安装与启动，包括常规使用命令
4. code 代码量，代码组织情况，软件功能架构
5. 结构图，包括软件整体代码结构图，以及对整体图的细化讲解
6. 常见如client 客户端入口及代码实现逻辑
7. server 服务端入口及代码实现逻辑


### 阅读和深入步骤

阅读资料步骤：

1. 阅读文档了解功能：中文或者英文文档，产出软件序言，使用指南，代码量情况
2. 使用软件熟悉功能：搭建软件使用环境和编译环境，产出安装与启动文档，完善代码量，结构图草图
3. 按照功能阅读代码：按照从客户端到服务端，从总体再到具体的顺序阅读代码，先阅读，再绘制草图，最后整理阅读文章和电子结构图，整理软件各部分笔记

具体步骤：

1. 阅读中英文文档，了解软件基本架构和使用方式
2. 搭建软件环境，将软件涉及的客户端，服务端搭建起来，结合官方文档，使用文档，操作软件
3. 对软件功能熟悉
4. 搭建软件编译环境，下载依赖等
5. 根据前面对软件的了解，先客户端，再服务端的顺序，第一次阅读源代码
6. 不清楚的地方，使用断点调试对代码进行调试
7. 整理软件整体架构文档，包含粗粒度逻辑代码实现，以及各部件之间的架构图绘制
   1. 绘图至少需要 4个版本才能定稿
   2. 第一版本大致描述清楚大概结构，不追求精细和完善，重点在于快速把整体逻辑表示出来
   3. 第二版在第一版的基础上，对绘图布局，组件位置，结合代码理解进行重新排列整理
   4. 第三版绘图可以进行电子化绘图
   5. 第四版绘图是整理各部分子图，汇集成一张完整的架构大图
8. 第二次阅读源代码，不熟悉的地方多调试
9.  补充完善文档内容和描述，完善结构绘图细节

代码分析时，产生结构图，重点注释清除每个目录和文件的作用，实现了什么逻辑。

源码阅读原则：
1. 先抽象，先总体
2. 再具体，某个细节
3. 展示重点，省略局部
4. 体现逻辑

## 逻辑图绘制

### 规范

- 线宽选择 1px
- 连线选择圆角 12px
- 一行字是框图高度 60px
- 红 254 67 101 FE4365
- 蓝 1 158 213  019ED5
- 绿 29 191 151 1DBF97
- 黑 35 31 32
- 标题字号 18px
- 文本字号 13px
- 字体 source code pro

- 红: 表示实现步骤，实现逻辑
- 蓝: 实现，接口到结构的具体实现，提前调用
- 绿: 继承，内嵌，使用，类型指示
- 蓝虚：表示内部存在联系或者 golang channel 使用

## showcode golang 项目应该具有完备性
  
- gorm
- redisgo
