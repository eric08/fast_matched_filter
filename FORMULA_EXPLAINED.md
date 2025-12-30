# 归一化互相关公式详解 / Normalized Cross-Correlation Formula Explained

## 公式 / Formula

```
CC(t) = Σ [T(i) × D(t+i)] / sqrt(Σ T(i)² × Σ D(t+i)²)
```

---

## 详细解释 / Detailed Explanation

### 中文详解

#### 1. 公式的物理意义

这个公式计算的是**归一化互相关系数**（Normalized Cross-Correlation Coefficient），它衡量模板信号 T 和数据信号 D 在时间 t 处的相似程度。

#### 2. 公式的三个部分

##### 第一部分：分子 - 互相关（Cross-Correlation）
```
分子 = Σ [T(i) × D(t+i)]
```

**含义**：这是模板和数据的点积（dot product）
- `T(i)`: 模板的第 i 个采样点的值
- `D(t+i)`: 数据从时间 t 开始的第 i 个采样点的值
- `Σ`: 对所有 i 求和，其中 i = 0, 1, 2, ..., N-1（N 是模板长度）

**举例说明**：
```
假设模板长度 N = 3
模板 T = [1.0, 2.0, 3.0]
数据在时间 t 处的窗口 D = [0.5, 1.0, 1.5]

分子 = T(0)×D(t+0) + T(1)×D(t+1) + T(2)×D(t+2)
     = 1.0×0.5 + 2.0×1.0 + 3.0×1.5
     = 0.5 + 2.0 + 4.5
     = 7.0
```

**意义**：
- 如果模板和数据波形相似，分子会很大（正值）
- 如果波形相反，分子会是负值
- 如果波形不相关，分子接近零

##### 第二部分：分母第一项 - 模板的能量
```
Σ T(i)²
```

**含义**：模板信号的总能量（平方和）
- 对模板的每个采样点求平方，然后求和
- 这是一个常数，只需要计算一次

**举例说明**：
```
模板 T = [1.0, 2.0, 3.0]

Σ T(i)² = T(0)² + T(1)² + T(2)²
        = 1.0² + 2.0² + 3.0²
        = 1.0 + 4.0 + 9.0
        = 14.0
```

##### 第三部分：分母第二项 - 数据窗口的能量
```
Σ D(t+i)²
```

**含义**：数据在时间 t 处的窗口的总能量
- 对数据窗口的每个采样点求平方，然后求和
- 这个值随着时间 t 的变化而变化

**举例说明**：
```
数据窗口 D = [0.5, 1.0, 1.5]

Σ D(t+i)² = D(t+0)² + D(t+1)² + D(t+2)²
          = 0.5² + 1.0² + 1.5²
          = 0.25 + 1.0 + 2.25
          = 3.5
```

##### 完整的分母
```
分母 = sqrt(Σ T(i)² × Σ D(t+i)²)
```

**含义**：这是模板能量和数据窗口能量的几何平均数
- 使用平方根确保分母的单位与分子匹配

**举例说明**：
```
分母 = sqrt(14.0 × 3.5)
     = sqrt(49.0)
     = 7.0
```

#### 3. 完整计算示例

```
CC(t) = 分子 / 分母
      = 7.0 / 7.0
      = 1.0
```

**结果解释**：CC(t) = 1.0 表示完美匹配（在这个例子中，数据恰好是模板的 0.5 倍缩放）

#### 4. 归一化的目的

分母的作用是**归一化**，确保：
- **结果范围**：CC(t) 总是在 [-1, 1] 之间
- **消除幅度影响**：不论信号的绝对幅度如何，只关注波形的形状相似度
- **+1**：完全正相关（波形形状完全相同）
- **0**：不相关（波形无关）
- **-1**：完全负相关（波形形状相反）

#### 5. 数学背景：Pearson 相关系数

这个公式本质上是**皮尔逊相关系数**（Pearson Correlation Coefficient）的简化形式，前提是：
- 模板 T 已经过预处理，均值为 0
- 数据 D 也经过高通滤波，均值接近 0

如果信号有非零均值，完整公式应该是：
```
CC(t) = Σ [(T(i)-μ_T) × (D(t+i)-μ_D)] / sqrt(Σ(T(i)-μ_T)² × Σ(D(t+i)-μ_D)²)
```

其中 μ_T 和 μ_D 分别是模板和数据窗口的均值。

FMF 的 `normalize='short'` 模式假设信号已经去均值化，使用简化公式。
FMF 的 `normalize='full'` 模式在每次计算时去除均值，使用完整公式。

#### 6. 在 FMF 中的实现

**CPU 实现**（fast_matched_filter/src/matched_filter.c）：
```c
// 计算分子（numerator）
float numerator = 0.0f;
for (int i = 0; i < n_samples_template; i++) {
    numerator += templates[i] * data[t + i];
}

// 分母中的模板能量已预先计算
float sum_square_template = ...; // 预先计算的 Σ T(i)²

// 计算数据窗口的能量
float sum_square_data = 0.0f;
for (int i = 0; i < n_samples_template; i++) {
    sum_square_data += data[t + i] * data[t + i];
}

// 计算相关系数
float denominator = sqrt(sum_square_template * sum_square_data);
if (denominator > STABILITY_THRESHOLD) {
    cc = numerator / denominator;
} else {
    cc = 0.0f;  // 避免除以零
}
```

**优化技巧**：
- `Σ T(i)²` 只需计算一次（在 Python 层完成）
- `Σ D(t+i)²` 使用累积和技巧快速计算（Neumaier 算法）

#### 7. 多台站网络扩展

对于多个台站和分量，每个通道计算一个 CC 值，然后加权求和：

```
CC_network(t) = Σ [w_s,c × CC_s,c(t + moveout_s,c)]
              s,c
```

其中：
- `s` = 台站索引
- `c` = 分量索引（例如：E、N、Z 三个分量）
- `w_s,c` = 通道权重（通常归一化使 Σw = 1）
- `moveout_s,c` = 波从震源到台站 s 的传播时间（单位：采样点）

**举例说明**：
```
假设有 2 个台站，每个台站 1 个分量
台站 1 的 moveout = 10 samples，权重 = 0.5
台站 2 的 moveout = 20 samples，权重 = 0.5

在时间 t = 100 处：
CC_1 在 t=100+10=110 处计算，得到 CC_1 = 0.8
CC_2 在 t=100+20=120 处计算，得到 CC_2 = 0.7

CC_network(100) = 0.5 × 0.8 + 0.5 × 0.7
                = 0.4 + 0.35
                = 0.75
```

这样可以综合多个台站的观测，提高检测的可靠性。

---

### English Explanation

#### 1. Physical Meaning of the Formula

This formula computes the **Normalized Cross-Correlation Coefficient (NCC)**, which measures the similarity between template signal T and data signal D at time t.

#### 2. Three Parts of the Formula

##### Part 1: Numerator - Cross-Correlation
```
Numerator = Σ [T(i) × D(t+i)]
```

**Meaning**: This is the dot product of the template and data
- `T(i)`: Value of the i-th sample in the template
- `D(t+i)`: Value of the i-th sample in the data window starting at time t
- `Σ`: Sum over all i, where i = 0, 1, 2, ..., N-1 (N is template length)

**Example**:
```
Assume template length N = 3
Template T = [1.0, 2.0, 3.0]
Data window at time t: D = [0.5, 1.0, 1.5]

Numerator = T(0)×D(t+0) + T(1)×D(t+1) + T(2)×D(t+2)
          = 1.0×0.5 + 2.0×1.0 + 3.0×1.5
          = 0.5 + 2.0 + 4.5
          = 7.0
```

**Interpretation**:
- If template and data waveforms are similar, numerator is large (positive)
- If waveforms are opposite, numerator is negative
- If waveforms are uncorrelated, numerator is near zero

##### Part 2: Denominator First Term - Template Energy
```
Σ T(i)²
```

**Meaning**: Total energy (sum of squares) of template signal
- Square each template sample, then sum
- This is a constant, computed only once

**Example**:
```
Template T = [1.0, 2.0, 3.0]

Σ T(i)² = T(0)² + T(1)² + T(2)²
        = 1.0² + 2.0² + 3.0²
        = 1.0 + 4.0 + 9.0
        = 14.0
```

##### Part 3: Denominator Second Term - Data Window Energy
```
Σ D(t+i)²
```

**Meaning**: Total energy of data window at time t
- Square each sample in data window, then sum
- This value changes as time t changes

**Example**:
```
Data window D = [0.5, 1.0, 1.5]

Σ D(t+i)² = D(t+0)² + D(t+1)² + D(t+2)²
          = 0.5² + 1.0² + 1.5²
          = 0.25 + 1.0 + 2.25
          = 3.5
```

##### Complete Denominator
```
Denominator = sqrt(Σ T(i)² × Σ D(t+i)²)
```

**Meaning**: Geometric mean of template and data window energies
- Square root ensures denominator units match numerator

**Example**:
```
Denominator = sqrt(14.0 × 3.5)
            = sqrt(49.0)
            = 7.0
```

#### 3. Complete Calculation Example

```
CC(t) = Numerator / Denominator
      = 7.0 / 7.0
      = 1.0
```

**Result Interpretation**: CC(t) = 1.0 indicates perfect match (in this example, data is exactly 0.5× scaled version of template)

#### 4. Purpose of Normalization

The denominator **normalizes** to ensure:
- **Result range**: CC(t) always in [-1, 1]
- **Removes amplitude effects**: Only cares about waveform shape similarity, not absolute amplitude
- **+1**: Perfect positive correlation (waveforms identical in shape)
- **0**: Uncorrelated (waveforms unrelated)
- **-1**: Perfect negative correlation (waveforms opposite in shape)

#### 5. Mathematical Background: Pearson Correlation

This formula is essentially a simplified **Pearson Correlation Coefficient**, assuming:
- Template T is pre-processed to have zero mean
- Data D is high-pass filtered to have near-zero mean

For signals with non-zero mean, the complete formula is:
```
CC(t) = Σ [(T(i)-μ_T) × (D(t+i)-μ_D)] / sqrt(Σ(T(i)-μ_T)² × Σ(D(t+i)-μ_D)²)
```

Where μ_T and μ_D are the means of template and data window respectively.

FMF's `normalize='short'` mode assumes signals are already demeaned (simplified formula).
FMF's `normalize='full'` mode removes mean at each computation (complete formula).

#### 6. Implementation in FMF

**CPU Implementation** (fast_matched_filter/src/matched_filter.c):
```c
// Compute numerator
float numerator = 0.0f;
for (int i = 0; i < n_samples_template; i++) {
    numerator += templates[i] * data[t + i];
}

// Template energy pre-computed
float sum_square_template = ...; // Pre-computed Σ T(i)²

// Compute data window energy
float sum_square_data = 0.0f;
for (int i = 0; i < n_samples_template; i++) {
    sum_square_data += data[t + i] * data[t + i];
}

// Compute correlation coefficient
float denominator = sqrt(sum_square_template * sum_square_data);
if (denominator > STABILITY_THRESHOLD) {
    cc = numerator / denominator;
} else {
    cc = 0.0f;  // Avoid division by zero
}
```

**Optimization Tricks**:
- `Σ T(i)²` computed once (in Python layer)
- `Σ D(t+i)²` computed efficiently using cumulative sum trick (Neumaier algorithm)

#### 7. Multi-Station Network Extension

For multiple stations and components, compute one CC per channel, then weighted sum:

```
CC_network(t) = Σ [w_s,c × CC_s,c(t + moveout_s,c)]
              s,c
```

Where:
- `s` = station index
- `c` = component index (e.g., E, N, Z components)
- `w_s,c` = channel weight (typically normalized so Σw = 1)
- `moveout_s,c` = wave propagation time from source to station s (in samples)

**Example**:
```
Assume 2 stations, 1 component each
Station 1: moveout = 10 samples, weight = 0.5
Station 2: moveout = 20 samples, weight = 0.5

At time t = 100:
CC_1 computed at t=100+10=110, result CC_1 = 0.8
CC_2 computed at t=100+20=120, result CC_2 = 0.7

CC_network(100) = 0.5 × 0.8 + 0.5 × 0.7
                = 0.4 + 0.35
                = 0.75
```

This combines observations from multiple stations to improve detection reliability.

---

## 总结 / Summary

### 中文
归一化互相关公式 `CC(t) = Σ [T(i) × D(t+i)] / sqrt(Σ T(i)² × Σ D(t+i)²)` 是模板匹配的核心。它通过计算模板和数据的点积（分子）并用它们的能量几何平均数（分母）归一化，得到一个 [-1, 1] 范围内的相似度度量。这使得算法能够识别波形形状的相似性，而不受信号幅度的影响，这对于地震检测至关重要。

### English
The normalized cross-correlation formula `CC(t) = Σ [T(i) × D(t+i)] / sqrt(Σ T(i)² × Σ D(t+i)²)` is the core of template matching. It computes the dot product of template and data (numerator) and normalizes by their geometric mean energy (denominator), yielding a similarity measure in [-1, 1]. This allows the algorithm to identify waveform shape similarity regardless of signal amplitude, which is crucial for seismic detection.
