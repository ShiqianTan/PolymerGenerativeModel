# Graph-to-String VAE：生成共聚物单体、比例与链架构

**论文**：Vogel & Weber, *Inverse design of copolymers including stoichiometry and chain architecture*, Chemical Science 16, 1161–1178 (2025，在线发表 2024-12-17)，[DOI](https://doi.org/10.1039/D4SC05900J)。综述 [47]。

[正式版与补充信息](https://doi.org/10.1039/D4SC05900J)；[代码与数据](https://github.com/Intelligent-molecular-systems/Inverse_copolymer_design)。

![Graph-to-String VAE](../figures/Copolymer.png)

## 数据与生成对象

原数据 42,966 个共轭共聚物，有 DFT-derived EA、IP 标签；正文描述 8 种 A 单体和 682 种 B 单体，比例 1:1/1:3/3:1，链架构 alternating/random/block。预印本图中文字存在 A 单体数 9 的不一致，实际复现应按正式版数据和代码统计，不能用示意图乘法推算样本量。

正式版通过 B–B 组合扩充 68,549 个无标签结构，形成 111,515 条部分标注数据；预印本曾写 138,447，属于版本差异。扩增结构的 EA/IP 应为缺失，不能填零标签。[正式版 2.3]

结构串包含单体 SMILES、单体计量和连接概率，例如：

```text
monomer_A.monomer_B|fraction_A|fraction_B|connection_probabilities
```

编码成带概率权重的有向聚合物图；节点/键保持化学信息，虚拟连接边表达单体间连接概率。生成得到的是统计 ensemble 描述，不是一条逐单体列出的确定共聚序列。[2.1]

## 模型

wD-MPNN 图编码器：主块三层，后接 μ、logσ² 两分支；latent 为 32 维。Transformer 解码器四层、每层四注意力头，使用自回归解码；latent 同时参与 cross-attention，并在位置编码后与每个 token embedding 拼接。另有性质 FFNN 预测 EA/IP。[2.2]

tokenization 同时处理 SMILES 和数值。RT-inspired tokenizer 区分环编号的数字与计量/概率的小数，避免同一“1”对应不同语义；训练、生成、数据加载必须采用同一种模式。[Appendix A.1.1]

## 损失

$$L=L_{Rec}+\beta L_{KL}+\alpha L_y.$$

重构用 weighted token CE；KL 对标准正态；EA/IP 标签 mask 后计算 MSE：

$$L_y=\frac1{NM}\sum_{i=1}^N\sum_{j=1}^{M=2}m_{ij}(y_{ij}-\hat y_{ij})^2.$$

原文是按整个 batch N 和性质数 M 平均，而不是按有效标签数平均。两种 reduction 会随标签稀疏度产生不同损失比例。β 使用周期性调度缓解 posterior collapse；α 控制性质组织潜空间的强度。[2.2.2、2.4]

## 划分与训练

80/10/10，**按单体组合分组划分**：同一组合的不同配比与架构不得跨集合。teacher forcing 训练，验证 early stopping；推断 beam size=5。常用环境在作者 README：Python 3.10.11，torch 1.13.1+cu117、PyG 2.3.0、onmt 3.1.3、RDKit 2023.03.1。

作者当前脚本默认值不代表论文最优值：scripts/train.py 包含 latent=32、decoder layers=4、max_beta=0.1、max_alpha=0.1、seed=42、ppguided=0。最后一项默认关闭性质引导；复现半监督模型时必须启用相应开关，并使用 guided 模型类。每个 checkpoint 保存其 model_config，优先从 checkpoint 重建设置。

## 逆向设计目标

论文没有直接学习产氢速率 HER，而是优化与其相关的 EA/IP。目标是尽量负的 EA 和接近 1 eV 的 IP，参照 SHE 能标，不能换成真空能级而沿用同一阈值。正式版式 (4) 结合 EA 与 IP 偏离项；运行作者 optimize_BO.py/optimize_GA_correct.py 时核对其确切目标和无效样本罚项，不只写成“最大化带隙”。

在 32 维 latent 上做 BO 或 GA。解码后重新编码，再预测性质并评估潜空间 cycle consistency，以防虚假 latent 最优点。候选提升是 surrogate 的判断，不能视为实测催化活性提升。[2.5、Appendix A.1.3]

## 可执行路线

```bash
git clone https://github.com/Intelligent-molecular-systems/Inverse_copolymer_design.git
cd Inverse_copolymer_design
conda env create -f environment.yml
conda activate G2S_VAE_env
# 先按 README 运行 data_processing 中 Transform_Batch_Data_g2s.py
python scripts/train.py --beta schedule --ppguided 1 --dec_layers 4
# 训练参数、数据增强和 tokenization 必须在后续脚本中保持一致
python scripts/generate.py
python scripts/optimize_BO.py
python scripts/optimize_GA_correct.py
```

上述训练命令启用方法所需开关，但其默认 β/α 并非承诺最优配置；需要再以论文搜索结果和 checkpoint 参数覆盖。

评估要同时比较化学单体、计量和连接架构；只比两个单体的 canonical SMILES 会高估重构率。原文以 16,000 个先验采样估计生成指标，使用 μ 评估确定性重构。记录无标签比例、mask reduction、划分的单体组合键、BO/GA 预算和无效样本率。

**证据定位**：正式版 2.1–2.5、Appendix A；作者 README、scripts/train.py、model/G2S_clean.py 与 guided 模型实现。数据版本冲突明确保留，不混用。
