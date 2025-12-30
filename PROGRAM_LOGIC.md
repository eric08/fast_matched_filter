# Fast Matched Filter (FMF) - Program Logic Documentation
# Fast Matched Filter (FMF) - 程序逻辑文档

## Table of Contents / 目录
1. [Overview / 概述](#overview--概述)
2. [Algorithm Fundamentals / 算法基础](#algorithm-fundamentals--算法基础)
3. [Architecture / 架构](#architecture--架构)
4. [Core Components / 核心组件](#core-components--核心组件)
5. [Implementation Details / 实现细节](#implementation-details--实现细节)
6. [Data Flow / 数据流](#data-flow--数据流)
7. [Performance Optimizations / 性能优化](#performance-optimizations--性能优化)

---

## Overview / 概述

### English
Fast Matched Filter (FMF) is an efficient seismic event detection library that uses matched filtering (template matching) to find earthquakes in continuous seismological data. The library provides both CPU and GPU implementations to accelerate the correlation computation between template waveforms and continuous data streams.

**Purpose**: Detect seismic events by computing normalized cross-correlation coefficients between known earthquake templates and continuous waveform data across multiple seismic stations and channels.

**Key Features**:
- Multi-station, multi-component seismic data processing
- CPU implementation with OpenMP parallelization
- GPU implementation with CUDA for massive parallelization
- Variable template lengths support
- Network-weighted correlation coefficient summation
- High numerical precision options

### 中文
Fast Matched Filter (FMF) 是一个高效的地震事件检测库，使用匹配滤波（模板匹配）技术在连续地震数据中寻找地震事件。该库提供了 CPU 和 GPU 两种实现，以加速模板波形与连续数据流之间的相关性计算。

**目的**：通过计算已知地震模板与多个地震台站和通道的连续波形数据之间的归一化互相关系数来检测地震事件。

**主要特性**：
- 多台站、多分量地震数据处理
- 采用 OpenMP 并行化的 CPU 实现
- 采用 CUDA 实现大规模并行的 GPU 版本
- 支持可变模板长度
- 网络加权相关系数求和
- 高数值精度选项

---

## Algorithm Fundamentals / 算法基础

### English

#### Matched Filter Technique
The matched filter is a signal processing technique that searches for a known signal (template) in a noisy data stream by computing the cross-correlation:

**Basic Formula**:
```
CC(t) = Σ [T(i) × D(t+i)] / sqrt(Σ T(i)² × Σ D(t+i)²)
```

Where:
- `CC(t)` = correlation coefficient at time t
- `T(i)` = template waveform samples
- `D(t+i)` = data waveform samples starting at time t
- The normalization ensures CC ∈ [-1, 1]

#### Multi-Station Network Processing
For multiple stations and components:

```
CC_network(t) = Σ [w_s,c × CC_s,c(t + moveout_s,c)]
```

Where:
- `w_s,c` = weight for station s, component c
- `moveout_s,c` = time delay (in samples) for wave propagation
- `CC_s,c` = correlation coefficient for station s, component c

### 中文

#### 匹配滤波技术
匹配滤波是一种信号处理技术，通过计算互相关来在噪声数据流中搜索已知信号（模板）：

**基本公式**：
```
CC(t) = Σ [T(i) × D(t+i)] / sqrt(Σ T(i)² × Σ D(t+i)²)
```

其中：
- `CC(t)` = 时间 t 的相关系数
- `T(i)` = 模板波形采样点
- `D(t+i)` = 从时间 t 开始的数据波形采样点
- 归一化确保 CC ∈ [-1, 1]

#### 多台站网络处理
对于多个台站和分量：

```
CC_network(t) = Σ [w_s,c × CC_s,c(t + moveout_s,c)]
```

其中：
- `w_s,c` = 台站 s、分量 c 的权重
- `moveout_s,c` = 波传播的时间延迟（采样点数）
- `CC_s,c` = 台站 s、分量 c 的相关系数

---

## Architecture / 架构

### English

The FMF library has a layered architecture:

```
┌─────────────────────────────────────────────────┐
│         User Interface Layer                    │
│  (Python API / Matlab Interface)                │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│         Python Bindings Layer                   │
│  (fast_matched_filter.py - ctypes wrapper)      │
└─────────────────┬───────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐  ┌───────▼────────┐
│   CPU Library  │  │   GPU Library  │
│  (C + OpenMP)  │  │  (CUDA C)      │
└────────────────┘  └────────────────┘
```

**Components**:
1. **Python API** (`fast_matched_filter.py`): High-level interface
2. **C/CUDA Libraries**: Low-level computation engines
3. **Build System**: Makefile-based compilation

### 中文

FMF 库采用分层架构：

```
┌─────────────────────────────────────────────────┐
│         用户接口层                               │
│  (Python API / Matlab 接口)                     │
└─────────────────┬───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│         Python 绑定层                            │
│  (fast_matched_filter.py - ctypes 包装器)        │
└─────────────────┬───────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐  ┌───────▼────────┐
│   CPU 库       │  │   GPU 库       │
│  (C + OpenMP)  │  │  (CUDA C)      │
└────────────────┘  └────────────────┘
```

**组件**：
1. **Python API** (`fast_matched_filter.py`)：高级接口
2. **C/CUDA 库**：低级计算引擎
3. **构建系统**：基于 Makefile 的编译

---

## Core Components / 核心组件

### English

#### 1. Python Interface (`fast_matched_filter.py`)

**Main Function**: `matched_filter()`
- **Input validation**: Checks dimensions and reshapes arrays
- **Library selection**: Routes to CPU or GPU implementation
- **Parameter handling**: Manages templates, moveouts, weights, data
- **Result processing**: Reshapes output and validates results

**Key Parameters**:
- `templates`: 4D array (n_templates × n_stations × n_components × n_samples)
- `moveouts`: Time delays for each station/component
- `weights`: Channel weights for network summation
- `data`: Continuous waveform data
- `step`: Correlation time step (decimation factor)
- `arch`: Implementation choice ('cpu', 'precise', 'gpu', 'variable_precise')
- `normalize`: Normalization method ('short' or 'full')

#### 2. CPU Implementation (`matched_filter.c`)

**Main Functions**:

1. **`matched_filter()`**: Fast CPU implementation with optimized sum-of-squares
   - Uses Neumaier algorithm for cumulative sum of squared data
   - Pre-computes `csum_square_data` array for efficiency
   - OpenMP parallelization over time windows
   - Best for typical seismic amplitudes

2. **`matched_filter_precise()`**: High-precision CPU implementation
   - Recalculates sum of squared data at each correlation
   - Slower but more accurate for large amplitudes
   - Supports both 'short' and 'full' normalization
   - Better numerical stability

3. **`matched_filter_variable_precise()`**: Variable template length support
   - Allows different template lengths per channel
   - Useful for templates with varying signal durations
   - Similar precision to `matched_filter_precise()`

**Helper Functions**:
- `network_corr()`: Computes weighted sum of correlations across network
- `network_corr_precise()`: High-precision version
- `cumsum_square_data()`: Pre-computes cumulative sum of squared data

#### 3. GPU Implementation (`matched_filter.cu`)

**Architecture**:
- CUDA kernel-based parallelization
- Each thread processes one correlation at one time point
- Shared memory usage for template and data caching
- Block-level parallelization across stations

**Key Kernel**: `network_corr()`
- Processes correlations in parallel across GPU blocks
- Uses shared memory for template and data segments
- Handles moveout-adjusted data alignment
- Computes normalized cross-correlation per channel

**Memory Management**:
- Chunked processing to fit GPU memory constraints
- Dynamic allocation based on data size
- Efficient data transfer between host and device

### 中文

#### 1. Python 接口 (`fast_matched_filter.py`)

**主函数**：`matched_filter()`
- **输入验证**：检查维度并重塑数组
- **库选择**：路由到 CPU 或 GPU 实现
- **参数处理**：管理模板、移动、权重、数据
- **结果处理**：重塑输出并验证结果

**关键参数**：
- `templates`：4D 数组（n_templates × n_stations × n_components × n_samples）
- `moveouts`：每个台站/分量的时间延迟
- `weights`：网络求和的通道权重
- `data`：连续波形数据
- `step`：相关时间步长（抽取因子）
- `arch`：实现选择（'cpu', 'precise', 'gpu', 'variable_precise'）
- `normalize`：归一化方法（'short' 或 'full'）

#### 2. CPU 实现 (`matched_filter.c`)

**主要函数**：

1. **`matched_filter()`**：优化平方和的快速 CPU 实现
   - 使用 Neumaier 算法计算数据平方的累积和
   - 预先计算 `csum_square_data` 数组以提高效率
   - 在时间窗口上进行 OpenMP 并行化
   - 最适合典型地震振幅

2. **`matched_filter_precise()`**：高精度 CPU 实现
   - 在每次相关计算时重新计算数据平方和
   - 速度较慢但对大振幅更准确
   - 支持 'short' 和 'full' 两种归一化
   - 更好的数值稳定性

3. **`matched_filter_variable_precise()`**：支持可变模板长度
   - 允许每个通道使用不同的模板长度
   - 适用于具有不同信号持续时间的模板
   - 精度类似于 `matched_filter_precise()`

**辅助函数**：
- `network_corr()`：计算网络中相关性的加权和
- `network_corr_precise()`：高精度版本
- `cumsum_square_data()`：预计算数据平方的累积和

#### 3. GPU 实现 (`matched_filter.cu`)

**架构**：
- 基于 CUDA 内核的并行化
- 每个线程处理一个时间点的一次相关
- 使用共享内存缓存模板和数据
- 跨台站的块级并行化

**关键内核**：`network_corr()`
- 跨 GPU 块并行处理相关性
- 使用共享内存存储模板和数据段
- 处理移动调整的数据对齐
- 计算每个通道的归一化互相关

**内存管理**：
- 分块处理以适应 GPU 内存限制
- 根据数据大小动态分配
- 主机和设备之间的高效数据传输

---

## Implementation Details / 实现细节

### English

#### CPU Processing Flow

1. **Initialization**:
   ```c
   // Allocate memory for cumulative sum of squared data
   csum_square_data = malloc(...);
   cumsum_square_data(data, ...);  // Pre-compute
   ```

2. **Template Loop**:
   ```c
   for (size_t t = 0; t < n_templates; t++) {
       // Find min/max moveouts for this template
       // Determine valid correlation time range
       
       #pragma omp parallel for
       for (size_t i = start_i; i < stop_i; i += step) {
           // Compute correlation at time i
           cc_sum[...] = network_corr(...);
       }
   }
   ```

3. **Correlation Computation** (per time point):
   ```c
   float network_corr(...) {
       for (each station) {
           for (each component) {
               // Compute numerator: Σ(template × data)
               // Compute denominator: sqrt(Σ template² × Σ data²)
               // Apply weights
               cc_sum += weight × (numerator / denominator);
           }
       }
       return cc_sum;
   }
   ```

#### GPU Processing Flow

1. **Memory Transfer**:
   - Copy templates, moveouts, weights to GPU
   - Process data in chunks to fit GPU memory

2. **Kernel Launch**:
   ```cuda
   // Launch grid of blocks, each processing multiple correlations
   network_corr<<<grid_dim, block_dim, shared_mem>>>(...)
   ```

3. **Thread-Level Processing**:
   ```cuda
   __global__ void network_corr(...) {
       // Load template to shared memory
       // Load data window to shared memory
       __syncthreads();
       
       // Each thread computes one correlation
       // Numerator: dot product
       // Denominator: sqrt of sum of squares
       // Store result
   }
   ```

#### Normalization Methods

**Short Normalization** (`normalize='short'`):
- Assumes templates and data have zero mean
- Faster computation
- Original implementation
- Formula: `CC = Σ(T×D) / sqrt(Σ T² × Σ D²)`

**Full Normalization** (`normalize='full'`):
- Removes mean at each correlation window
- Slower but more robust
- Better for data with DC offset
- Formula: `CC = Σ((T-μT)×(D-μD)) / sqrt(Σ(T-μT)² × Σ(D-μD)²)`

### 中文

#### CPU 处理流程

1. **初始化**：
   ```c
   // 为数据平方的累积和分配内存
   csum_square_data = malloc(...);
   cumsum_square_data(data, ...);  // 预计算
   ```

2. **模板循环**：
   ```c
   for (size_t t = 0; t < n_templates; t++) {
       // 查找此模板的最小/最大移动
       // 确定有效的相关时间范围
       
       #pragma omp parallel for
       for (size_t i = start_i; i < stop_i; i += step) {
           // 计算时间 i 的相关性
           cc_sum[...] = network_corr(...);
       }
   }
   ```

3. **相关性计算**（每个时间点）：
   ```c
   float network_corr(...) {
       for (each station) {
           for (each component) {
               // 计算分子：Σ(模板 × 数据)
               // 计算分母：sqrt(Σ 模板² × Σ 数据²)
               // 应用权重
               cc_sum += weight × (numerator / denominator);
           }
       }
       return cc_sum;
   }
   ```

#### GPU 处理流程

1. **内存传输**：
   - 将模板、移动、权重复制到 GPU
   - 分块处理数据以适应 GPU 内存

2. **内核启动**：
   ```cuda
   // 启动块网格，每个块处理多个相关性
   network_corr<<<grid_dim, block_dim, shared_mem>>>(...)
   ```

3. **线程级处理**：
   ```cuda
   __global__ void network_corr(...) {
       // 将模板加载到共享内存
       // 将数据窗口加载到共享内存
       __syncthreads();
       
       // 每个线程计算一次相关
       // 分子：点积
       // 分母：平方和的平方根
       // 存储结果
   }
   ```

#### 归一化方法

**短归一化** (`normalize='short'`)：
- 假设模板和数据具有零均值
- 计算速度更快
- 原始实现
- 公式：`CC = Σ(T×D) / sqrt(Σ T² × Σ D²)`

**完全归一化** (`normalize='full'`)：
- 在每个相关窗口移除均值
- 速度较慢但更稳健
- 更适合有直流偏移的数据
- 公式：`CC = Σ((T-μT)×(D-μD)) / sqrt(Σ(T-μT)² × Σ(D-μD)²)`

---

## Data Flow / 数据流

### English

```
Input Data
    │
    ├─ Templates: [n_templates, n_stations, n_components, n_samples_template]
    ├─ Data: [n_stations, n_components, n_samples_data]
    ├─ Moveouts: [n_templates, n_stations, n_components]
    └─ Weights: [n_templates, n_stations, n_components]
    │
    ▼
Python Validation & Reshaping
    │
    ├─ Dimension checks
    ├─ Array flattening
    └─ C-contiguous memory layout
    │
    ▼
C Library Call (ctypes)
    │
    ├─ matched_filter (CPU fast)
    ├─ matched_filter_precise (CPU accurate)
    ├─ matched_filter_variable_precise (CPU variable length)
    └─ matched_filter (GPU)
    │
    ▼
Core Computation
    │
    ├─ For each template:
    │   ├─ Determine valid time range based on moveouts
    │   └─ For each time step:
    │       ├─ For each station/component:
    │       │   ├─ Align data with moveout
    │       │   ├─ Compute correlation coefficient
    │       │   └─ Apply weight
    │       └─ Sum across network (if network_sum=True)
    │
    ▼
Output
    │
    ├─ network_sum=True: [n_templates, n_correlations]
    └─ network_sum=False: [n_templates, n_stations, n_components, n_correlations]
```

### 中文

```
输入数据
    │
    ├─ 模板：[n_templates, n_stations, n_components, n_samples_template]
    ├─ 数据：[n_stations, n_components, n_samples_data]
    ├─ 移动：[n_templates, n_stations, n_components]
    └─ 权重：[n_templates, n_stations, n_components]
    │
    ▼
Python 验证与重塑
    │
    ├─ 维度检查
    ├─ 数组展平
    └─ C 连续内存布局
    │
    ▼
C 库调用（ctypes）
    │
    ├─ matched_filter（CPU 快速）
    ├─ matched_filter_precise（CPU 精确）
    ├─ matched_filter_variable_precise（CPU 可变长度）
    └─ matched_filter（GPU）
    │
    ▼
核心计算
    │
    ├─ 对于每个模板：
    │   ├─ 根据移动确定有效时间范围
    │   └─ 对于每个时间步：
    │       ├─ 对于每个台站/分量：
    │       │   ├─ 根据移动对齐数据
    │       │   ├─ 计算相关系数
    │       │   └─ 应用权重
    │       └─ 跨网络求和（如果 network_sum=True）
    │
    ▼
输出
    │
    ├─ network_sum=True：[n_templates, n_correlations]
    └─ network_sum=False：[n_templates, n_stations, n_components, n_correlations]
```

---

## Performance Optimizations / 性能优化

### English

#### CPU Optimizations

1. **OpenMP Parallelization**:
   - Time-window level parallelization with `#pragma omp parallel for`
   - Each thread processes independent time windows
   - Good load balancing for uniform moveouts

2. **Cumulative Sum Trick**:
   - Pre-compute cumulative sum of squared data
   - Extract sum over any window in O(1) time
   - Trade memory for speed (Neumaier algorithm)
   - Formula: `Σ(data[i:j]²) = csum[j] - csum[i]`

3. **Memory Layout**:
   - C-contiguous arrays for cache efficiency
   - Minimize pointer arithmetic overhead
   - Aligned memory access patterns

4. **SIMD Vectorization**:
   - Compiler flags: `-ftree-vectorize -march=native`
   - Auto-vectorization of inner loops
   - Platform-specific optimizations

#### GPU Optimizations

1. **Massive Parallelism**:
   - Thousands of concurrent threads
   - Each correlation computed independently
   - Scales with number of CUDA cores

2. **Shared Memory**:
   - Cache templates in fast shared memory
   - Reduce global memory bandwidth requirements
   - Coalesced memory access patterns

3. **Chunked Processing**:
   - Split large datasets into GPU-sized chunks
   - Overlap computation with data transfer
   - Avoid memory overflow

4. **Warp-Level Optimization**:
   - Align block dimensions to warp size (32)
   - Minimize warp divergence
   - Efficient use of GPU resources

#### Algorithm Optimizations

1. **Step Parameter**:
   - Decimate correlation output by factor of `step`
   - Reduces computation time proportionally
   - Trade temporal resolution for speed

2. **Weight Zeroing**:
   - Skip channels with zero weights
   - Reduces unnecessary computation
   - Automatic in correlation functions

3. **Moveout Range Optimization**:
   - Compute valid time range per template
   - Skip invalid correlations at data boundaries
   - Prevents out-of-bounds access

### 中文

#### CPU 优化

1. **OpenMP 并行化**：
   - 使用 `#pragma omp parallel for` 进行时间窗口级并行化
   - 每个线程处理独立的时间窗口
   - 对于均匀移动具有良好的负载平衡

2. **累积和技巧**：
   - 预计算数据平方的累积和
   - 在 O(1) 时间内提取任意窗口的和
   - 用内存换速度（Neumaier 算法）
   - 公式：`Σ(data[i:j]²) = csum[j] - csum[i]`

3. **内存布局**：
   - C 连续数组以提高缓存效率
   - 最小化指针算术开销
   - 对齐的内存访问模式

4. **SIMD 矢量化**：
   - 编译器标志：`-ftree-vectorize -march=native`
   - 内循环的自动矢量化
   - 平台特定优化

#### GPU 优化

1. **大规模并行**：
   - 数千个并发线程
   - 每个相关性独立计算
   - 随 CUDA 核心数量扩展

2. **共享内存**：
   - 在快速共享内存中缓存模板
   - 减少全局内存带宽需求
   - 合并的内存访问模式

3. **分块处理**：
   - 将大数据集分割成 GPU 大小的块
   - 计算与数据传输重叠
   - 避免内存溢出

4. **Warp 级优化**：
   - 将块维度对齐到 warp 大小（32）
   - 最小化 warp 分歧
   - 有效使用 GPU 资源

#### 算法优化

1. **步长参数**：
   - 按 `step` 因子抽取相关输出
   - 按比例减少计算时间
   - 用时间分辨率换速度

2. **权重归零**：
   - 跳过权重为零的通道
   - 减少不必要的计算
   - 在相关函数中自动进行

3. **移动范围优化**：
   - 计算每个模板的有效时间范围
   - 在数据边界跳过无效相关
   - 防止越界访问

---

## Usage Examples / 使用示例

### English

#### Basic Usage
```python
import fast_matched_filter as fmf
import numpy as np

# Load your data
templates = np.load('templates.npy')  # Shape: (n_templates, n_stations, n_components, n_samples_template)
data = np.load('data.npy')            # Shape: (n_stations, n_components, n_samples_data)
moveouts = np.load('moveouts.npy')    # Shape: (n_templates, n_stations, n_components)

# Set weights (e.g., uniform)
n_templates, n_stations, n_components, _ = templates.shape
weights = np.ones((n_templates, n_stations, n_components)) / (n_stations * n_components)

# Run matched filter
cc = fmf.matched_filter(
    templates=templates,
    moveouts=moveouts,
    weights=weights,
    data=data,
    step=1,              # Compute correlation at every sample
    arch='cpu',          # Use CPU implementation
    normalize='short',   # Assume zero-mean data
    network_sum=True     # Return network-summed correlations
)

# cc.shape = (n_templates, n_correlations)
# Find detections
threshold = 0.5
detections = np.where(cc > threshold)
```

#### Testing
```python
# Run built-in test
templates, moveouts, data, step, cc, runtime = fmf.test_matched_filter(
    n_templates=10,
    n_stations=5,
    n_components=3,
    template_duration=10,     # seconds
    data_duration=3600,       # seconds
    sampling_rate=100,        # Hz
    step=10,
    arch='cpu'
)

print(f"Computation took {runtime:.2f} seconds")
print(f"Max correlation: {np.max(cc):.3f}")  # Should be close to 1.0
```

### 中文

#### 基本使用
```python
import fast_matched_filter as fmf
import numpy as np

# 加载您的数据
templates = np.load('templates.npy')  # 形状：(n_templates, n_stations, n_components, n_samples_template)
data = np.load('data.npy')            # 形状：(n_stations, n_components, n_samples_data)
moveouts = np.load('moveouts.npy')    # 形状：(n_templates, n_stations, n_components)

# 设置权重（例如，均匀权重）
n_templates, n_stations, n_components, _ = templates.shape
weights = np.ones((n_templates, n_stations, n_components)) / (n_stations * n_components)

# 运行匹配滤波
cc = fmf.matched_filter(
    templates=templates,
    moveouts=moveouts,
    weights=weights,
    data=data,
    step=1,              # 在每个采样点计算相关性
    arch='cpu',          # 使用 CPU 实现
    normalize='short',   # 假设零均值数据
    network_sum=True     # 返回网络求和的相关性
)

# cc.shape = (n_templates, n_correlations)
# 查找检测
threshold = 0.5
detections = np.where(cc > threshold)
```

#### 测试
```python
# 运行内置测试
templates, moveouts, data, step, cc, runtime = fmf.test_matched_filter(
    n_templates=10,
    n_stations=5,
    n_components=3,
    template_duration=10,     # 秒
    data_duration=3600,       # 秒
    sampling_rate=100,        # Hz
    step=10,
    arch='cpu'
)

print(f"计算耗时 {runtime:.2f} 秒")
print(f"最大相关性：{np.max(cc):.3f}")  # 应该接近 1.0
```

---

## Key Design Decisions / 关键设计决策

### English

1. **Separation of CPU and GPU Code**:
   - Different compilation paths (gcc vs nvcc)
   - Shared interface through Python bindings
   - Allows independent optimization

2. **Ctypes for Python Bindings**:
   - No Cython dependency
   - Direct C function calls
   - Simple build process
   - Easy to debug

3. **Flattened Array Representation**:
   - Converts multi-dimensional arrays to 1D
   - Simplifies C function signatures
   - Index calculation in C code
   - Memory efficiency

4. **Template-by-Template Processing**:
   - Reduces memory footprint
   - Enables better cache utilization
   - Simplifies OpenMP parallelization
   - Sequential template loop, parallel time loop

5. **Optional Network Summation**:
   - `network_sum=True`: Returns summed correlations (smaller output)
   - `network_sum=False`: Returns per-channel correlations (detailed analysis)
   - Different use cases supported

### 中文

1. **CPU 和 GPU 代码分离**：
   - 不同的编译路径（gcc vs nvcc）
   - 通过 Python 绑定共享接口
   - 允许独立优化

2. **使用 Ctypes 进行 Python 绑定**：
   - 无需 Cython 依赖
   - 直接 C 函数调用
   - 简单的构建过程
   - 易于调试

3. **展平的数组表示**：
   - 将多维数组转换为 1D
   - 简化 C 函数签名
   - C 代码中的索引计算
   - 内存效率

4. **逐模板处理**：
   - 减少内存占用
   - 实现更好的缓存利用
   - 简化 OpenMP 并行化
   - 顺序模板循环，并行时间循环

5. **可选的网络求和**：
   - `network_sum=True`：返回求和的相关性（较小的输出）
   - `network_sum=False`：返回每个通道的相关性（详细分析）
   - 支持不同的用例

---

## References / 参考文献

### Paper / 论文
Beaucé, Eric, W. B. Frank, and Alexey Romanenko (2017). Fast matched-filter (FMF): an efficient seismic matched-filter search for both CPU and GPU architectures. _Seismological Research Letters_, doi: [10.1785/0220170181](https://doi.org/10.1785/0220170181)

### Documentation / 文档
- Official Documentation: https://ebeauce.github.io/FMF_documentation/
- GitHub Repository: https://github.com/beridel/fast_matched_filter

---

## Summary / 总结

### English
Fast Matched Filter is a high-performance seismic event detection library that efficiently computes normalized cross-correlations between template earthquakes and continuous seismological data. The library leverages both CPU (with OpenMP) and GPU (with CUDA) parallelization to achieve significant speedups over traditional implementations. Key features include support for multi-station networks, variable template lengths, flexible normalization methods, and both summed and per-channel output options. The implementation uses careful algorithm optimization (cumulative sums, memory layout, vectorization) to maximize performance while maintaining numerical accuracy.

### 中文
Fast Matched Filter 是一个高性能的地震事件检测库，可以高效计算模板地震与连续地震数据之间的归一化互相关。该库利用 CPU（使用 OpenMP）和 GPU（使用 CUDA）并行化，相比传统实现实现了显著的加速。主要特性包括支持多台站网络、可变模板长度、灵活的归一化方法以及求和和逐通道输出选项。该实现使用了精心的算法优化（累积和、内存布局、矢量化）以在保持数值精度的同时最大化性能。
