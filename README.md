[README.md](https://github.com/user-attachments/files/33866441/README.md)
# 五子棋 AI 教练 · 13×13

[![GitHub](https://img.shields.io/badge/GitHub-linxuan-181717?logo=github)](https://github.com/2184020029/linxuan)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

一个单文件、零外部依赖的 13×13 五子棋 AI 教练。棋盘由原生 Canvas 绘制，AI 搜索在 Web Worker 中运行；默认只分析、不代下，可手动开启“自动对手”。

整个应用只有一个 `index.html`，下载后双击即可使用，不需要 Node.js、服务器或网络。

**仓库地址：** <https://github.com/2184020029/linxuan>

## 快速开始

1. 下载本仓库的 [index.html](index.html)。
2. 双击文件，用浏览器打开。
3. 在棋盘上落子，右侧会显示推荐点、Top-15 热力图和复盘评分。

推荐使用最新版 Chrome / Edge。Safari 或受企业策略限制的环境如果无法创建 Blob Worker，会自动降级到主线程计算。

如果仓库启用了 GitHub Pages，根目录的 `index.html` 可以直接作为站点入口。

## 功能

- **AI 教练**：每手后台搜索，返回 Top-15 候选点、唯一推荐点、上一步损失分。
- **VCF / VCT 提示**：VCF 标记为“必杀点”，VCT 标记为“疑似胜势”，不把疑似胜势说成确定胜利。
- **自动对手**：开启后，另一色由 AI 采用当前推荐点自动落子；你执子的一方始终手动。
- **提示开关**：可关闭棋盘上的热力图和金色推荐圈，侧栏反馈和评分图仍保留。
- **落子评分曲线**：只统计你所执的一方，0–100 分，100 表示当前最佳落点，最低分用红点标出。
- **AI 思考深度**：浅 / 标准 / 深三档，分别对应不同的时间和节点预算。
- **悔棋 / 重开 / 执黑 / 执白**。
- **自适应高分屏**：Canvas 按 devicePixelRatio 缩放。
- **Worker 失败降级**：Blob Worker 创建失败时，用同一份 worker 代码在主线程同步计算，状态栏标记“降级模式”。

## 操作

| 操作 | 说明 |
| --- | --- |
| 点击棋盘 | 在空点落子；分析中也可以落子，旧搜索会被中断 |
| 执黑 / 执白 | 选择你的执子方；切换会重置棋盘 |
| 自动对手 | 开启后另一色自动落子；关闭后双方手动 |
| 提示 | 开关棋盘上的热力图和推荐圈 |
| AI 思考深度 | 浅 / 标准 / 深 |
| 悔棋 | 撤销最后一手，并重新分析当前局面 |
| 重开 | 清空棋盘，开始新对局 |

## 规则与模式

- 13×13 棋盘，**自由五子棋规则**，不实现禁手。
- 黑方先行；双方都手动落子时是研究/复盘模式。
- “执黑 / 执白”表示你的视角，不影响规则。
- 开启“自动对手”后，AI 只替另一色落子，不会替你落子。

## 评分与损失

- **损失分**：上一轮最佳点分数 − 上一手实际落点分数；越大说明越亏。
- **落子评分**：把损失分映射到 0–100，`100 × exp(-loss / 12000)`，并截断到 0–100。
  - 100：当前最佳落点
  - 80 左右：略有损失
  - 20 左右：明显下崩
  - 0：损失极大
- 评分曲线只统计你所执的一方；底部状态栏的“第 N 手”是全局总手数，曲线里的“第 N 手”是你自己的第 N 手。

## AI 引擎

- 棋盘：`Int8Array(169)`，索引 `y * 13 + x`。
- 横、竖、两条斜线预计算线段索引。
- Zobrist 哈希（黑/白各 169 个随机 32 位值 + 行棋方键）。
- 置换表：Map，包含 depth / score / flag / move，上限 300k，超出后清空。
- 点评估：`myScore + 0.9 * oppScore`，使用 5/6 长度模式窗口；模式分值：
  - 五连 10,000,000
  - 活四 500,000
  - 冲四 50,000
  - 活三 20,000
  - 眠三 1,500
  - 活二 800
  - 眠二 100
- 叶子评估：`myTotal - 0.95 * oppTotal`，通过 make/unmake 增量更新经过落点的四条线。
- 候选点：切比雪夫距离 ≤2 的空点；排序取前 12，强制应手插到第一位。
- TSS：VCF 深度 ≤14 / 节点 ≤20k；VCT 深度 ≤8 / 节点 ≤30k，并对根候选做对手 VCF/VCT 反向校验。
- 主搜索：Negamax + Alpha-Beta + 迭代加深，置换表 flag 正确区分 EXACT / LOWER / UPPER。
- 根节点：每个候选都用完整窗口 `[-INF, +INF]` 独立搜索，保证热力图分数不是 fail-low / fail-high 边界值。
- 时间与节点：每 1024 节点检查一次；通过内部中断信号退出，不用超时器硬终止搜索。

### 搜索预算

| UI 档位 | 时间预算 | 节点上限 |
| --- | --- | --- |
| 浅 | 300 ms | 80,000 |
| 标准 | 1,000 ms | 200,000 |
| 深 | 2,500 ms | 400,000 |

TSS 使用总节点预算的 30%。

### Worker 与中断

```text
主线程 → Worker: { type: 'analyze', requestId, board, side, timeBudgetMs, nodeBudget }
Worker → 主线程: { type: 'progress', requestId, depth }
Worker → 主线程: { type: 'result', requestId, best, top15, lossFromPlayer, meta }
```

- 所有结果都校验 `requestId`，过期结果直接丢弃。
- 分析中再次落子时，会中断旧 Worker 并重建；新 Worker 先用一次 300ms 快速分析补算上一局面，再搜索当前局面，避免快速落子时损失分变成 `null`。
- 降级模式是主线程同步计算，无法在计算中途打断。

## 项目结构

```text
index.html   # 完整应用（HTML + CSS + Worker + 主线程 JS）
README.md    # 本文件
LICENSE      # MIT License
```

没有构建步骤，也没有外部字体、图片、CDN 或 npm 依赖。

## 浏览器支持

- Chrome / Edge：支持 Blob Worker，正常模式。
- Safari 或禁用 Blob Worker 的环境：自动降级到主线程，功能可用但计算时会卡顿。
- 需要支持 Canvas、Web Worker、`devicePixelRatio`。

## 已知限制

- VCT 只表示“疑似胜势”。防守分支有宽度上限，不保证覆盖所有分支。
- 降级模式是同步计算，无法中途中断。
- 快速连续落子时，补算以最新一手为主；更早的中间手可能没有完整损失分。
- 自由规则，不实现三三、四四、长连等禁手判定。

## License

本项目使用 [MIT License](LICENSE)。
ploading README.md…]()
# linxuan
