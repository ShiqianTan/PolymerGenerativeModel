# polyGen：固定化学连接下的三维周期链构象生成

**论文**：Jain et al., *A Learning Framework for Atomic-Level Polymer Structure Generation*, Chemistry of Materials 37, 7337–7346 (2025)。综述 [64] 与 [67] 为同一工作，去重处理。[预印本入口](https://arxiv.org/abs/2504.17656)，[作者期刊 PDF](https://ramprasad.mse.gatech.edu/wp-content/uploads/2025/09/jain-et-al-2025-a-learning-framework-for-atomic-level-polymer-structure-generation.pdf)。

**可读版本**：[arXiv 预印本](https://arxiv.org/abs/2504.17656)。本笔记的公式与训练数字来自该版本；其作者名单与正式版不同，不能混作同一版本的逐项设置。[代码](https://github.com/Ramprasad-Group/polyGen)。

![polyGen](../figures/polyGen.png)

## 首先弄清任务

输入重复单元的原子类型、规定连接与周期性，输出原子三维位置和周期方向盒长的构象集合。化学结构是条件，不是待生成的新分子式。这也不是熔体全原子模拟，不直接生成完整非晶形貌或热平衡 Boltzmann 分布。

综述将它放在 VAE 段落。准确结构是：**条件图编码 + 结构 VAE + 潜空间 DiT；生成模块用 Gaussian flow matching**。VAE 是第一阶段表示学习，不是整个生成算法。[预印本图 2、4.3–4.4]

## 数据与预处理

polyChainStructures 有 3855 个 DFT 优化无限单链结构，最多 208 原子，含 H。划分 3084/386/385。联合训练增加 QM9 的 100,000 个分子训练样本，聚合物每 epoch 上采样 30 倍；仅聚合物训练的对照也保留这一上采样率。每步随机旋转和平移是数据增强，不等价于架构严格等变。[2.1、4.6]

原子具有分数坐标 f、笛卡尔坐标 p、条件向量 C；聚合物需将盒长、坐标缩放与周期跨边界连接一起保存。QM9 无周期结构，仅提供笛卡尔位置。链结构数据可从 [Khazana](https://khazana.gatech.edu/dataset) 获取。

## 第一阶段：结构 VAE

条件编码器 Ec 从化学图构造每个原子 Ci；E(f,p,C) 输出每原子 μ、σ，采样 Zi；D(Z,C) 还原 fhat、phat、盒长 b̂z。这里是逐原子 latent，不能换成一个全图平均向量，否则丢失可变原子数对应关系。

$$L_{AE}=w_bL_{box}+w_fL_{frac}+w_pL_{pos}+w_{KL}L_{KL}+w_dL_{bond}+w_aL_{angle}+w_\tau L_{dihedral}.$$

盒长与位置等采用 MSE；笛卡尔位置先去中心；键长、角度做几何监督；二面角使用周期角差，而不是直接相减。预印本盒长目标含按原子数 N 的缩放 b_z/(10 N^(1/3))，移植时必须复核单位与实现。所有权重应从最终配置读取，不自行设为 1。[式 8]

## 第二阶段：Gaussian flow matching DiT

设真实结构 latent 为 Z1，噪声 Z0~N(0,I)，随机 t~U(0,1)：

$$Z_t=(1-t)Z_0+tZ_1,\quad u_t=\frac{Z_1-Z_t}{1-t}=Z_1-Z_0.$$

网络预测 Ẑ1，再得到 ût=(Ẑ1−Zt)/(1−t)。损失是逐原子的速度 MSE：

$$L_G=\frac1N\sum_i\|u_t^{(i)}-\hat u_t^{(i)}\|^2
=\frac{1}{(1-t)^2N}\sum_i\|Z_1^{(i)}-\hat Z_1^{(i)}\|^2.$$

训练将 t 截至 0.9 以避免数值不稳定。条件包括图化学 Ci、时间、数据来源 embedding S，以及 self-conditioning Zsc。先做一次预测，再把它作为条件做第二次；训练以概率 0.5 丢弃 self-conditioning。[4.4、式 9–13]

## 注意力的具体约束

通过原子对的图距离分五类：同一原子、直接成键、隔一个角、隔一个二面角、更远距离。one-hot → MLP → 标量 bias，加入注意力打分。这个“位置”指图相对关系，不是 SMILES 序号或普通绝对坐标编码。[4.4.1]

## 训练、采样与判定

结构 VAE 300 epochs，取验证最优 checkpoint；DiT 最多 500 epochs。采样从 Z0 出发，T 步 Euler 更新 Zt+Δt=Zt+Δt·ût，Δt=1/T，再用冻结的 D 解码。T、潜维度、DiT 头数、层数、学习率和各几何权重需要使用作者实际配置，预印本提到 SI B/C 但本地该版本没有完整参数表，应以作者仓库对应版本的配置补足。

生成后由距离重建连接图，并与目标化学图做同构检查；任一键长 <0.8 Å 即剔除。通过此过滤仍不等于构象能量正确。报告全量通过率和通过样本的键长、键角、二面角分布，与 DFT 分布作 KL 比较；最大结构的生成失败率也应保留。[4.5、2.2]

```text
stage1: graph_condition -> VAE encode -> geometry decode -> weighted geometry losses
freeze structural VAE
stage2: encode DFT structure -> draw noise/time -> two-pass DiT -> velocity MSE
inference: graph_condition + noise -> Euler trajectory -> geometry decode
filter: periodic connectivity isomorphism + minimum bond length
evaluate: acceptance / diversity / bond-angle-dihedral distributions
```

## 最容易复现错的地方

周期链跨盒边界的键必须用周期几何；二面角 359° 与 1° 差 2°；loss 的 averaging 需 mask 掉 padding 原子；联合训练必须注明上采样率；flow-matching 不能误写成离散 token 的 DDPM；三维有效率不能引用综述“VAE 通常 >90%”作为本模型成绩。

本包提供方法说明与来源，未运行 DFT、模型训练或构象能量验证。执行前优先核对正式期刊版和对应仓库版本。
