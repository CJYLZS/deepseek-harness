# Agent Note: bwrap profile 中可配置的 GPU 设备透传

Status: implemented

[English](2026-08-15-bwrap-gpu-device-passthrough.md) | 中文

## 问题

bwrap profile 通过 `--dev /dev` 构建全新的 `/dev`，因此受限命令看不到宿主的任何设备节点。在 WSL2 上 GPU 是半虚拟化的 dxgkrnl 设备 `/dev/dxg`——CUDA/NVML 都通过它访问 GPU，且不存在 `/dev/nvidia*` 节点——所以依赖 GPU 的 ML 工作（`torch.cuda.is_available()`、`nvidia-smi`）在受限环境下失败，而 `dsh-sandbox-local` 此前没有任何旋钮可以把设备节点带进沙箱，除非通过 `runnerCommand` 整体替换 runner。

## 决策

`dsh-sandbox-local` 新增 `gpuDeviceNodes: string[]` 配置键（默认空）。提供方构建的每个 bwrap profile——平台 bwrap rung 与 `runnerCommand` 覆盖路径都一样——都会把列出的每个节点以 `--dev-bind <node> <node>` 绑定进全新的 `/dev`，但仅当该节点在宿主机上存在时。条目在插件加载时校验：非空绝对路径，重复项合并，任何违规都会立即失败。存在性守卫让同一份列表可以服务异构主机（裸机 `/dev/nvidia*` 对比 WSL2 `/dev/dxg`），非 GPU 主机完全不受影响。

默认空是有意为之：设备暴露是文件效果模式词汇之外的维度，因此默认组合的沙箱表面不发生变化；希望受限命令访问 GPU 的部署需要显式开启。Landlock 与 Seatbelt rung 不受影响——前者没有 mount 概念，后者没有对应机制。

## 测试

`local.spec.ts` 通过传入真实临时文件路径作为伪节点（profile builder 只做存在性检查），在任何主机上确定性固定该行为：默认无绑定、只绑定存在的节点、`runnerCommand` 与平台 rung 两条路径都收到配置的列表、重复项合并、空白或相对路径条目使插件加载失败。

## 备选方案

- **硬编码 `['/dev/dxg']` 列表**——否决：部署相关选择写成常量违反 no-hardcoded-tunables 约定，且裸机 NVIDIA 或 AMD ROCm 主机需要不同的节点集。
- **默认绑定 `/dev/dxg`**——否决：静默地用 GPU 访问拓宽每个默认组合的沙箱，而这是模式词汇从未承诺过的资源。
- **独立的 GPU 透传插件**——否决：需要在提供方上凭空发明一个 profile 扩展点，且没有消费方支撑；行为应属于拥有该功能的提供方。
- **整体绑定 `/dev`**——否决：远比单个设备宽泛；`--dev /dev` 的意义就在于空白设备命名空间。

## 后果

- WSL2 部署只需一行 `cordis.yml`（`gpuDeviceNodes: ['/dev/dxg']`）即可让受限命令使用 GPU；配置后两种沙箱模式都会暴露该节点，README 已记录。
- 未配置的部署与非 GPU 主机零变化，仅每次包装时对每个列出的节点多一次 `existsSync` stat。
- 配置错误在加载时立即失败，而存在性守卫让跨主机配置不会失效。
- 沙箱的文件效果承诺不变；设备访问是部署显式选择的独立维度。
