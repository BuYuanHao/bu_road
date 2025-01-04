## Cypher查询语句

1. 要仅包含具有属性的节点/关系，请使用 HAS() 函数，然后写出标识符和您希望它具有的属性。

    ```
    WHERE has(n.name)
    ```

2.  如果希望即使没有属性也返回 true，则可以使用问号.

    ```
    WHERE n.name ?= '张三'
    ```

3. 相反的，没有属性则为false，则使用！。

    ```
    WHERE n.name != '张三'
    ```

## 中心性算法

## 度中心性

### 理论基础
度中心性基于一个简单的假设：与更多节点相连的节点更重要。

场景-社交网络分析

    识别社交网络中的意见领袖
    发现活跃用户和潜在网红
    社群营销目标用户筛选

场景-电商平台

    分析商品推荐关系网络
    识别热门商品和关联产品
    优化商品陈列和促销策略
### 数学定义
```
DC(v) = deg(v)
其中：
- DC(v)为节点v的度中心性
- deg(v)为节点v的度
```

### 细分类型
1. 入度中心性（In-Degree）
   - 衡量指向该节点的边数
   - 表示节点的受欢迎程度

2. 出度中心性（Out-Degree）
   - 衡量从该节点出发的边数
   - 表示节点的影响力

### 实现代码
```cypher
// 基础度中心性
CALL gds.degree.stream('myGraph')
YIELD nodeId, score

// 带权重的度中心性
CALL gds.degree.stream('myGraph', {
    relationshipWeightProperty: 'weight'
})
YIELD nodeId, score

// 归一化度中心性
CALL gds.degree.stream('myGraph', {
    normalized: true
})
YIELD nodeId, score
```

### 性能特征
- 时间复杂度：O(V):只需遍历每个节点一次
- 空间复杂度：O(V)
- 并行化：易于并行处理:需要存储每个节点的度数

## 中介中心性(Betweenness Centrality)

### 理论基础
衡量节点作为网络中其他节点对之间最短路径的中转点的频率。


场景-物流网络优化

    识别关键物流节点
    优化仓储位置
    评估路线脆弱性

## 数学定义
```
BC(v) = ∑(s≠v≠t) (σst(v)/σst)
其中：
- σst为s到t的最短路径数
- σst(v)为经过节点v的s到t的最短路径数
```

### 算法步骤
1. 计算所有节点对之间的最短路径
2. 对每个节点，计算通过它的最短路径比例
3. 汇总得到最终分数

### 实现代码
```cypher
// 基础中介中心性
CALL gds.betweenness.stream('myGraph')
YIELD nodeId, score

// 采样中介中心性（提高性能）
CALL gds.betweenness.stream('myGraph', {
    samplingSize: 1000
})
YIELD nodeId, score

// 带权重的中介中心性
CALL gds.betweenness.stream('myGraph', {
    relationshipWeightProperty: 'weight'
})
YIELD nodeId, score
```

### 性能
- 时间复杂度：O(V³):需要计算所有节点对之间的最短路径,计算每个节点在最短路径中出现的次数
- 空间复杂度：O(V + E)

## 接近中心性(Closeness Centrality)

### 理论基础
测量节点到网络中所有其他节点的平均最短路径长度的倒数。

场景-应急响应系统

    选择最佳应急设施位置
    优化救援站点布局
    评估响应时间

场景-城市规划

    公共设施选址
    交通枢纽规划
    社区服务中心布局

### 数学定义
```
CC(v) = (n-1) / ∑(u≠v) d(v,u)
其中：
- n为节点总数
- d(v,u)为节点v到u的最短路径长度
```



### 实现代码
```cypher
// 基础接近中心性
CALL gds.closeness.stream('myGraph')
YIELD nodeId, score

// 带权重的接近中心性
CALL gds.closeness.stream('myGraph', {
    relationshipWeightProperty: 'weight'
})
YIELD nodeId, score
```

### 性能
- 时间复杂度：O(V * (V + E)):需要计算每个节点到其他所有节点的最短路径
- 空间复杂度：O(V + E)
## PageRank中心性

### 理论基础
基于递归定义：节点的重要性取决于指向它的节点的重要性。

场景-搜索引擎优化

    网页重要性排名
    内容推荐系统
    用户行为分析
场景-学术引用网络

    论文影响力评估
    研究主题分析
    学者影响力排名
### 数学定义
```
PR(v) = (1-d) + d * ∑(u→v) (PR(u)/OutDeg(u))
其中：
- d为阻尼因子（通常为0.85）
- OutDeg(u)为节点u的出度
```

### 关键参数
1. 阻尼因子（dampingFactor）
   - 控制随机游走的概率
   - 影响收敛速度

2. 最大迭代次数
   - 控制算法终止条件
   - 平衡精度和性能

### 实现代码
```cypher
// 基础PageRank
CALL gds.pageRank.stream('myGraph')
YIELD nodeId, score

// 自定义参数的PageRank
CALL gds.pageRank.stream('myGraph', {
    maxIterations: 20,
    dampingFactor: 0.85,
    tolerance: 1e-7
})
YIELD nodeId, score
```

## 算法对比

### 性能比较表（v：节点 E：边）

| 指标 | 度中心性 | 中介中心性 | 接近中心性 | PageRank |
|-----|---------|------------|------------|----------|
| 计算复杂度 | O(V) | O(V*V*V) | O(V*(V+E)) | O(k*(V+E)) |
| 内存消耗 | 低 | 高 | 中 | 中 |
| 并行化能力 | 强 | 中 | 中 | 强 |
| 增量更新 | 支持 | 部分支持 | 不支持 | 支持 |
| 适用规模 | 大规模 | 中等规模 | 中等规模 | 大规模 |

### 应用场景对比

| 场景 | 最适合的算法 | 原因 |
|-----|-------------|------|
| 社交网络影响力 | PageRank | 考虑连接质量和递归影响 |
| 网络控制点分析 | 中介中心性 | 识别信息流控制点 |
| 快速重要性评估 | 度中心性 | 计算简单，直观有效 |
| 信息传播效率 | 接近中心性 | 考虑全局传播距离 |

## 实践建议

### 算法选择考虑因素
1. 数据规模
   - 大规模图优先考虑度中心性和PageRank
   - 中等规模可使用全部算法

2. 性能要求
   - 实时计算场景选择度中心性
   - 批处理场景可选择复杂算法

3. 业务场景
   - 社交网络分析优先PageRank
   - 物流网络优先中介中心性
   - 应急响应网络优先接近中心性

### 优化建议
1. 数据预处理
   - 移除无关的节点和边
   - 预计算部分指标

2. 参数调优
   - PageRank的阻尼因子调整
   - 采样率的选择
   - 并行度的设置

3. 结果后处理
   - 归一化处理
   - 结果缓存策略
   - 增量更新机制