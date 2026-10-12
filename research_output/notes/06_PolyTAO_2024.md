# PolyTAO：描述符条件、半模板与无模板生成

**论文**：Qiu & Sun, *On-demand reverse design of polymers with PolyTAO*, npj Computational Materials 10, 273 (2024)，[DOI](https://doi.org/10.1038/s41524-024-01466-5)。综述 [69]。

[原文与补充信息](https://doi.org/10.1038/s41524-024-01466-5)；[作者示例](https://github.com/hkqiu/PolymerGenerationPretrainedModel)；[公开 checkpoint](https://huggingface.co/hkqiu/PolymerGenerationPretrainedModel)。

![PolyTAO](../figures/PolyTAO.png)

## 数据与标签

PI1M 接近百万个 p-SMILES，无实验性能标签。用 RDKit 2023.3.2 为每个结构计算 15 个描述符，形成“计算描述符 → 同一结构”的监督训练对；80% 训练、20% 测试。测试生成输入为描述符，输出为 SMILES。不能将这百万对当成百万实测 Tg 或带隙数据。[Methods]

SI Table S1 的 15 项：NumHAcceptors、NumHDonors、NHOHCount、NOCount、NumAliphaticCarbocycles、NumAliphaticHeterocycles、NumAliphaticRings、NumAromaticCarbocycles、NumAromaticHeterocycles、NumAromaticRings、RingCount、NumRotatableBonds、NumHeteroatoms、HeavyAtomCount、MolWt。

表称 MolWt 为 exact molecular weight，但 RDKit MolWt 与 ExactMolWt 是不同函数；复现应使用作者 notebook 的具体函数。MolWt 输入被转为整数，使用截断还是四舍五入、是否保留 * 等细节，应直接对照示例，不能自行改变。模型输入是有固定格式的串，描述符顺序同样属于训练配置。[SI、Prompt engineering]

## 架构与损失

基于原 PolyNC 的 encoder–decoder 和其初始化权重，规模超过 22M，encoder 编码性质提示，decoder 通过 cross-attention 与因果自回归生成 p-SMILES。

训练的自然目标为条件序列负对数似然：

$$L_{seq}=-\sum_{t\in nonPAD}\log p_\theta(s_t\mid s_{<t},d_1,\ldots,d_{15}).$$

这是与 encoder–decoder teacher-forcing 一致的实现表达；原正文未完整报告优化器、学习率、batch、epochs 和逐项 loss 设置，需结合 notebook/checkpoint config。**描述符误差和 R² 是生成评估，不能未经代码验证就加入训练 loss。**

重构 x→x 与性质→结构是不同输入关系：PolyTAO 的提示通常不包含完整模板 SMILES，而包含模板的描述符。“半模板”保留这 15 维结构约束，不应误解为复制一个结构片段。

## 三种任务设置

| 阶段 | encoder 输入 | decoder 标签 | 目的 |
|---|---|---|---|
| 基础预训练 | 15 维描述符提示 | SMILES | 学习基础性质到结构 |
| 半模板微调 | 15 描述符 + 1 目标性质 | SMILES | 保留描述符约束同时调目标 |
| 无模板微调 | 1 个目标性质 | SMILES | 只按指定性质生成 |

SI 选择三个 downstream 数据集：atomization energy 390、electron affinity 368、band gap chain 3380。band gap 指链带隙，不是 bulk/crystal 的标签混合。其他 proxy 集合还包括 Tg、折射率等，但不是每个都进行了相同生成验证。[SI Table S2]

评估真实目标依赖另训 GNN proxy。它们每个训练五次；需要固定对应数据划分，避免生成器记忆的结构又出现在预测器测试集。高 proxy R² 不足以保证生成的 OOD 结构可靠。

## 已确认结果的含义

199,159 个 top-1 测试输出中，化学有效率 99.27%；有效唯一生成结构的 15 描述符平均 R² 约 0.96。该 R² 是重新计算描述符对输入描述符的拟合，不是实验性能预测的统一 R²。[Results、Table 1]

top-k 任务还评估 uniqueness、novelty、Tanimoto similarity 与 SA score。原文 novelty 对训练集和测试集都排重；对照其他论文时需统一口径。top-k 是每个提示生成 k 个候选，不能自动等同于 top-k token sampling 参数。

## 实际复现路线

先下载作者 checkpoint 和 tokenizer；用 notebooks/generation-FigureS2.ipynb 验证基础描述符生成；随后按 notebooks/band gap.ipynb、atomization energy.ipynb、electron affinity.ipynb 做半模板；用 template-free band gap.ipynb 做无模板。作者仓库后续加入其他模型，需使用本包保存 commit，避免将新 PolyTAO variants 与这篇 2024 工作混合。

```text
canonical p-SMILES -> compute 15 RDKit descriptors -> fixed prompt serialization
load original tokenizer + original PolyNC/PolyTAO config
teacher forcing fine-tune: descriptor/property prompt -> target SMILES
for each held-out prompt:
    generate k outputs with fixed decoding settings
    parse / check endpoints / canonicalize / deduplicate
    recalculate descriptor values
    independently predict downstream property and record uncertainty/domain
```

不得依据模型名称自行推断它是标准 T5；具体类与 config 来自公开权重。建议先保存 20 个作者示例提示和期望输入格式作为回归检查，再调训练；提示列名、分隔符、数值精度和截断规则都要固定。

## 复现缺口

预训练权重与生成示例公开，完整从头预训练的所有参数及数据清洗过程不是正文/SI 都有。应分别标注“公开权重推断可复现”“方法重训”“原数值全复现”。目前笔记不声称已执行下载权重、微调或实验性能验证。

**证据定位**：Methods 的 Dataset/Prompt engineering/Model settings、正文 Table 1、SI S1/S3、作者 notebooks。
