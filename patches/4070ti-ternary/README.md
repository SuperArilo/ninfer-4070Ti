# 4070 Ti / 三元 delta 补丁包

生成时间：2026-09-22 19:05（CST）
生成者：柚子皮（受离洛昀委托）

## 0. 先说清一件事：4090 和 4070 Ti 不需要"架构补丁"

Ambolio 的仓库叫 `ninfer-4090-windows`，我们本地是 RTX 4070 Ti —— **两张卡都是 Ada / sm_89**：

| | RTX 4090 | RTX 4070 Ti |
|---|---|---|
| 架构 | Ada | Ada |
| compute capability | **8.9** | **8.9** |
| 静态 smem / SM | 48 KiB（opt-in 上限 100 KiB） | 同 |
| 编出来的二进制 | 同一份（`-DCMAKE_CUDA_ARCHITECTURES=89`） | 同一份 |

所以**不存在"4070 Ti 专用 patch"**：编译产物通用，唯一的差别是**显存（24 GB vs 12 GB）**，
而显存差异只体现在运行参数上（`--max-context / --kv-capacity / --kv-dtype / --host-state-slots /
--host-kv-mib`），那些在 `start.bat` 里，不在二进制里。

真正需要打补丁的是**我们这条三元（PQ2_0/PTQ1_0）线**，也就是本补丁包的内容。

## 1. 补丁内容

base（共同基准）＝ Ambolio `v1.0.8-windows` 的树：
`6eb70a0784b68c87efcbeb0b08bd2c3a95914492`
我们的线 ＝ `SuperArilo/ninfer-4070Ti@e7f6d81a6112682362797be0af8f878ddca57b43`（3 笔提交 / 48 个文件）

| 文件 | 内容 | 适用基准 |
|---|---|---|
| `0001-ternary-*.patch` | 三元端口主体（39 文件：`src/ops/linear/ternary/*` 新内核 + artifact/layout 注册 + CMake） | v1.0.8 |
| `0002-tools-*.patch` | Python 侧注册 Prism 三元格式（`PQ2_0_G128` / `PTQ1_0_G128`） | v1.0.8 |
| `0003-porter-*.patch` | 采纳 CraneBW 的更新三元优化 + MSVC `C3493` 修复 + CI `-k 0` | v1.0.8 |
| `ninfer-4070ti-ternary-delta-vs-ambolio-v1.0.8.patch` | 上面三笔合成一个 diff（48 文件，234 KB） | v1.0.8 ✅ 实测可应用 |
| `patch-01-clean-on-v1.1.3.patch` | 48 个文件里**能干净带到 v1.1.3 上**的那 18 个 | v1.1.3 ✅ 实测可应用 |
| `manual-port-list.txt` | 剩下 30 个需要人工移植的文件清单 | — |
| `delta.namestatus.txt` | 48 文件的 A/M/D 全表 | — |

## 2. 怎么用

### 打在自己的 v1.0.8 基线上（就是现在这条跑得好的线）
```bash
cd <repo>
git checkout 6eb70a07            # 或任何 v1.0.8 派生的树
git apply -3 ../ninfer-4070ti-ternary-delta-vs-ambolio-v1.0.8.patch
# 或用 format-patch 那三发（保留提交信息）：
git am ../000{1,2,3}-*.patch
```

### 想叠加到上游 v1.1.3（v3 artifact 线）
```bash
cd <repo@v1.1.3>
git apply -3 ../patch-01-clean-on-v1.1.3.patch     # 18 个文件，干净
# 剩下 30 个（manual-port-list.txt）必须人工移植，见第 3 节
```
`-3`（three-way）在有 base blob 时能自动三方合并；目标仓没有 base 提交时改用
`patch -p1 --merge < ...`（带 fuzz，冲突会落成 `.rej`）。

## 3. 移植到 v1.1.3 的坑（为什么不能一步合并）

`v1.0.8 → v1.1.3` 上游改了 **974 个文件**，是一次架构级重建：

- **artifact 架构 v2 → v3**（上游提交 `1072/1073/1074` 一带共 289+362+233 个文件）——
  我们的三元补丁正是挂在 v2 的 reader/binder 上，所以 `src/artifact/*` 全冲突；
- **`src/targets/qwen3_6_27b/**` 整层被挪走**，`src/targets/qwen3_6/impl/runtime/*`
  里的 `dflash_impl.h / program_impl.h / text_context_impl.h / text_prefill_impl.h` 在上游已删；
- 代价回收：`v1.1.x` **只读 v3 artifact**，我们的 `Ternary-Bonsai-2-27B.gguf` → PQ2_0 制品是 **v2**，
  用 v1.1.3 的二进制**起不来**。

结论：往 v1.1.3 走 = 三元链（转换 + 读取注册）要重做一遍，不是 `git merge` 的事。
短期该吃的那口肉是 `--kv-dtype rk4v4-e8`（sm_89 专用 4-bit KV，`layouts_impl.h:649` 明确
要求 CC 8.9），它**就在现有二进制里**，不需要升级。

## 4. 同步到新版上游的推荐流程（全部在 fork 内自洽）

```bash
# 0) 补丁就存在 fork 的 patches/4070ti-ternary 分支上（孤分支，不参与编译）
git fetch origin patches/4070ti-ternary

# 1) 从上游目标版本拉一条同步分支
git worktree add ../wt-sync -b sync/upstream-v1.1.3 refs/tags/v1.1.3-windows
cd ../wt-sync

# 2) 先贴 18 个能干净贴上的
git apply -3 <(git show origin/patches/4070ti-ternary:patches/4070ti-ternary/patch-01-clean-on-v1.1.3.patch)

# 3) 剩下 30 个（manual-port-list.txt）人工合并；或用 Git 三方能力逐笔吃
git am -3 <(git show origin/patches/4070ti-ternary:patches/4070ti-ternary/0001-ternary-*.patch)
#   冲突 → 就地改 → git add → git am --continue

# 4) 推分支 + 手动起 CI
git push -u origin HEAD
#   网页 Actions → Windows Build (Ada sm_89) → Run workflow → 选这条分支
```

原则：**能自动三方合并就自动，冲突就停下来人工合**，绝不 `-X theirs/ours` 一刀切 ——
三元线和 artifact v2→v3 的差别必须一个个看，切错了编出来的东西会静默降级。

## 5. 复现命令（校验用）

```bash
# 基准与线头
git rev-parse 6eb70a0784b68c87efcbeb0b08bd2c3a95914492   # v1.0.8 树
git rev-parse e7f6d81a6112682362797be0af8f878ddca57b43   # 我们的线

# 三份东西的差异分类
git diff --name-status <v1.0.8> <ours>                   # 48 文件
git diff --name-only    <v1.0.8> <v1.1.3-windows>        # 974 文件

# 三方合并预演（不需要共同祖先，显式给 base）
git merge-tree --write-tree --messages --merge-base <v1.0.8> <ours> <v1.1.3-windows>
# → 15 个内容冲突 + 9 个"上游已删/我们改了"，共 24 个需人工
```
