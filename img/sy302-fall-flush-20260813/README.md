# SY302 跌倒检测冲水误报优化仿真报告

> 数据批次：0729；雷达顶装高度：2.4 m；检测半径：135 cm。
>
> 标签口径：躺地必须报警；坐马桶和冲水整段不得报警；坐地、下蹲为灰区，报警或不报警均可。

## 1. 结论摘要

在本批 219 个文件上，新增“人体形态 + 冲水紧凑簇”门控后：

- 12 个冲水文件中，当前 C 基线算法有 7 个文件误报，优化后为 0 个；
- 48 个坐马桶文件和 15 个走动文件，优化前后均为 0 个误报；
- 当前基线检出的 27 个躺地文件在优化后全部保留，没有因冲水门控新增漏报；
- 躺地总检出率仍为 27/48（56.25%）。其余 21 个漏报来自原有高度下降和确认状态机，不是本次门控造成；
- 坐地报警 5/48、下蹲报警 0/48，按需求作为灰区单独展示，不计入总体指标。

因此，本轮优化在现有数据上实现了“清除可复现冲水误报，同时不损失原本已检出的躺地样本”。下一阶段应独立处理躺地漏检问题，并增加不同卫生间、马桶位置及马桶附近真实跌倒数据验证。

## 2. 数据与坐标转换

冲水数据为 8 列：

```text
帧索引 点类型 距离 水平角 俯仰角 速度 功率 SNR
```

其中 `0=静态点、1=动态点`，距离和坐标单位为 cm。坐标严格按固件 `coordinate_conversion()` 的等价关系转换：

```text
z_down = distance / sqrt(1 + tan(ant13)^2 + tan(ant23)^2)
x      = z_down * tan(ant13)
y      = z_down * tan(ant23)
height = 240 - z_down
```

正常动作数据为 6 列：

```text
帧索引 动静态标识 速度索引 x y z
```

所有点均按 135 cm 检测半径参与复现。少量 UART 断行、粘包或字段异常记录由读取器丢弃并计数。

## 3. 冲水与人体的特征差异

![冲水误报与人体动作特征对比](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/feature_comparison.png)

四幅箱线图从左上到右下分别表示：

1. **低姿态水平展宽**：点云在水平面的 RMS 尺度。冲水通常约 7 cm，躺地通常约 20 cm；红色虚线 10.5 cm 是人体展宽阈值。
2. **低姿态垂直跨度**：同一帧最高点与最低点的高度差。红色虚线 16 cm 是人体垂直跨度阈值。
3. **低姿态点数**：每帧有效点数。红色虚线 14 点用于识别密集簇。
4. **低姿态中心半径**：点云质心到雷达垂直投影点的水平距离。红色虚线 88 cm 是本批冲水簇半径下限。

箱体中线是中位数，箱体上下边分别为第 25/75 百分位，须线表示常规分布范围，空心圆为离群样本。单一特征会有重叠，因此最终使用“密集 + 远离中心 + 水平紧凑”的联合条件识别冲水簇。

## 4. 优化判据

每个已经满足低姿态条件的证据帧计算：

```text
human_shape = horizontal_rms >= 10.5 cm
              && vertical_span >= 16 cm
              && !flush_like

flush_like = point_count >= 14
             && centroid_radius in [88, 115] cm
             && horizontal_rms <= 14 cm
```

在原 C 算法的快速跌倒或补充跌倒确认条件已经满足后，再要求：

```text
human_shape_count / low_evidence_count >= 0.30
flush_like_count  / low_evidence_count <= 0.25
```

这项门控不改变原有高度滤波、20 秒主路径和 40 秒补充路径，只在算法准备上报时判断累计低姿态证据是否更像人体。

## 5. 文件级结果

| 类别 | 标签口径 | 文件数 | 当前 C 报警 | 优化后报警 |
|---|---|---:|---:|---:|
| 冲水误报 | 负样本，整段不得报警 | 12 | 7 | 0 |
| 坐马桶 | 负样本 | 48 | 0 | 0 |
| 走动 | 负样本 | 15 | 0 | 0 |
| 下蹲 | 灰区 | 48 | 0 | 0 |
| 坐地 | 灰区 | 48 | 5 | 5 |
| 躺地 | 正样本，必须报警 | 48 | 27 | 27 |

![各动作类别优化前后报警率](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/alarm_rate_by_class.png)

坐地和下蹲排除后的总体指标：

| 算法 | TP | FN | FP | TN | 召回率 | 假阳性率 | 精确率 | 准确率 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 当前 C 基线 | 27 | 21 | 7 | 68 | 56.25% | 9.33% | 79.41% | 77.24% |
| 优化后 | 27 | 21 | 0 | 75 | 56.25% | 0% | 100% | 82.93% |

## 6. 代表性冲水误报时序

![代表性冲水误报时序](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/representative_flush_timeline.png)

图中各层含义：

- 第一层灰线是帧最大高度，蓝线是按 C 算法处理后的平滑高度，紫色虚线是动态低姿态阈值，红色点线是补充上报高度阈值；
- 第二层蓝线是水平 RMS 展宽，橙线是垂直跨度，水平虚线为各自人体形态阈值；
- 第三层绿色阶梯是人体形态判定，橙色阶梯是冲水簇判定；红点代表基线报警，黑点代表该报警被优化门控拦截；
- 第四层紫线是 300 帧窗口低姿态占比，绿线是低姿态证据中人体形态占比，红色虚线为形态占比阈值 0.30。

本例中高度条件足以让基线算法进入报警，但低姿态点云长期呈“紧凑冲水簇”而非人体展开形态，因此优化门控拦截报警。

## 7. 全部冲水文件结果

| 编号 | 基线报警 | 优化后报警 | 基线报警时刻 / s | 报警时人体形态占比 | 报警时冲水簇占比 |
|---:|---:|---:|---:|---:|---:|
| 01 | 是 | 否 | 38.0 | 0.418 | 0.449 |
| 02 | 是 | 否 | 33.8 | 0.141 | 0.543 |
| 03 | 是 | 否 | 36.6 | 0.330 | 0.445 |
| 04 | 否 | 否 | — | — | — |
| 05 | 是 | 否 | 34.2 | 0.256 | 0.387 |
| 06 | 是 | 否 | 32.6 | 0.180 | 0.530 |
| 07 | 是 | 否 | 33.6 | 0.170 | 0.525 |
| 08 | 否 | 否 | — | — | — |
| 09 | 否 | 否 | — | — | — |
| 10 | 是 | 否 | 45.3 | 0.217 | 0.434 |
| 11 | 否 | 否 | — | — | — |
| 12 | 否 | 否 | — | — | — |

以下为全部 12 个冲水文件的完整时序。每张图依次给出高度、水平/垂直形态、帧级判据、快速主路径占比和 300 帧补充路径占比。

### 冲水时序 01

![冲水时序 01](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_01_timeline.png)

### 冲水时序 02

![冲水时序 02](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_02_timeline.png)

### 冲水时序 03

![冲水时序 03](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_03_timeline.png)

### 冲水时序 04

![冲水时序 04](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_04_timeline.png)

### 冲水时序 05

![冲水时序 05](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_05_timeline.png)

### 冲水时序 06

![冲水时序 06](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_06_timeline.png)

### 冲水时序 07

![冲水时序 07](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_07_timeline.png)

### 冲水时序 08

![冲水时序 08](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_08_timeline.png)

### 冲水时序 09

![冲水时序 09](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_09_timeline.png)

### 冲水时序 10

![冲水时序 10](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_10_timeline.png)

### 冲水时序 11

![冲水时序 11](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_11_timeline.png)

### 冲水时序 12

![冲水时序 12](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/all_flush_timelines/flush_12_timeline.png)

## 8. 代表性跌倒时序选择

代表样本覆盖 3 名测试对象以及“直接躺/走后躺”两种动作：

| 编号 | 对象 | 动作 | 基线报警 / s | 优化后报警 / s | 人体形态占比 | 冲水簇占比 |
|---:|---|---|---:|---:|---:|---:|
| 01 | HT | 直接躺 | 26.6 | 26.6 | 0.872 | 0 |
| 02 | HT | 走后躺 | 32.5 | 32.5 | 0.791 | 0 |
| 03 | TF | 直接躺 | 23.2 | 23.2 | 0.819 | 0 |
| 04 | TF | 走后躺 | 26.0 | 26.0 | 0.774 | 0 |
| 05 | YQ | 直接躺 | 25.5 | 25.5 | 0.699 | 0 |
| 06 | YQ | 走后躺 | 27.8 | 27.8 | 0.729 | 0 |

这 6 个代表性跌倒在优化前后报警时刻完全一致，且报警窗口中的冲水簇占比均为 0，说明新增门控没有拦截这些已检出的真实躺地样本。

### 跌倒时序 01：HT 直接躺

![HT 直接躺](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/representative_fall_timelines/fall_01_HT_direct_timeline.png)

### 跌倒时序 02：HT 走后躺

![HT 走后躺](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/representative_fall_timelines/fall_02_HT_walk_timeline.png)

### 跌倒时序 03：TF 直接躺

![TF 直接躺](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/representative_fall_timelines/fall_03_TF_direct_timeline.png)

### 跌倒时序 04：TF 走后躺

![TF 走后躺](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/representative_fall_timelines/fall_04_TF_walk_timeline.png)

### 跌倒时序 05：YQ 直接躺

![YQ 直接躺](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/representative_fall_timelines/fall_05_YQ_direct_timeline.png)

### 跌倒时序 06：YQ 走后躺

![YQ 走后躺](https://raw.githubusercontent.com/sHORIZONz/tuchuang/imag/img/sy302-fall-flush-20260813/images/representative_fall_timelines/fall_06_YQ_walk_timeline.png)

## 9. 风险与后续验证

1. 半径 `[88,115] cm` 是基于当前马桶位置拟合的安装相关参数，不应直接视为跨场景常量。
2. 应增加马桶附近真实跌倒、不同马桶尺寸/水位、不同安装偏差和多人活动数据，重点验证冲水簇门控的边界。
3. 目前躺地召回率仅 56.25%，建议把“跌倒主流程召回率优化”作为独立任务，避免与本轮误报抑制混合调参。
4. 固件移植时应输出 `horizontal_rms`、`vertical_span`、`point_count`、`centroid_radius`、`shape_ratio`、`flush_ratio` 和最终上报路径，便于现场回归。

## 10. 附件数据

- [类别统计 CSV](data/class_summary.csv)
- [总体指标 CSV](data/metrics.csv)
- [脱敏时序清单 CSV](data/timeline_manifest_public.csv)
