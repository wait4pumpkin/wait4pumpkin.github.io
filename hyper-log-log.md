# HyperLogLog算法

## 基数统计

统计集合中不重复的元素个数。要准确计算必定就要耗内存，如果不要求准确，就可以考虑使用概率算法。

## 基本原理

假设进行多组伯努利实验，组数为n。在每组实验中，当结果出现1时，停止该组实验，设该组的实验次数为kn。其中实验次数最多的为kmax。

根据最大似然估计，n = 2 ^ kmax。

如果只跑一轮，可能误差会比较大，所以跑m轮，再取调和平均数。

```
HLL = constant * m * 2 ^ R
```

constant是一个修正系数，根据m而定。

## 实际使用

把输入参数进行hash，转换为比特串。比特串可以拆为高低两部分，低位可用于选择第几组。高位第n为出现1，可以认为是k。

## Reference

[Sketch of the Day: HyperLogLog — Cornerstone of a Big Data Infrastructure](https://research.neustar.biz/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/)
