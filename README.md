# Gaussian_Splatting

## Colmap 安装与使用

Colmap 是用于图像序列结构重建的重要工具，通过特征匹配估算相机位姿及生成稀疏点云，是 NeRF 或 Gaussian Splatting 重建流程中必备组件。

### 1. 安装依赖（以 Ubuntu 为例）

```bash
sudo apt-get update
sudo apt-get install -y \
  git cmake ninja-build build-essential \
  libboost-program-options-dev libboost-filesystem-dev libboost-graph-dev libboost-system-dev \
  libeigen3-dev libflann-dev libfreeimage-dev libmetis-dev libgoogle-glog-dev \
  libgtest-dev libsqlite3-dev libglew-dev qtbase5-dev libqt5opengl5-dev \
  libcgal-dev libceres-dev
```

### 2. 编译安装 Colmap

```bash
git clone https://github.com/colmap/colmap.git
cd colmap
mkdir build && cd build
sudo cmake .. \
  -D CMAKE_CUDA_COMPILER="/usr/local/cuda-11.3/bin/nvcc" \
  -D CMAKE_CUDA_ARCHITECTURES='89'
sudo make -j$(nproc)
sudo make install
```

⚠️ 注意替换 CUDA 路径与架构（如 75/80/89 分别对应 RTX20/30/40 系列）。

### 3. 使用 Colmap

#### 3.1 图形界面重建

```bash
colmap gui
```

新建工程，将图片放入 images 文件夹。运行 “Automatic reconstruction”，完成后可查看 sparse / dense / mesh 等结果和相机位姿。

#### 3.2 Nerfstudio 命令行调用

使用深度特征（SuperPoint+SuperGlue）替代 SIFT：

```bash
ns-process-data images \
  --sfm-tool hloc \
  --feature-type superpoint \
  --matcher-type superglue \
  --data '/path/to/IMG' \
  --output-dir '/path/to/IMG'
```

#### 3.3 修复定位误差

使用 colmap gui 导入定位不准确的模型，删除异常图像，执行三角化与增量映射：

```bash
colmap point_triangulator --database_path … --image_path … --input_path … --output_path …
colmap mapper --database_path … --image_path … --input_path … --output_path … --Mapper.fix_existing_images 1
```

#### 3.4 地面方向校正

```bash
colmap model_orientation_aligner \
  --image_path … \
  --input_path sparse/0 \
  --output_path aligned_model
```

## 3D Gaussian Splatting 安装与使用

3D Gaussian Splatting（3DGS）是一种速度很快、效果精良的新兴三维重建技术。

### 1 环境安装

```bash
git clone https://github.com/graphdeco-inria/gaussian-splatting --recursive
cd gaussian-splatting
```
🔴**clone 以后，用本项目中的 train.py 文件替换 gaussian-splatting 中的 train.py 文件，以实现测试集划分。**

```bash
conda env create --file environment.yml
conda activate gaussian_splatting
```

### 2 使用流程

#### 视频拆帧

```bash
ffmpeg -i crane.mov -qscale:v 1 -vf fps=1 %04d.jpg
```

#### 准备 COLMAP 输出

将 COLMAP sparse 模型和校正后的图像放入项目目录，创建 distorted/ 与 input/，复制对应内容，跳过 matching 执行转换：

```bash
python convert.py -s data/crane --skip_matching
```

#### 训练模型

```bash
python train.py -s data/crane --data_device cpu --iterations 30000
```

可选使用 GPU，训练 30k 次约 30 分钟（视 GPU 而定）。

#### 可视化结果

```bash
./SIBR_viewers/install/bin/SIBR_gaussianViewer_app -m ./output/<model-ID>
```

支持 WSAD（移动）、IKJL（视角切换控制）。

### 常用命令速查

```bash
# 跳转已有环境
sudo mount machine3:/nfs/gs_env ~/anaconda3/envs/gs

# 转换场景
python convert.py -s data/indoor3 --skip_matching

# COLMAP 畸变校正
colmap image_undistorter --image_path … --input_path … --output_path …

# 重新训练
python train.py -s data/center --data_device cpu -r 1 \
  --iterations 100000 --densify_until 60000 --densification_interval 250 \
  --cache-path 'path'

# 实时渲染
./SIBR_viewers/install/bin/SIBR_gaussianViewer_app -m ./output/…

# 跳过训练直接渲染
python render.py -m ./output/… --skip_train

# 文件句柄错误处理（Linux）
ulimit -n 4096
```
