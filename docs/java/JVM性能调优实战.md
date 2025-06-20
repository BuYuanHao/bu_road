# JVM性能调优实战指南

## 目录
1. [JVM内存模型概述](#jvm内存模型概述)
2. [性能问题诊断](#性能问题诊断)
3. [JVM参数调优](#jvm参数调优)
4. [垃圾回收器选择与调优](#垃圾回收器选择与调优)
5. [性能监控工具](#性能监控工具)
6. [实战案例分析](#实战案例分析)
7. [调优最佳实践](#调优最佳实践)

## JVM默认配置详解

### 🔍 查看当前JVM默认配置
```bash
# 查看所有JVM默认参数
java -XX:+PrintFlagsFinal -version | grep -E "(UseG1GC|UseParallelGC|NewRatio|SurvivorRatio)"

# 查看具体应用的JVM配置
jinfo -flags <pid>                    # 查看运行时JVM参数
jinfo -flag NewRatio <pid>            # 查看特定参数值

# 查看默认垃圾收集器
java -XX:+PrintCommandLineFlags -version

# 输出示例：
# -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:+PrintCommandLineFlags 
# -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC
```

### ⚙️ JVM关键默认配置表

#### 内存相关默认值
| 参数 | JDK8默认值 | JDK11默认值 | JDK17默认值 | 说明 |
|------|------------|-------------|-------------|------|
| `-Xms` | 物理内存/64 | 物理内存/64 | 物理内存/64 | 初始堆大小 |
| `-Xmx` | 物理内存/4 | 物理内存/4 | 物理内存/4 | 最大堆大小 |
| `-Xmn` | 未设置 | 未设置 | 未设置 | 年轻代大小(由NewRatio决定) |
| `NewRatio` | 2 | 2 | 2 | 老年代:年轻代=2:1 |
| `SurvivorRatio` | 8 | 8 | 8 | Eden:Survivor=8:1 |
| `MetaspaceSize` | ~21MB | ~21MB | ~21MB | 元空间初始大小 |
| `MaxMetaspaceSize` | 无限制 | 无限制 | 无限制 | 元空间最大大小 |

#### 垃圾收集器默认选择
```bash
# JDK8默认选择逻辑
if (物理内存 >= 2GB && CPU核心数 >= 2) {
    垃圾收集器 = "Parallel GC";        # -XX:+UseParallelGC
} else {
    垃圾收集器 = "Serial GC";          # -XX:+UseSerialGC
}

# JDK9+默认选择逻辑  
if (物理内存 >= 2GB && CPU核心数 >= 2) {
    垃圾收集器 = "G1 GC";              # -XX:+UseG1GC (JDK9+)
} else {
    垃圾收集器 = "Serial GC";          # -XX:+UseSerialGC
}

# 查看当前默认垃圾收集器
java -XX:+PrintCommandLineFlags -version 2>&1 | grep -o "Use.*GC"
```

#### GC相关默认参数
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `MaxGCPauseMillis` | 200ms (G1GC) | G1收集器的暂停时间目标 |
| `GCTimeRatio` | 99 | 期望GC时间占比1% |
| `ParallelGCThreads` | CPU核心数 | 并行GC线程数 |
| `ConcGCThreads` | ParallelGCThreads/4 | 并发GC线程数 |
| `G1HeapRegionSize` | 堆大小/2048 | G1区域大小(1MB-32MB) |

### 📊 不同内存规模的默认行为

#### 小内存环境 (< 2GB)
```bash
# 典型配置：1GB物理内存的服务器
默认垃圾收集器: Serial GC
默认堆大小: -Xms16m -Xmx256m
年轻代:老年代: 1:2
Eden:Survivor: 8:1:1

# 实际内存分配 (256MB堆)
老年代: 170MB (256 * 2/3)
年轻代: 86MB  (256 * 1/3)
Eden区: 68MB   (86 * 8/10)
S0区:   9MB    (86 * 1/10)
S1区:   9MB    (86 * 1/10)
```

#### 中等内存环境 (2-8GB)
```bash
# 典型配置：8GB物理内存的服务器
默认垃圾收集器: Parallel GC (JDK8) / G1GC (JDK9+)
默认堆大小: -Xms128m -Xmx2g
年轻代:老年代: 1:2
并行GC线程数: 等于CPU核心数

# 实际内存分配 (2GB堆)
老年代: 1365MB (2048 * 2/3)
年轻代: 683MB  (2048 * 1/3)
Eden区: 546MB  (683 * 8/10)
S0区:   68MB   (683 * 1/10)
S1区:   68MB   (683 * 1/10)
```

#### 大内存环境 (> 8GB)
```bash
# 典型配置：32GB物理内存的服务器
默认垃圾收集器: G1GC (JDK9+)
默认堆大小: -Xms512m -Xmx8g
G1区域大小: 16MB (8GB/512)
最大暂停时间: 200ms

# G1GC默认配置分析
G1NewSizePercent: 5%     # 年轻代最小占比
G1MaxNewSizePercent: 60% # 年轻代最大占比
G1HeapRegionSize: 16MB   # 区域大小
```

### 🚀 默认配置性能特点

#### JDK8 Parallel GC默认行为
```bash
# 优点
+ 高吞吐量，适合批处理任务
+ 多线程并行回收，充分利用CPU
+ 成熟稳定，广泛使用

# 缺点  
- GC停顿时间较长(几百毫秒)
- 不适合延迟敏感的应用
- Full GC时会停止所有应用线程

# 适用场景
✓ 批处理作业
✓ 数据分析任务  
✓ 对吞吐量要求高的后台服务
✗ 在线交易系统
✗ 实时响应系统
```

#### JDK9+ G1GC默认行为
```bash
# 优点
+ 低延迟，暂停时间可预测
+ 适合大内存应用
+ 并发回收，减少停顿

# 缺点
- 吞吐量相比Parallel GC略低
- 内存开销相对较大
- 需要更多CPU资源

# 适用场景  
✓ 在线Web应用
✓ 微服务架构
✓ 大内存缓存服务
✓ 延迟敏感的系统
```

### 🔧 默认配置的常见问题

#### 1. 堆内存默认值过小
```bash
# 问题现象
- 频繁触发GC
- 应用启动后堆内存快速增长
- OutOfMemoryError

# 解决方案
-Xms4g -Xmx4g                # 根据应用需求设置合适的堆大小
-XX:+AlwaysPreTouch          # 启动时就分配所有内存，避免运行时分配延迟
```

#### 2. 元空间默认值不足
```bash
# 问题现象  
- java.lang.OutOfMemoryError: Metaspace
- 大量类加载的应用启动失败

# 解决方案
-XX:MetaspaceSize=256m       # 设置合适的元空间初始大小
-XX:MaxMetaspaceSize=512m    # 设置元空间上限，避免无限制增长
```

#### 3. 默认GC不适合业务场景
```bash
# 在线服务使用JDK8默认Parallel GC的问题
- API响应时间不稳定
- 用户体验差，偶尔出现明显卡顿

# 解决方案
-XX:+UseG1GC                # 切换到G1收集器
-XX:MaxGCPauseMillis=100    # 设置暂停时间目标
```

### 📋 默认配置检查脚本
```bash
#!/bin/bash
echo "=== JVM默认配置检查 ==="
echo "JDK版本:"
java -version

echo -e "\n当前JVM默认参数:"
java -XX:+PrintFlagsFinal -version 2>/dev/null | grep -E "(UseG1GC|UseParallelGC|NewRatio|SurvivorRatio|MetaspaceSize)" | grep -v "product"

echo -e "\n推荐的堆内存配置 (基于当前系统):"
TOTAL_MEM=$(free -m | awk '/^Mem:/{print $2}')
RECOMMENDED_HEAP=$((TOTAL_MEM * 75 / 100))
echo "物理内存: ${TOTAL_MEM}MB"
echo "推荐堆大小: ${RECOMMENDED_HEAP}MB"
echo "推荐配置: -Xms${RECOMMENDED_HEAP}m -Xmx${RECOMMENDED_HEAP}m"

echo -e "\n系统资源信息:"
echo "CPU核心数: $(nproc)"
echo "推荐GC线程数: $(nproc)"

if [ $TOTAL_MEM -gt 4096 ]; then
    echo -e "\n推荐垃圾收集器: G1GC"
    echo "推荐配置: -XX:+UseG1GC -XX:MaxGCPauseMillis=100"
else
    echo -e "\n推荐垃圾收集器: Parallel GC"
    echo "推荐配置: -XX:+UseParallelGC"
fi
```

### 🔄 默认配置 vs 生产配置对比

#### 典型Web应用配置对比 (8GB内存服务器)
```bash
# ===== 默认配置 =====
# JDK8默认
-Xms128m                              # 初始堆内存过小
-Xmx2g                                # 最大堆内存偏小
-XX:+UseParallelGC                    # 使用Parallel GC
-XX:NewRatio=2                        # 年轻代占1/3
-XX:SurvivorRatio=8                   # Eden:Survivor = 8:1
-XX:ParallelGCThreads=8               # 8个并行GC线程

# 存在的问题：
❌ 堆内存设置过小，容易发生GC
❌ Parallel GC暂停时间长，影响响应
❌ 没有GC日志，难以监控和调优

# ===== 生产配置 =====
# 优化后的配置
-Xms4g -Xmx4g                         # 固定堆内存4GB
-XX:+UseG1GC                          # 使用G1收集器
-XX:MaxGCPauseMillis=100              # 限制GC暂停时间
-XX:G1HeapRegionSize=16m              # 设置合适的区域大小
-XX:+G1UseAdaptiveIHOP                # 自适应优化
-XX:MetaspaceSize=256m                # 设置元空间大小
-XX:MaxMetaspaceSize=512m             # 限制元空间上限

# 监控和调试参数
-XX:+PrintGC                          # 打印GC日志
-XX:+PrintGCDetails                   # 打印详细GC信息
-XX:+PrintGCTimeStamps                # 打印时间戳
-XX:+PrintGCApplicationStoppedTime    # 打印应用停顿时间
-Xloggc:/var/log/gc.log               # GC日志文件
-XX:+UseGCLogFileRotation             # 日志文件轮转
-XX:NumberOfGCLogFiles=5              # 保留5个日志文件
-XX:GCLogFileSize=10M                 # 每个日志文件10MB

# 故障排查参数
-XX:+HeapDumpOnOutOfMemoryError       # OOM时生成堆转储
-XX:HeapDumpPath=/var/log/heapdump/   # 堆转储文件路径
-XX:+PrintFlagsFinal                  # 打印所有JVM参数

# 效果对比：
✅ 堆内存充足，GC频率大幅降低
✅ GC暂停时间从200ms+降低到50ms内
✅ 完整的监控和故障排查能力
✅ 系统稳定性和性能显著提升
```

#### 高并发应用配置示例 (16GB内存服务器)
```bash
# ===== 默认配置性能测试结果 =====
吞吐量: 1000 QPS
平均响应时间: 200ms
P99响应时间: 2000ms
GC暂停时间: 300ms+
Full GC频率: 每小时2-3次

# ===== 生产配置 =====
-Xms8g -Xmx8g                         # 8GB固定堆内存
-XX:+UseG1GC                          # G1收集器
-XX:MaxGCPauseMillis=50               # 严格的暂停时间要求
-XX:G1HeapRegionSize=16m              # 16MB区域大小
-XX:G1NewSizePercent=40               # 年轻代最小40%
-XX:G1MaxNewSizePercent=60            # 年轻代最大60%
-XX:ParallelGCThreads=16              # 16个并行线程
-XX:ConcGCThreads=4                   # 4个并发线程
-XX:+G1UseAdaptiveIHOP                # 自适应优化
-XX:G1MixedGCCountTarget=8            # 混合GC目标8次

# 性能提升结果：
✅ 吞吐量: 2500 QPS (提升150%)
✅ 平均响应时间: 80ms (提升60%)
✅ P99响应时间: 300ms (提升85%)
✅ GC暂停时间: 30ms内 (提升90%)
✅ Full GC频率: 几乎没有
```

### 📈 从默认配置到生产配置的升级路径

#### 第一步：基础优化
```bash
# 当前：使用默认配置
java -jar myapp.jar

# 优化1：设置合适的堆内存
java -Xms4g -Xmx4g -jar myapp.jar

# 预期效果：减少内存动态分配开销，降低GC频率
```

#### 第二步：收集器优化
```bash
# 优化2：切换到G1收集器
java -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -jar myapp.jar

# 预期效果：GC暂停时间大幅减少，响应时间更稳定
```

#### 第三步：监控优化
```bash
# 优化3：添加监控和日志
java -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=100 \
     -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps \
     -Xloggc:gc.log -jar myapp.jar

# 预期效果：可以监控GC表现，为进一步优化提供数据支持
```

#### 第四步：精细调优
```bash
# 优化4：根据GC日志分析结果进行精细调优
java -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=50 \
     -XX:G1HeapRegionSize=16m -XX:G1NewSizePercent=40 \
     -XX:+G1UseAdaptiveIHOP -XX:MetaspaceSize=256m \
     -XX:+PrintGCDetails -Xloggc:gc.log -jar myapp.jar

# 预期效果：达到最佳的性能和稳定性平衡
```

### 🛠️ 不同场景的默认配置问题及解决方案

#### 1. 微服务架构
```bash
# 常见问题：
- 默认堆内存对容器化应用来说过大
- 启动时间长，影响扩缩容
- 内存利用率低

# 解决方案：
-Xms512m -Xmx512m                     # 小堆内存配置
-XX:+UseG1GC                          # 低延迟收集器
-XX:MaxGCPauseMillis=100              # 合理的暂停时间
-XX:+AlwaysPreTouch                   # 预先分配内存，加快启动
-XX:+UseTLAB                          # 线程本地分配缓冲区
```

#### 2. 大数据处理
```bash
# 常见问题：
- 默认堆内存太小，频繁GC
- 处理大对象时内存不足
- 批处理任务对延迟不敏感但需要高吞吐量

# 解决方案：
-Xms16g -Xmx16g                       # 大堆内存
-XX:+UseParallelGC                    # 高吞吐量收集器
-XX:ParallelGCThreads=16              # 充分利用CPU
-XX:+UseParallelOldGC                 # 老年代也使用并行收集
-XX:+UseLargePages                    # 使用大页内存
```

#### 3. 实时系统
```bash
# 常见问题：
- 默认GC暂停时间不可预测
- 响应时间抖动严重
- 实时性要求极高

# 解决方案：
-Xms4g -Xmx4g                         # 固定堆大小
-XX:+UseZGC                           # 超低延迟收集器(JDK11+)
-XX:+UnlockExperimentalVMOptions      # 解锁实验性功能
-XX:+UseLargePages                    # 大页内存
-XX:+AlwaysPreTouch                   # 预分配内存
```

## JVM内存模型概述

### 堆内存结构
```
堆内存 (Heap) - 所有Java对象存储的主要区域
├── 年轻代 (Young Generation) - 新创建对象的存放区域，GC频率高但速度快
│   ├── Eden区 - 对象首次分配的区域，占年轻代约80%
│   ├── Survivor0区 (S0) - 第一次GC后存活对象的存放区域，占年轻代约10%
│   └── Survivor1区 (S1) - 与S0轮换使用，占年轻代约10%
└── 老年代 (Old Generation) - 长期存活对象的存放区域，GC频率低但耗时长
```

**内存分配流程说明：**
1. 新对象首先在Eden区分配
2. Eden区满时触发Minor GC，存活对象移至Survivor区
3. 对象在Survivor区经历多次GC后，晋升至老年代
4. 老年代满时触发Major GC或Full GC

### 非堆内存区域
- **方法区 (Method Area)**: 存储类信息、常量、静态变量
- **程序计数器 (PC Register)**: 当前线程执行字节码的行号
- **本地方法栈 (Native Method Stack)**: 本地方法调用栈
- **直接内存 (Direct Memory)**: NIO使用的堆外内存

## 性能问题诊断

### 常见性能问题症状

#### 1. 内存泄漏
```bash
# 症状特征
- 应用运行时间越长，内存占用越高          # 内存持续增长不释放
- 频繁触发Full GC                       # 老年代回收频繁但效果差
- OutOfMemoryError异常                   # 最终导致内存溢出

# 诊断命令
jmap -histo <pid>                        # 查看对象统计信息，找出占用内存最多的对象类型
jmap -dump:format=b,file=heap.hprof <pid>  # 生成堆转储文件，用于详细分析

# 示例：分析对象统计结果
# num     #instances    #bytes  class name
# 1:      12345678    987654321  [C                    # char数组过多，可能字符串泄漏
# 2:      5432109     543210987  java.lang.String      # 字符串对象过多
# 3:      1234567     123456789  com.example.User      # 自定义对象过多，检查缓存
```

#### 2. CPU使用率过高
```bash
# 诊断步骤
top -H -p <pid>                # 查看进程下各线程的CPU使用情况
                               # -H: 显示线程级别信息
                               # -p: 指定进程ID
                               
jstack <pid>                   # 生成线程栈快照，查看线程在做什么
                               # 重点关注RUNNABLE状态的线程
                               
printf "%x\n" <thread_id>      # 将top命令中的线程ID转换为16进制
                               # 用于在jstack输出中查找对应线程

# 实际操作示例：
# 1. top -H -p 1234 发现线程5678占用CPU 90%
# 2. printf "%x\n" 5678 得到 162e
# 3. 在jstack输出中搜索 nid=0x162e 找到对应线程栈
# 4. 分析栈信息定位热点代码
```

#### 3. GC频繁
```bash
# GC日志分析
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintGCApplicationStoppedTime
```

## JVM参数调优

### 内存参数设置

#### 堆内存参数
```bash
# 基础堆内存设置
-Xms2g                    # 初始堆大小，建议与-Xmx相同，避免动态扩容开销
-Xmx4g                    # 最大堆大小，不超过物理内存的75%，留给操作系统和其他进程
-Xmn1g                    # 年轻代大小，一般设置为堆大小的1/4到1/3
-XX:SurvivorRatio=8       # Eden:Survivor = 8:1，即Eden占年轻代80%，每个Survivor占10%

# 内存比例计算示例：
# 堆内存4GB，年轻代1GB，老年代3GB
# Eden区: 1GB * 8/10 = 800MB
# S0区:   1GB * 1/10 = 100MB  
# S1区:   1GB * 1/10 = 100MB

# 完整示例配置
-Xms4g -Xmx4g -Xmn1g -XX:SurvivorRatio=8

# 注意事项：
# 1. 年轻代过大：老年代相对变小，可能频繁Full GC
# 2. 年轻代过小：对象过早晋升老年代，降低GC效率
# 3. 生产环境建议通过GC日志分析后调整
```

#### 非堆内存参数
```bash
# 方法区/元空间
-XX:MetaspaceSize=256m         # 初始元空间大小
-XX:MaxMetaspaceSize=512m      # 最大元空间大小

# 直接内存
-XX:MaxDirectMemorySize=1g     # 最大直接内存
```

### GC相关参数

#### G1GC参数调优
```bash
# G1GC基础配置
-XX:+UseG1GC                           # 启用G1收集器，适合大内存应用(>4GB)
-XX:MaxGCPauseMillis=100               # 最大GC暂停时间目标，G1会尽量达成此目标
                                       # 在线服务建议100ms，延迟敏感应用可设置50ms
-XX:G1HeapRegionSize=16m               # G1区域大小，堆内存越大区域应该越大
                                       # 计算公式：堆大小/2048，范围1MB-32MB
-XX:G1NewSizePercent=30                # 年轻代最小占比，默认5%，建议20-40%
-XX:G1MaxNewSizePercent=60             # 年轻代最大占比，默认60%，给老年代留足空间
-XX:ParallelGCThreads=8                # 并行GC线程数，一般设置为CPU核心数

# 高级G1配置参数
-XX:+G1UseAdaptiveIHOP                 # 自适应调整IHOP阈值，让G1自动优化
-XX:G1MixedGCCountTarget=16            # 混合GC目标次数，用于分摊老年代回收成本
-XX:G1MixedGCLiveThresholdPercent=85   # 混合GC存活对象阈值，超过此比例的区域不回收
-XX:G1HeapWastePercent=5               # 允许的堆内存浪费百分比

# 完整G1生产环境配置示例 (8GB堆内存)
-XX:+UseG1GC 
-XX:MaxGCPauseMillis=100               # 暂停时间目标100ms
-XX:G1HeapRegionSize=16m               # 8GB/512=16MB区域大小  
-XX:G1NewSizePercent=30                # 年轻代最小30%
-XX:G1MaxNewSizePercent=60             # 年轻代最大60%
-XX:+G1UseAdaptiveIHOP                 # 启用自适应优化
-XX:G1MixedGCCountTarget=16            # 混合GC目标16次
-XX:ParallelGCThreads=8                # 8个并行线程

# 不同内存规模的区域大小建议：
# 2-4GB:  8MB  (-XX:G1HeapRegionSize=8m)
# 4-8GB:  16MB (-XX:G1HeapRegionSize=16m)  
# 8-16GB: 32MB (-XX:G1HeapRegionSize=32m)
# >16GB:  32MB (最大值)
```

#### CMS参数调优
```bash
# CMS收集器配置
-XX:+UseConcMarkSweepGC                # 启用CMS
-XX:+UseParNewGC                       # 年轻代使用ParNew
-XX:CMSInitiatingOccupancyFraction=70  # CMS触发比例
-XX:+UseCMSInitiatingOccupancyOnly     # 只使用设定的触发比例
-XX:+CMSParallelRemarkEnabled          # 并行重标记
-XX:+CMSScavengeBeforeRemark           # 重标记前清理年轻代
```

## 垃圾回收器选择与调优

### 垃圾回收器对比

| 收集器 | 适用场景 | 优点 | 缺点 |
|--------|----------|------|------|
| G1GC | 大内存应用(>4GB) | 低延迟、可预测暂停时间 | 吞吐量相对较低 |
| CMS | 对延迟敏感的应用 | 并发收集、低延迟 | 内存碎片、CPU敏感 |
| Parallel | 吞吐量优先的应用 | 高吞吐量 | STW时间较长 |
| ZGC | 超大内存应用(>32GB) | 超低延迟(<10ms) | JDK11+、内存开销大 |

### G1GC调优实践

#### 1. 确定合适的暂停时间目标
```bash
# 根据业务需求设置暂停时间
-XX:MaxGCPauseMillis=100    # 100ms适合大多数在线应用
-XX:MaxGCPauseMillis=50     # 50ms适合延迟敏感应用
```

#### 2. 调整区域大小
```bash
# 区域大小计算：堆大小 / 2048
# 4GB堆 -> 16MB区域大小
-XX:G1HeapRegionSize=16m

# 8GB堆 -> 32MB区域大小
-XX:G1HeapRegionSize=32m
```

#### 3. 优化Mixed GC
```bash
# Mixed GC调优参数
-XX:G1MixedGCLiveThresholdPercent=85   # 混合GC存活对象阈值
-XX:G1MixedGCCountTarget=16            # 混合GC目标次数
-XX:G1OldCSetRegionThreshold=10        # 老年代回收区域数量
```

## 性能监控工具

### 1. 命令行工具

#### jstat - JVM统计信息
```bash
# GC统计
jstat -gc <pid> 5s            # 每5秒输出GC信息，持续监控内存使用情况
jstat -gcutil <pid> 5s        # GC利用率统计，以百分比显示内存使用率
jstat -gccapacity <pid>       # 查看各内存区域的容量信息
jstat -gcnew <pid>            # 专门查看年轻代GC信息
jstat -gcold <pid>            # 专门查看老年代GC信息

# 示例输出解读 (jstat -gcutil)
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 12.44   0.00  27.20   9.49  96.70  93.76    166    0.074     0    0.000    0.074

# 字段说明：
# S0: Survivor0区使用率 (12.44%)
# S1: Survivor1区使用率 (0.00%)     - 当前S0和S1中只有一个在使用
# E:  Eden区使用率 (27.20%)         - Eden区使用了27.2%
# O:  老年代使用率 (9.49%)          - 老年代使用率较低，说明内存充足
# M:  元空间使用率 (96.70%)         - 元空间使用率过高，可能需要调整
# CCS: 压缩类空间使用率 (93.76%)
# YGC: 年轻代GC次数 (166次)
# YGCT: 年轻代GC总耗时 (0.074秒)
# FGC: Full GC次数 (0次)            - 没有Full GC是好现象
# FGCT: Full GC总耗时 (0.000秒)
# GCT: 总GC耗时 (0.074秒)

# 关键指标分析：
# 1. 老年代使用率超过70%需要关注
# 2. 元空间使用率超过90%可能导致Full GC
# 3. Young GC平均耗时 = YGCT/YGC
# 4. Full GC频率过高(>1次/小时)需要优化
```

#### jmap - 内存映像工具
```bash
# 查看内存使用
jmap -heap <pid>              # 堆内存详情
jmap -histo <pid> | head -20  # 对象统计Top20

# 生成堆转储
jmap -dump:live,format=b,file=heap.hprof <pid>
```

#### jstack - 线程快照工具
```bash
# 生成线程转储
jstack <pid> > thread.dump

# 分析死锁
jstack <pid> | grep -A 10 "Found Java-level deadlock"
```

### 2. 图形化工具

#### JVisualVM配置
```bash
# 启动参数（支持远程监控）
-Dcom.sun.management.jmxremote=true
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

#### MAT内存分析
```bash
# MAT分析要点
1. Leak Suspects Report - 内存泄漏嫌疑
2. Dominator Tree - 支配树分析
3. Histogram - 对象统计
4. Top Components - 大对象分析
```

### 3. 生产环境监控

#### Prometheus + Grafana监控
```yaml
# JMX配置
-javaagent:jmx_prometheus_javaagent.jar=8080:config.yaml

# config.yaml示例
rules:
  - pattern: java.lang<type=GarbageCollector, name=(.*)><>CollectionCount
    name: jvm_gc_collection_count
    labels:
      gc: "$1"
```

## 实战案例分析

### 案例1：电商系统内存泄漏

#### 问题现象
```
- 应用启动正常，运行24小时后开始频繁Full GC
- 堆内存使用率持续增长，从20%增长到95%
- 响应时间从100ms增长到5000ms
```

#### 诊断过程
```bash
# 1. 生成堆转储
jmap -dump:live,format=b,file=leak.hprof <pid>

# 2. 使用MAT分析
# 发现：大量User对象未被释放
# 原因：静态Map缓存用户信息，没有清理机制
```

#### 解决方案
```java
// 原问题代码 - 存在内存泄漏风险
private static Map<String, User> userCache = new HashMap<>();

// 问题分析：
// 1. 使用静态Map作为缓存，对象永远不会被回收
// 2. 没有大小限制，内存会无限增长
// 3. HashMap在并发环境下不安全

// 优化后代码 - 使用安全的缓存实现
private static Map<String, User> userCache = new ConcurrentHashMap<>();
private static final int MAX_CACHE_SIZE = 10000;          // 设置缓存上限
private static final long EXPIRE_TIME = 30 * 60 * 1000;   // 30分钟过期时间

// 方案1：添加LRU清理机制
if (userCache.size() > MAX_CACHE_SIZE) {
    // 清理最久未使用的条目，保持缓存大小在合理范围
    cleanupOldEntries();
}

// 方案2：使用Guava Cache (推荐)
private static final Cache<String, User> userCache = CacheBuilder.newBuilder()
    .maximumSize(10000)                    // 限制缓存大小
    .expireAfterWrite(30, TimeUnit.MINUTES) // 写入后30分钟过期
    .expireAfterAccess(10, TimeUnit.MINUTES) // 访问后10分钟过期
    .removalListener(notification -> {      // 监听缓存移除事件
        System.out.println("Cache removed: " + notification.getKey());
    })
    .build();

// 方案3：定期清理任务
@Scheduled(fixedRate = 300000) // 每5分钟执行一次
public void cleanExpiredCache() {
    long currentTime = System.currentTimeMillis();
    userCache.entrySet().removeIf(entry -> {
        UserWrapper wrapper = (UserWrapper) entry.getValue();
        return currentTime - wrapper.getTimestamp() > EXPIRE_TIME;
    });
}

// 包装类，添加时间戳
class UserWrapper {
    private User user;
    private long timestamp;
    
    public UserWrapper(User user) {
        this.user = user;
        this.timestamp = System.currentTimeMillis();
    }
    
    // getter和setter方法...
}
```

#### 调优参数
```bash
# 调优前
-Xms2g -Xmx4g -XX:+UseParallelGC

# 调优后
-Xms4g -Xmx4g -XX:+UseG1GC 
-XX:MaxGCPauseMillis=100 
-XX:G1HeapRegionSize=16m
```

### 案例2：高并发API响应慢

#### 问题现象
```
- 高峰期API响应时间超过3s
- CPU使用率达到90%
- Young GC频繁，每次停顿50ms+
```

#### 诊断分析
```bash
# 1. GC日志分析
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps

# 发现问题：
# - Young GC每10秒触发一次
# - 每次GC停顿时间50-80ms
# - 年轻代设置过小，导致频繁GC
```

#### 优化方案
```bash
# 调优前参数
-Xms4g -Xmx4g -Xmn512m -XX:+UseParallelGC

# 调优后参数
-Xms6g -Xmx6g -Xmn2g 
-XX:+UseG1GC 
-XX:MaxGCPauseMillis=50
-XX:ParallelGCThreads=8
```

#### 效果对比
```
调优前：
- API平均响应时间：2.5s
- GC停顿时间：50-80ms
- GC频率：每10秒一次

调优后：  
- API平均响应时间：200ms
- GC停顿时间：10-20ms
- GC频率：每30秒一次
```

## 调优最佳实践

### 1. 调优流程

#### 性能基线建立
```bash
# 1. 收集基础性能数据
ab -n 10000 -c 100 http://localhost:8080/api/test

# 2. 监控关键指标
- 吞吐量 (TPS/QPS)
- 响应时间 (RT)
- GC停顿时间
- 内存使用率
```

#### 渐进式调优
```bash
# 调优步骤
1. 分析当前性能瓶颈
2. 设定调优目标
3. 单一变量调整
4. 压测验证效果
5. 记录调优结果
6. 重复上述过程
```

### 2. 参数模板

#### 高吞吐量应用模板
```bash
# 适用场景：批处理、数据处理、科学计算等对吞吐量要求高但对延迟不敏感的应用
-Xms8g -Xmx8g                          # 固定堆大小8GB，避免动态扩容开销
-XX:+UseParallelGC                     # 使用Parallel收集器，专门为吞吐量优化
-XX:ParallelGCThreads=8                # 并行GC线程数，等于CPU核心数
-XX:+UseParallelOldGC                  # 老年代使用Parallel收集器
-XX:+UseAdaptiveSizePolicy             # 启用自适应大小策略，JVM自动调优内存比例

# 可选的吞吐量优化参数
-XX:GCTimeRatio=99                     # 设置GC时间占比目标为1%，最大化应用运行时间
-XX:MaxGCPauseMillis=200               # 虽然是吞吐量优先，但也可设置最大暂停时间避免过长停顿
-XX:+UseLargePages                     # 使用大页内存，减少TLB miss，提升性能
-XX:NewRatio=2                         # 老年代:年轻代=2:1，适合长生命周期对象较多的场景

# 完整配置示例 (16核CPU，32GB内存的批处理服务器)
-Xms16g -Xmx16g 
-XX:+UseParallelGC 
-XX:ParallelGCThreads=16 
-XX:+UseParallelOldGC 
-XX:+UseAdaptiveSizePolicy 
-XX:GCTimeRatio=99 
-XX:+UseLargePages

# 效果预期：
# - 应用吞吐量最大化，GC停顿时间相对较长但频率低
# - 适合离线任务、数据导入导出、大数据计算等场景
# - 不适合在线服务，因为GC停顿时间可能达到几百毫秒
```

#### 低延迟应用模板
```bash
# 适用场景：在线交易系统、实时计算、高频交易等对响应时间极其敏感的应用
-Xms4g -Xmx4g                          # 固定堆大小，避免内存分配延迟
-XX:+UseG1GC                           # 使用G1收集器，专为低延迟设计
-XX:MaxGCPauseMillis=50                # 设置GC暂停时间目标为50ms，适合延迟敏感应用
-XX:G1HeapRegionSize=16m               # 区域大小16MB，平衡回收效率和延迟
-XX:+G1UseAdaptiveIHOP                 # 自适应调整初始堆占用百分比阈值

# 进一步的低延迟优化参数
-XX:G1NewSizePercent=40                # 年轻代最小占比40%，减少对象晋升
-XX:G1MaxNewSizePercent=70             # 年轻代最大占比70%，优先在年轻代回收
-XX:G1ReservePercent=15                # 保留15%堆空间，避免to-space overflow
-XX:G1MixedGCCountTarget=8             # 减少混合GC次数，降低单次回收成本
-XX:+G1UseStringDeduplication          # 启用字符串去重，减少内存占用

# 完整低延迟配置示例 (交易系统)
-Xms8g -Xmx8g                          # 8GB固定堆内存
-XX:+UseG1GC 
-XX:MaxGCPauseMillis=30                # 更严格的暂停时间目标30ms
-XX:G1HeapRegionSize=16m 
-XX:+G1UseAdaptiveIHOP 
-XX:G1NewSizePercent=40 
-XX:G1MaxNewSizePercent=70 
-XX:G1ReservePercent=15 
-XX:G1MixedGCCountTarget=8 
-XX:ParallelGCThreads=8                # 并发GC线程数
-XX:ConcGCThreads=2                    # 并发标记线程数，一般为ParallelGCThreads/4

# 极致低延迟优化（实验性，需要大量测试）
-XX:+UnlockExperimentalVMOptions       # 解锁实验性选项
-XX:+UseZGC                            # 使用ZGC收集器(JDK11+)，暂停时间<10ms
-XX:+UseShenandoahGC                   # 或使用Shenandoah GC，也是低延迟收集器

# 效果预期：
# - GC暂停时间控制在50ms以内，多数情况下10-30ms
# - 适合在线服务、API接口、实时系统
# - 吞吐量相比Parallel GC会有所下降(5-15%)
# - 需要充足的CPU资源支持并发回收
```

#### 大内存应用模板
```bash
# 适用于缓存服务、大数据处理等场景
-Xms16g -Xmx16g 
-XX:+UseG1GC 
-XX:MaxGCPauseMillis=100 
-XX:G1HeapRegionSize=32m 
-XX:ParallelGCThreads=16
```

### 3. 监控告警设置

#### 关键指标阈值
```bash
# GC相关告警阈值设置
- Full GC频率 > 1次/小时              # 频繁Full GC说明内存配置不当或存在内存泄漏
- GC停顿时间 > 100ms                 # 停顿时间过长影响用户体验，需要调优
- Young GC频率 > 1次/分钟            # 年轻代GC过于频繁，可能年轻代过小
- 堆内存使用率 > 85%                 # 内存使用率过高，接近OOM风险
- 元空间使用率 > 90%                 # 元空间不足可能导致类加载异常
- GC时间占比 > 5%                    # GC开销过大，影响应用性能

# 应用性能告警阈值
- API平均响应时间 > 1s               # 接口响应过慢，影响用户体验
- API P99响应时间 > 3s               # 长尾延迟过高，需要优化
- 错误率 > 1%                        # 错误率过高，可能系统不稳定
- CPU使用率 > 80%                    # CPU负载过高，可能影响性能
- 内存使用率 > 85%                   # 内存紧张，可能导致频繁GC

# 详细的GC监控指标
- 年轻代GC平均耗时 < 50ms            # Young GC应该很快完成
- 老年代GC平均耗时 < 200ms           # Old GC耗时相对可以接受
- 每小时GC总暂停时间 < 30s           # 控制总的GC开销
- 内存分配速率监控                    # MB/s，了解对象创建频率
- 对象晋升速率监控                    # MB/s，了解对象进入老年代的速度

# 告警配置示例 (Prometheus AlertManager规则)
# 
# - alert: HighFullGCRate
#   expr: increase(jvm_gc_collections_total{gc="PS MarkSweep"}[1h]) > 1
#   labels: { severity: warning }
#   annotations: { summary: "Full GC频率过高" }
#
# - alert: HighGCPauseTime  
#   expr: rate(jvm_gc_collection_seconds_sum[5m]) > 0.05
#   labels: { severity: critical }
#   annotations: { summary: "GC暂停时间超过5%" }
```

### 4. 常用JVM参数速查表

#### 内存相关
```bash
-Xms<size>              # 初始堆大小
-Xmx<size>              # 最大堆大小  
-Xmn<size>              # 年轻代大小
-XX:NewRatio=n          # 老年代/年轻代比值
-XX:SurvivorRatio=n     # Eden/Survivor比值
-XX:MetaspaceSize=<size> # 元空间初始大小
```

#### GC相关
```bash
-XX:+UseG1GC                    # 使用G1收集器
-XX:+UseSerialGC                # 使用Serial收集器
-XX:+UseParallelGC              # 使用Parallel收集器
-XX:+UseConcMarkSweepGC         # 使用CMS收集器
-XX:MaxGCPauseMillis=<ms>       # 最大GC停顿时间
-XX:ParallelGCThreads=<n>       # 并行GC线程数
```

#### 调试相关
```bash
-XX:+PrintGC                    # 打印GC信息
-XX:+PrintGCDetails             # 打印GC详细信息
-XX:+PrintGCTimeStamps          # 打印GC时间戳
-XX:+HeapDumpOnOutOfMemoryError # OOM时生成堆转储
-XX:HeapDumpPath=<path>         # 堆转储文件路径
```

## JVM调优检查清单

### 📋 调优前准备工作
```bash
# 1. 环境信息收集
java -version                           # 确认JDK版本
cat /proc/cpuinfo | grep processor      # 确认CPU核心数
free -h                                 # 确认内存大小
ulimit -a                               # 确认系统限制

# 2. 应用特性分析
- 应用类型：在线服务/批处理/实时计算
- 业务特点：CPU密集/IO密集/内存密集
- 访问模式：突发流量/稳定负载/周期性波动
- 对象生命周期：短生命周期对象/长生命周期对象比例
- 内存需求：预估堆内存、非堆内存需求
```

### ⚡ 快速诊断命令
```bash
# 一键收集JVM关键信息的脚本
#!/bin/bash
PID=$1
echo "=== JVM进程信息 ==="
ps aux | grep $PID

echo "=== 堆内存使用 ==="
jmap -heap $PID

echo "=== GC统计信息 ==="
jstat -gcutil $PID

echo "=== 线程信息 ==="
jstack $PID | grep -E "(BLOCKED|WAITING|TIMED_WAITING)" | wc -l

echo "=== 类加载信息 ==="
jstat -class $PID

# 使用方法：./jvm_check.sh <pid>
```

### 🎯 性能调优决策树
```
内存问题？
├── 是 → 内存泄漏？
│   ├── 是 → 堆转储分析，修复代码
│   └── 否 → 调整堆内存大小
└── 否 → GC问题？
    ├── 是 → GC频繁？
    │   ├── 是 → 年轻代过小？调整-Xmn
    │   └── 否 → GC停顿时间长？选择低延迟收集器
    └── 否 → CPU问题？
        ├── 是 → 线程竞争？优化并发控制
        └── 否 → IO瓶颈，优化业务逻辑
```

### ✅ 调优效果验证
```bash
# 1. 性能基准测试
ab -n 10000 -c 100 http://localhost:8080/api/test

# 2. 压力测试 (推荐使用JMeter或Gatling)
# 3. 长时间运行测试 (至少24小时)
# 4. 关键指标对比
#    - 吞吐量变化 (QPS/TPS)
#    - 响应时间变化 (P50/P99)
#    - GC指标变化 (频率/停顿时间)
#    - 内存使用变化
#    - CPU使用率变化
```

## 总结

JVM性能调优是一个系统性工程，需要：

1. **充分理解应用特性**：了解业务场景、访问模式、数据量等
2. **建立性能基线**：记录调优前的关键性能指标  
3. **渐进式调优**：单一变量调整，避免一次性大幅改动
4. **持续监控**：建立完善的监控体系，及时发现性能问题
5. **总结经验**：记录调优过程和效果，形成知识库

### 🏆 调优黄金法则

1. **测量优于猜测**：始终基于数据进行调优决策
2. **理解优于记忆**：理解原理比记住参数更重要  
3. **渐进优于激进**：小步调整，逐步优化
4. **监控优于修复**：预防问题比解决问题更有效
5. **适合优于最优**：适合业务的配置就是最好的配置

记住：**没有万能的调优参数，只有最适合当前应用的配置**。调优需要结合具体业务场景，通过不断测试和优化来达到最佳效果。

### 📚 推荐学习资源

- **官方文档**：Oracle JVM调优指南
- **经典书籍**：《深入理解Java虚拟机》- 周志明
- **在线工具**：GCeasy.io (GC日志分析)、Eclipse MAT (内存分析)
- **监控工具**：Prometheus + Grafana、APM工具(如SkyWalking)

---
*本文档会持续更新，欢迎补充和完善！* 