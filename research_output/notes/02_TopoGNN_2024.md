# TopoGNN：同时学习拓扑重构、尺寸与架构类别

**论文**：Jiang, Dieng & Webb, *Property-guided generation of complex polymer topologies using variational autoencoders*, npj Computational Materials 10, 139 (2024)，[DOI](https://doi.org/10.1038/s41524-024-01328-0)。综述 [63]。

**原文与补充信息**：见 [npj Computational Materials 文章页面](https://doi.org/10.1038/s41524-024-01328-0)；[作者代码](https://github.com/webbtheosim/poly-topoGNN-vae)。

![TopoGNN](../figures/TopoGNN.png)

## 数据是什么

1342 个化学均一的粗粒化聚合物图，每个 90–100 个等价 beads。线形与环形各 11 个；branch、comb、star、dendrimer 各 330 个。训练/验证/测试为 858/215/269，按类别分层。标签是稀溶液模拟的平均平方回转半径 ⟨Rg²⟩ 和六分类拓扑标签。不能把数据理解为 1342 种真实化学重复单元，也不能把 ⟨Rg²⟩ 当成 Rg。[Methods: Description of dataset]

补齐到 100 节点，ghost 节点不连边。邻接矩阵 A 和节点特征 X 均为 100×100；节点特征就是该节点的邻接向量。节点和边化学类型相同，因此不存在原子元素或键型特征。

11 维描述符依次包含节点数、边数、平均度、平均邻居度、密度、直径、半径、代数连通度、度中心性、介数中心性和度同配性。部分量需对全图聚合，精确定义与归一化应按作者 data_utils.py 和 SI。[Methods、SI Discussion 1]

## 模型结构

两层 GIN 图编码器生成 32 维 hg；DNN 将 11 维描述符编码为 32 维 ht；拼接成 64 维 h，分别投影到 μ 与 log σ²，得到 8 维潜空间。CNN 解码邻接矩阵；两个独立网络预测 ⟨Rg²⟩ 和拓扑类别。对照 GNN 只用图分支，Topo 只用描述符分支。[Methods: VAE architecture、图 8]

## 损失函数

$$L=L_{Rec}+L_{KL}+\lambda_{Reg}L_{Reg}+\lambda_{Cls}L_{Cls}.$$

- 重构：A 的逐元素二元交叉熵。
- KL：对角高斯与标准正态的散度。
- 回归：原文定义的尺寸回归误差；实施时按作者 model_utils.py 对平方量和缩放的处理。
- 分类：六分类交叉熵。

邻接矩阵非常稀疏，单纯预测全零会获得高普通准确率。因此模型选择使用 balanced accuracy，而不是仅看 accuracy。BCE 按求和还是平均处理，会改变它与 KL、辅助损失的相对权重，移植到 PyTorch 时必须保持作者的 reduction。[Loss functions、Model training]

## 已报告训练设置

| 设置 | 原文 |
|---|---|
| 框架 | TensorFlow |
| 优化器与轮数 | Adam，1000 epochs |
| batch 候选 | 32、64、128 |
| 学习率候选 | 1e-4、1e-3、1e-2 |
| λReg、λCls 候选 | 各 0.01、0.1、1、10、100 |
| checkpoint 口径 | 总验证损失、ELBO 或重构 BACC |
| 最优模型选择 | BACC、KL、回归 R²、分类 F1 组成的验证距离 |

三个编码器共 2025 组组合；不是一个单一“通用最优配置”。作者 README 环境：Python 3.8、TensorFlow 2.5.0、spektral 1.3.0、networkx 2.8.4、scikit-learn 1.2.2。最佳组合应由 SI/作者搜索结果对应具体架构确定。

## 条件生成其实怎样实现

从原数据找满足目标 ⟨Rg²⟩ ±2 和拓扑类别的父代，在其潜向量附近加均值 0、方差 0.1 的高斯扰动；模型预测新候选的尺寸与类别并预筛；对邻接输出做对称化与图清理；清理后重新核验尺寸；通过图同构排除已有或重复拓扑。[Methods: Generative tasks]

原文训练中没有硬性强制 A 对称；生成时使用 max(Ahat, Ahatᵀ)。不要改成平均再把结果当成原实现。图清理还必须处理孤立 ghost、连通性、节点数与异常连接，详见 graph_utils.py 和 SI Discussion 4。

## 复现操作

```bash
git clone https://github.com/webbtheosim/poly-topoGNN-vae.git
cd poly-topoGNN-vae
# 在作者版本环境下，修改数据目录与输出目录
python run_vae.py
```

run_vae.py 含 HPC 数组任务语境，单机使用时检查任务索引。先跑 result.ipynb/数据构建和划分；无需重扫全部 2025 组也可先验证一种组合，但须标为方法复现。使用 polymer_generation_playground.ipynb 进行目标生成，figure_data.ipynb、figure_result.ipynb 重现图表。[作者 README]

作者仓库 README 的 Download Data 链接仍是占位文本，须进一步从正式论文 Data availability 或作者获取完整原始标签。建议以已知拓扑图的小规模重构检查环境，不能以自己生成的标签代替论文模拟标签。

## 物理验证

论文对目标 7.5±2、30±2、50±2 等拓扑重新执行粗粒化 MD，并在类似尺寸下比较流变性。沿用 SI 的 bead 势、温度、浓度、平衡和采样时间；仅满足图分类不等于尺寸或黏度得到物理验证。本模型不会输出原子 SMILES，也不保证某个拓扑能被精确合成。

**证据定位**：正文图 6/8、Methods 的 Representation、VAE architecture、Loss functions、Generative tasks；SI Discussion 1–4；作者 README/model_utils.py。
