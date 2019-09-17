# Reed-solomon codes

纠删码，可以允许数据丢失，但无法容忍数据篡改

假设数据块数量为n，校验块数量为m，恢复时任取n块都可以，最大容忍丢失m块。

## 校验码计算

假设输入数据为D = (D1, D2, ..., Dn)，生成的校验码为P = (P1, P2, ..., Pm)

$$
\left\{
\begin{matrix}
    1 & 0 & 0 \\
    0 & 1 & 0 \\
    0 & 0 & 1 \\
    B_{11} & B_{12} & B_{13} \\
    B_{21} & B_{22} & B_{23} \\
\end{matrix}
\right\}

\times

\left\{
\begin{matrix}
    D_1 \\
    D_2 \\
    D_3
\end{matrix}
\right\}

=

\left\{
\begin{matrix}
    D_1 \\
    D_2 \\
    D_3 \\
    P_1 \\
    P_2
\end{matrix}
\right\}
$$

单位阵是为了减少计算，B是Vandermode矩阵

$$
\left\{
\begin{matrix}    
    1 & 1 & 1 & \cdots & 1 \\
    1 & 2 & 3 & \cdots & n \\
    \vdots & \vdots & \vdots & \ddots & \vdots \\
    1 & 2^{m-1} & 3^{m-1} & \cdots & n^{m-1}
\end{matrix}
\right\}
$$

这个设定是为了保证编码矩阵任意n行都是可逆的

## 数据恢复

获取任意n块数据，同时根据对应的行构造编码矩阵，计算编码矩阵的逆，左乘该逆即可恢复数据。

## 特点

+ 低冗余度，高可靠性
+ 数据恢复代价高，要读取n块，总量其实是不变，但n块通常分布在不同地方
+ 数据更新代价高，更新相当于重新编码

## Reference

(Reed Solomon Tutorial: Backblaze Reed Solomon Encoding Example Case)[https://www.youtube.com/watch?v=jgO09opx56o]
