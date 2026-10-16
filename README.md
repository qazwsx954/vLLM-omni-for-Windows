[README.md](https://github.com/user-attachments/files/33504317/README.md)
# vLLM-omni-for-Windows
可以在Windows上直接运行的vLLM-omni版本，无需经过WSL2和Docker，简化过程。本项目的GUI前端正在制作中，尚未完成，敬请期待。Bilibili哔站账号：战双单推人；也许名字怪怪的，不过用惯了也懒得换名字了。
# vLLM-Omni 0.30.0 for Windows

在 Windows 上运行 **vLLM 0.30.0** 与 **vLLM-Omni 0.30.0** 的完整移植：补齐平台缺失能力、修复
POSIX-only 调用、打通 4bit 量化与显存受限场景，并提供可复现的构建、安装与验证流程。

> 验证环境：Windows 11 / RTX 4070 Ti SUPER 16GB（Ada, SM 8.9）/ CUDA Toolkit 13.4 /
> torch 2.13.0+cu130 / Python 3.12.7 — vLLM 验证器 **PASS=28 / FAIL=0 / WARN=0**。

---

## 一、项目简介

上游 `SystemPanic/vllm-windows` 的 tag 止于 **v0.29.0**，官方未提供 v0.30.0 的 Windows 版本。
本项目以**三方合并**方式将该 fork 的 Windows 适配移植到上游 0.30.0：

```
base   = vllm-project/vllm        v0.29.0
ours   = vllm-project/vllm        v0.30.0
theirs = SystemPanic/vllm-windows v0.29.0  (= base + Windows 适配)
```

合并 74 个文件（+1216 / −432），手工解决 6 处冲突，并逐个补齐 0.30.0 新增代码中的 POSIX 依赖。
vLLM-Omni 为纯 Python 包（无 C/CUDA 扩展），适配集中在 POSIX-only 的 API 上。

### 已验证能力

| 能力 | 结果 |
|---|---|
| 文本生成 | `RESULT: PASS` |
| 语音合成（OmniVoice / MOSS-TTS-Nano / Audio8 二阶段） | 24–48 kHz 有效语音 |
| **7B AWQ 三阶段 omni**（thinker → talker → codec） | 单卡 16 GB 跑通，24 kHz / 16.32 s |
| **图像生成**（DreamLite 0.39B，diffusers 适配器） | 512×512 有效图像，约 30 s |
| **视频生成**（Wan2.1-T2V-1.3B **MXFP4 4bit**） | 832×480 / 33 帧 H.264，205 s |
| FlashInfer 运行期 JIT | `RESULT: PASS` |
| NCCL（本地编译） | `nccl.dll` 2.29.7，单卡 all-reduce `rc=0` |
| `parallel_stage_init=True` | 初始化耗时降低 42% |

### 明确不支持

`bitsandbytes(NF4/NF8)`（上游 vLLM 0.30.0 已移除）· `GGUF` / `AQLM`（上游未内置）·
`Flash Attention 3`（官方 Windows 禁用构建）· `RTX SPARK ARM64` · `CUDA Toolkit 13.0–13.2`（对齐冲突）。

> 多 GPU 的 TP/PP/DP **未实测** —— 本机为单卡；NCCL 本身已就绪。

---

## 二、环境要求

| 项 | 要求 |
|---|---|
| OS | Windows 10/11 x64 |
| GPU | NVIDIA，计算能力 ≥ 8.0（Ampere / Ada / Hopper / Blackwell）；驱动 ≥ 580 |
| **CUDA Toolkit** | **必须为 `12.8` 或 `>= 13.3`** |
| Python | 3.12.x |
| Visual Studio | 2019+（含“使用 C++ 的桌面开发”）—— **仅源码编译 vLLM 时需要** |

### CUDA Toolkit 版本兼容性（重要）

| 版本 | 支持 | 说明 |
|---|---|---|
| `<= 12.7` | ❌ | 不在兼容范围 |
| **`12.8`** | ✅ | 旧版本线中**唯一**受支持的版本 |
| `13.0 / 13.1 / 13.2` | ❌ | `cuda.h` 的 128 字节对齐与 MSVC 冲突，已明确放弃 |
| **`>= 13.3`** | ✅ | 13.3 起恢复 64 字节对齐，为推荐版本 |

> 本项目验证环境为 **CUDA Toolkit 13.4**。请务必避开 `13.0`–`13.2`。

---

## 三、目录结构

```
vllm-omni-0.30.0-windows/
├── vllm_omni/                  # vLLM-Omni 源码（纯 Python）
│   ├── config/                 # 配置与量化工厂（OmniINCConfig 等）
│   ├── diffusion/              # 扩散管线、加载器、offloader
│   ├── engine/                 # 编排器与 stage 运行时
│   ├── entrypoints/            # CLI / OpenAI 兼容服务
│   ├── model_executor/models/  # 模型实现
│   └── utils/
│       ├── posix_compat.py     # ★ Windows POSIX 兼容层（本项目新增）
│       └── file_lock.py        # fcntl.flock → msvcrt.locking
├── requirements/               # common / windows / cuda
├── examples/                   # 离线推理与在线服务示例
├── tests/
├── scripts/
│   ├── vllm-win-env.bat        # ★ 统一环境入口
│   ├── build/                  # 构建与安装
│   ├── test/                   # 测试与验证
│   └── tools/                  # 补丁、下载、审查
├── windows/
│   ├── docs/                   # 项目文档（构建、测试、移植、审核报告）
│   ├── scripts/                # 项目脚本副本
│   ├── configs/                # 部署配置（如单卡 7B AWQ）
│   └── patches/                # vllm-omni-windows-posix.patch
├── 使用说明.txt                 # 详细使用说明（安装 / 配置 / 启动 / FAQ）
└── README.md                   # 本文件
```

文档索引（`windows/docs/`）：

| 文件 | 内容 |
|---|---|
| `README.md` | 文档索引 |
| `平台适配与移植状态.md` | 支持 / 不支持、移植工作、遗留项 |
| `编译与测试报告.md` | 编译产物、测试结果、源码修复 |
| `构建指南.md` | 从零构建流程与已排查的问题 |
| `模型格式支持清单.md` | 模型格式 / 量化选型与实例审查 |
| `源码质量审核报告.md` | 源码审核结论与问题清单 |
| `vllm-omni-0.30.0-接口清单.md` | 接口与移植差异清单 |

---

## 四、安装

完整步骤与排错见 **`使用说明.txt`**。以下为最短路径。

### 4.1 预编译包（推荐，无需编译）

```bat
cd /d <包目录>
python -m venv .venv
.venv\Scripts\activate.bat
python -m pip install --upgrade pip setuptools wheel

python -m pip install torch==2.13.0+cu130 torchaudio==2.13.0+cu130 ^
    torchvision==0.28.0+cu130 --index-url https://download.pytorch.org/whl/cu130

python -m pip install --no-deps --ignore-installed ^
    wheels\vllm-0.30.0-cp312-cp312-win_amd64.whl

python -m pip install -r requirements\common.txt
python -m pip install -r requirements\windows.txt
python -m pip install -r requirements\cuda.txt
python -m pip install --no-deps --no-build-isolation -e .

python scripts\tools\apply_windows_patches.py
set PYTHONUTF8=1
python scripts\test\verify_vllm.py --expect-version 0.30.0
```

或使用一键脚本：

```bat
scripts\build\install_vllm_omni_windows.bat
scripts\build\install_vllm_omni_windows.bat --verify-only
```

> `--force-reinstall` 会连带卸载全部依赖，易触发本机的批量删除保护；请改用
> `--no-deps --ignore-installed`。若 `build/` 清理受阻，先执行 `move build build.bak`。

### 4.2 源码编译 vLLM（约 1–2 小时）

```bat
call "C:\Program Files\Microsoft Visual Studio\18\Community\VC\Auxiliary\Build\vcvarsall.bat" x64
set DISTUTILS_USE_SDK=1
set VLLM_TARGET_DEVICE=cuda
set MAX_JOBS=4
set TORCH_CUDA_ARCH_LIST=8.9
set PYTHONUTF8=1
python -m pip install --no-build-isolation -vvv .
```

也可使用一键流水线：`Build-vLLM-Windows.bat` 或 `build_vllm_windows.ps1 -Mode full`。

---

## 五、运行

先加载统一环境：

```bat
call scripts\vllm-win-env.bat
```

### 5.1 离线推理

```bat
python scripts\test\smoke_generate.py                    :: 文本
scripts\test\run_omni_e2e.bat "Hello, Omni on Windows."  :: 语音合成
scripts\test\run_qwen25_omni_7b_awq.bat                  :: 7B 三阶段 omni
scripts\test\run_dreamlite_t2i.bat                       :: 图像生成
scripts\test\run_wan21_mxfp4_t2v.bat                     :: 视频生成（4bit）
```

> ⚠️ **Wan2.1 MXFP4 不可使用 `--enable-cpu-offload`**：离线量化 checkpoint 搭配 offload 会在 CPU 上
> 加载权重，而 MXFP4 Marlin 的 `gptq_marlin_repack` 为 CUDA-only。

### 5.2 本地服务（localhost）

启动 OpenAI 兼容服务：

```bat
python -m vllm_omni.entrypoints.openai.api_server --model <模型> --host 127.0.0.1 --port 8000
```

| 项 | 默认 | 说明 |
|---|---|---|
| 监听地址 | `127.0.0.1` | 仅本机（localhost）可访问 |
| 端口 | `8000` | 被占用时用 `--port` 指定其它端口 |
| 健康检查 | `http://127.0.0.1:8000/health` | |
| 模型列表 | `http://127.0.0.1:8000/v1/models` | |
| 对话补全 | `http://127.0.0.1:8000/v1/chat/completions` | OpenAI 兼容 |

示例：

```bat
curl http://127.0.0.1:8000/health
curl http://127.0.0.1:8000/v1/models
```

**说明与注意事项**

- **开放局域网**：需显式指定 `--host 0.0.0.0`。这会暴露服务到内网，请自行评估风险。
- **Windows 下的进程间通信**：vLLM-Omni 多阶段链路在本地使用 **TCP loopback**，
  不使用 `ipc://`（Windows 无 Unix domain socket）。因此请为相关端口预留空间。
- **端口冲突**：若启动报 `Address already in use`，更换 `--port`；多阶段部署内部的
  默认协调端口（如 `29550`）被占用时同样需调整或释放。
- **防火墙**：仅本机访问无需放行；开放局域网时需在 Windows 防火墙中允许对应端口。

---

## 六、依赖

### 6.1 核心运行时

| 包 | 版本 | 说明 |
|---|---|---|
| Python | 3.12.x | |
| **CUDA Toolkit** | **`12.8` 或 `>= 13.3`** | **禁止 `13.0`–`13.2`** |
| torch | 2.13.0+cu130 | CUDA 13.0 通道 |
| torchaudio / torchvision | 2.13.0 / 0.28.0 (cu130) | |
| vLLM | 0.30.0（Windows wheel） | C/CUDA 扩展已编译 |
| vLLM-Omni | 0.30.0 / 0.30.1.dev2 | 纯 Python，editable 安装 |

其余按 `requirements/common.txt`、`windows.txt`、`cuda.txt` 安装（验证环境 29/29 满足，
`pip check` 无冲突）。

### 6.2 量化方法（选型参考）

`awq` / `auto_awq` / `gptq` / `gptq_marlin` / `awq_marlin` / `fp8` / `fbgemm_fp8` /
`compressed-tensors` / `modelopt`(NVFP4) / `mxfp4` / `torchao` / `humming` /
**`inc`（Intel Neural Compressor / AutoRound）**。

> `auto-round` 与 `auto_round` 均为 `inc` 的别名。AutoRound 导出的 MXFP4 checkpoint 声明的
> `quant_method` 为 `auto-round`，vLLM 会自动路由至 `inc`（已由 Wan2.1-T2V-1.3B 验证）。

不可用：`bitsandbytes/NF4`、`GGUF`、`OpenVINO int4`、`NVFP4`（需 Blackwell）。

### 6.3 已知非阻塞告警

- `xformers`：预编译版本面向 PyTorch 2.10/cu128/Py3.10，与本机 2.13/cu130/Py3.12 不匹配，
  扩展加载失败。vLLM 主链路不依赖，仅告警。
- `vllm._C` / `vllm._moe_C`（非 stable-ABI）加载失败，自动回退至 stable-ABI 版本，属预期行为。

---

## 七、许可证

本项目（Windows 移植部分）沿用上游许可证：**Apache License 2.0**（见 `LICENSE`）。

- 打包的模型权重遵循各自仓库的许可（如 Wan2.1 为 Apache-2.0；DreamLite 见其仓库声明）。
- 第三方代码保留其原始许可声明与源码头部的 SPDX 标注。

---

## 八、致谢

本项目建立在以下开源项目与作者的工作之上，谨致谢意。

### 上游与 Windows 移植

| 项目 | 作者 / 组织 | 用途 |
|---|---|---|
| **vLLM** | vLLM Project | 推理引擎本体 |
| **vLLM-Omni** | vLLM-Omni 贡献者 | 全模态多阶段推理框架（本移植对象） |
| **vllm-windows** | **SystemPanic** | v0.29.0 的 Windows 适配基线（三方合并的 `theirs`） |
| **nccl-windows** | **SystemPanic**（fork of NVIDIA/nccl） | Windows 版 NCCL |
| **flashinfer-windows** | **SystemPanic**（fork of flashinfer-ai） | Windows 版 FlashInfer |
| **humming-windows** | **SystemPanic** | 放宽 Triton 约束以消除依赖冲突 |
| **DeepGEMM-windows / DeepJIT-windows / fastsafetensors-windows** | **SystemPanic** | 配套 Windows 组件 |
| **NCCL** | NVIDIA | 多 GPU 集合通信库（Windows 支持基于 PR #1922） |
| **FlashInfer** | flashinfer-ai | 注意力与采样内核 |
| **FlashAttention** | Dao-AILab（Tri Dao 等） | FA2 内核 |
| **Marlin** | IST-DASLab / vLLM 集成 | 本项目 MXFP4 在 Ada 上的实际计算内核 |

### 框架与库

| 项目 | 作者 / 组织 | 用途 |
|---|---|---|
| PyTorch | Meta / PyTorch 团队 | 基础框架 |
| triton-windows | triton-lang / 社区 | Windows 版 Triton |
| Transformers / Diffusers / Safetensors / tokenizers | Hugging Face | 模型加载、扩散管线、权重容器 |
| **AutoRound / Intel Neural Compressor (INC)** | Intel | Wan2.1 的 MXFP4 4bit checkpoint 及其 `inc` 后端 |
| ONNX Runtime、NumPy、SciPy、regex、orjson、msgspec、pyzmq、tornado、soundfile、imageio、cache-dit、s3tokenizer 等 | 各自作者 | 依赖组件 |

### 模型（本项目实测使用）

| 模型 | 作者 / 组织 | 用途 |
|---|---|---|
| **Wan2.1-T2V-1.3B（-Diffusers）** | **Wan-AI（阿里巴巴通义万相）** | 文生视频；MXFP4 4bit 变体由 **INCModel**（Intel AutoRound）导出 |
| **DreamLite-base** | **carlofkl** | 0.39B 端侧文生图；验证通用 diffusers 适配器通路 |
| **Qwen2.5-Omni-7B-AWQ** | **Qwen 团队（阿里巴巴）** | 7B 三阶段 omni；验证 AWQ 4bit 与显存适配 |
| Qwen2.5-0.5B-Instruct | Qwen 团队 | 文本生成自检 |
| OmniVoice | **k2-fsa** | 单阶段 diffusion TTS |
| MOSS-TTS-Nano | **OpenMOSS-Team** | AR TTS |
| Audio8-TTS-Preview | **Audio8** | 二阶段链路与共享内存传输验证 |

### 平台、工具与镜像

| 项目 / 服务 | 作者 / 组织 | 用途 |
|---|---|---|
| Visual Studio / MSVC、Windows SDK | Microsoft | 编译工具链 |
| CMake、Ninja、Git、Rust（setuptools-rust） | 各自作者 | 构建工具 |
| **hf-mirror.com** | 社区镜像 | Hugging Face 下载镜像（本机直连 TLS 校验失败） |
| **ModelScope（魔搭）** | 阿里巴巴 | 模型检索与镜像源 |
| pytest、pytest-asyncio、pytest-mock、ruff、mypy | 各自作者 | 测试与静态检查 |

> 本移植**未修改任何对外接口的签名或语义**，所有改动均为平台适配（等价或降级）。
> Windows 上仍有无法完整复制的 POSIX 语义（属主校验、`O_NOFOLLOW`、只读标记），
> 已通过 `fchmod_is_effective()` / `ownership_is_verifiable()` 等探针显式暴露，而非静默降级。
