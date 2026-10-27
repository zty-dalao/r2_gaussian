# R2-Gaussian 安装说明（CUDA 12.6）

本文记录 Windows 和 Linux 下的安装流程。当前配置使用 PyTorch 2.6.0 + CUDA 12.6；不要把旧配置中的 `pytorch=1.12.1`、`torchvision=0.13.1` 或 `cudatoolkit=11.6` 与本方案混用。

## 一、准备工作

需要准备：

- NVIDIA 驱动和 CUDA Toolkit 12.6，并确认 `nvcc --version` 可用；
- Git，并确保仓库的 submodules 已拉取；
- Python 3.9；
- PyTorch 2.6.0 和 torchvision 0.21.0 的 CUDA 12.6 wheel；
- 两个本地 CUDA 扩展需要 C++ 编译器。

本项目的 CUDA 扩展源码已经在 `cuda_rasterizer/forward.cu` 和 `backward.cu` 中添加了 `M_PI` 定义。如果是重新克隆项目，请确认两个文件的 include 后面都有：

```cpp
#ifndef M_PI
#define M_PI 3.14159265358979323846
#endif
```

## 二、Windows 安装

### 1. 安装编译工具

安装 Visual Studio Build Tools 2022，并勾选：

- Desktop development with C++；
- MSVC v143 - VS 2022 C++ x64/x86 build tools；
- Windows 10/11 SDK。

### 2. 创建 Conda 环境

在 PowerShell 中执行：

```powershell
conda create -n r2_gaussian_cuda126 python=3.9 pip
conda activate r2_gaussian_cuda126
cd E:\workspace\r2_gaussian
```

### 3. 安装 PyTorch 和普通依赖

```powershell
python -m pip install torch==2.6.0 torchvision==0.21.0 --index-url https://download.pytorch.org/whl/cu126
```

```powershell
python -m pip install numpy==1.24.1 plyfile tqdm tensorboard tensorboardX pyyaml opencv-python==4.10.0.84 matplotlib pydicom tifffile imagecodecs pyvista SimpleITK scikit-image Cython==0.29.36 open3d==0.18.0
```

### 4. 配置 CUDA 和编译环境

建议从 **x64 Native Tools Command Prompt for VS 2022** 启动，然后激活 Conda 环境：

```cmd
conda activate r2_gaussian_cuda126
cd /d E:\workspace\r2_gaussian
```

如果使用 PowerShell，则设置：

```powershell
$env:CUDA_HOME="C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.6"
$env:CUDA_PATH=$env:CUDA_HOME
$env:Path="$env:CUDA_HOME\bin;$env:CONDA_PREFIX\Lib\site-packages\torch\lib;$env:Path"
$env:MAX_JOBS="1"
```

验证：

```powershell
nvcc --version
where cl
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
```

### 5. 编译本地 CUDA 扩展

```powershell
python -m pip install setuptools==59.5.0
python -m pip install --no-build-isolation --verbose -e .\r2_gaussian\submodules\simple-knn
python -m pip install --no-build-isolation --verbose -e .\r2_gaussian\submodules\xray-gaussian-rasterization-voxelization
```

两个 `pip install` 命令必须分别执行。`MAX_JOBS=1` 可以降低 Windows 并行编译时内存不足导致的失败概率。

## 三、Linux 安装

Linux 不需要 Visual Studio Build Tools，使用 GCC/G++ 编译器即可。需要安装与驱动匹配的 NVIDIA 驱动、CUDA Toolkit 12.6、`gcc`、`g++`、`make` 和 Git。

### 1. 创建环境并安装依赖

```bash
conda create -n r2_gaussian_cuda126 python=3.9 pip
conda activate r2_gaussian_cuda126
cd /path/to/r2_gaussian
python -m pip install torch==2.6.0 torchvision==0.21.0 --index-url https://download.pytorch.org/whl/cu126
python -m pip install numpy==1.24.1 plyfile tqdm tensorboard tensorboardX pyyaml opencv-python==4.10.0.84 matplotlib pydicom tifffile imagecodecs pyvista SimpleITK scikit-image Cython==0.29.36 open3d==0.18.0
```

### 2. 配置 CUDA 和 GCC

```bash
export CUDA_HOME=/usr/local/cuda-12.6
export PATH="$CUDA_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$CUDA_HOME/lib64:$LD_LIBRARY_PATH"
export MAX_JOBS=1
nvcc --version
g++ --version
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
```

如果 CUDA 安装在 `/usr/local/cuda`，将 `CUDA_HOME` 改为 `/usr/local/cuda`。如果是重新克隆项目，请按本文开头的说明在两个 `.cu` 文件中添加 `M_PI` 定义。

### 3. 编译本地 CUDA 扩展

```bash
python -m pip install setuptools==59.5.0
python -m pip install --no-build-isolation --verbose -e ./r2_gaussian/submodules/simple-knn
python -m pip install --no-build-isolation --verbose -e ./r2_gaussian/submodules/xray-gaussian-rasterization-voxelization
```

## 四、安装验证

Windows 和 Linux 都可以执行：

```bash
python -c "import torch; import simple_knn._C; import xray_gaussian_rasterization_voxelization._C; print('CUDA extensions OK')"
```

预期输出：

```text
CUDA extensions OK
```

要先导入 `torch`，再导入两个扩展；Windows 下这样可以确保 PyTorch 的 DLL 已经加载。

## 五、数据量和 submodules 大小

Google Drive 数据集约为：

| 数据目录 | 大小 |
|---|---:|
| `real_dataset` | 约 3.91 GB |
| `real_dataset_naf_format` | 约 1.77 GB |
| `synthetic_dataset` | 约 5.74 GB |
| `synthetic_dataset_naf_format` | 约 5.73 GB |
| **总计** | **约 17.15 GB** |

完整下载并解压建议预留 25～30 GB 磁盘空间。

当前工作区本地 `r2_gaussian/submodules` 文件夹大小约为 **138.5 MB（约 132 MiB）**，这个数字包含编译生成的 `build` 和 `*.egg-info` 文件：

- `simple-knn`：约 52.9 MB；
- `xray-gaussian-rasterization-voxelization`：约 85.6 MB。

如果只计算源码、排除 `build`、`*.egg-info` 和缓存文件，当前源码约为 25 MB（约 24 MiB）。

## 六、常见错误

- `Microsoft Visual C++ 14.0 or greater is required`：Windows 未安装或未加载 MSVC Build Tools；
- `M_PI is undefined`：在 `forward.cu` 和 `backward.cu` 中添加本文开头的宏定义；
- `ninja: build stopped`：查看日志中第一个 `error:`，不要只看最后的汇总错误；可先设置 `MAX_JOBS=1`；
- `ModuleNotFoundError: No module named 'torch'`：使用 `--no-build-isolation`，并确认当前 Python 属于目标 Conda 环境；
- `DLL load failed`：先执行 `import torch`，并将 CUDA 的 `bin` 和 `torch/lib` 加入 PATH（Linux 使用 `LD_LIBRARY_PATH`）。
