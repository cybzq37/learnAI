
GPS轨迹点 → 候选路段筛选 → 构建HMM模型 → Viterbi算法求解 → 路径规划重建 → 匹配轨迹


### 1. 候选路段筛选（`get_candidate_roads`）

- 对每个GPS点，使用**R-tree空间索引**在指定半径内搜索附近路段
- 计算点到路段的**垂直距离**和**投影点**
- 按距离排序，取前N个作为候选路段（`max_candidates`）
- 如果某个GPS点没有候选路段，该点将被忽略

### 2. 构建HMM模型参数

#### 观测序列（`observation_sequence`）
- 有效GPS点的索引序列（过滤掉没有候选路段的点）

#### 状态空间（`state_candidates`）
- 每个观测点对应的候选路段索引列表

#### 发射概率（`calculate_emission_probability`）

- 衡量GPS点与候选路段的匹配程度
- 基于**高斯分布**，考虑两个因素：
  - **空间距离**：GPS点到路段的垂直距离
  - **方向因子**：GPS航向与路段方向的夹角
- 公式：`P = heading_factor × exp(-0.5 × (distance_param × distance / σ)²)`

#### 转移概率（`calculate_transition_probability`）

- 衡量两个连续GPS点之间的候选路段组合是否合理
- 比较**GPS直线距离**与**路网最短路径距离**的差异
- 差异越小，概率越高（**指数分布**）
- 公式：`P = exp(-distance_param × |route_distance - straight_distance| / β)`


### 3. Viterbi算法求解最优路径（`ViterbiSolver`）

#### 前向传播（`solve` → `_calculate_probability_matrix`）
- 从第一个观测点开始，递推计算每个时刻每个状态的最大概率
- 记录**最优前驱状态**（`psi_matrices`）和**累积概率**（`zeta_matrices`）
- 支持**对数概率模式**（避免数值下溢），将乘法转为加法

#### 后向回溯（`_backtrack`）
- 从最后一个时刻的概率最大值开始
- 通过`psi_matrices`向前回溯，得到完整的最优状态序列


GPS 点成为观测值，他的集合称为状态空间

每个GPS点按照距离远近都会有多个候选路段


假设当前GPS点有m个候选路段，下一个GPS点有n个候选路段，那么就构成了 m*n 的矩阵

T个观测点一共纯在T-1次状态转移

发射概率 

计算当前点候选路段 到 下个点所有候选路段 的概率

发射概率 * 转移概率

viterbi算法：
前向计算：计算每个候选路段的最大概率，对于当前每个候选路段，都是有可能从前面的m个转化而来，只计算最大概率，3个候选就有3个最大概率
反向回溯计算：从最后一个候选边的最大概率开始算，往前计算所有候选路段连在一起的最大概率

