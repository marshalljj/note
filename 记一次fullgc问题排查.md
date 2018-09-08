# 记一次fullgc问题排查

## 问题场景
线上cpu飙高（198%）。

## 排查步骤
1. 找出cpu使用率最高的线程: top -> gc
2. 查看gc情况：jstat -gcutil -> 频繁fullgc
3. 查看堆内存统计: jmap -histo:live -> 找出内存泄漏的类。
4. 查看相关代码

## note
其中步骤3找出占用内存高的类都是系统类(eg. LinkedList.Node), 无法确定此类被哪里引用。需要过滤出**应用相关类**， DefaultRow。
