# 使用自己的数据进行3D重建与神经场渲染流程 (Plenoxel & NeRF)

本项目包含利用[Plenoxel (svox2)](https://github.com/sxyu/svox2) 和[NeRF](https://github.com/bmild/nerf)进行3D重建与渲染的完整流程。  
推荐运行环境：Ubuntu 18.04/20.04，Python 3.8+，CUDA 11.x（可选GPU），并保证已安装Anaconda。

## 目录

- [1. 安装与环境配置](#1-安装与环境配置)
    - [1.1 安装 plenoxel（svox2）](#11-安装-plenoxel)
    - [1.2 安装 NeRF 及 LLFF 工具](#12-安装-nerf-及-llff-工具)
    - [1.3 安装 COLMAP](#13-安装-colmap)
- [2. 数据准备](#2-数据准备)
    - [2.1 拍摄视频 & 抽帧](#21-拍摄视频--抽帧)
    - [2.2 COLMAP 预处理和格式转换](#22-colmap-预处理和格式转换)
- [3. Plenoxel 训练与推理](#3-plenoxel-训练与推理)
- [4. NeRF 训练与推理](#4-nerf-训练与推理)

---

## 1. 安装与环境配置

### 1.1 安装 plenoxel (svox2)

```sh
git clone https://github.com/sxyu/svox2.git
cd svox2

conda env create -f environment.yml
conda activate plenoxel

pip install -e . --verbose
```

### 1.2 安装 NeRF 及 LLFF 工具

```sh
git clone https://github.com/Fyusion/LLFF.git

pip install scikit-image imageio
```

### 1.3 安装 COLMAP

```sh
sudo apt-get install \
    git cmake ninja-build build-essential \
    libboost-program-options-dev libboost-graph-dev libboost-system-dev \
    libeigen3-dev libflann-dev libfreeimage-dev libmetis-dev \
    libgoogle-glog-dev libgtest-dev libgmock-dev libsqlite3-dev \
    libglew-dev qtbase5-dev libqt5opengl5-dev libcgal-dev libceres-dev

sudo apt-get install -y nvidia-cuda-toolkit nvidia-cuda-toolkit-gcc  # 可选, 若有Nvidia GPU

git clone https://github.com/colmap/colmap.git
cd colmap
mkdir build && cd build
cmake .. -GNinja
# 若需指定CUDA版本，使用如下命令（按需更改路径）：
# cmake .. -GNinja -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda -DCMAKE_CUDA_ARCHITECTURES=89
ninja
sudo ninja install
```

**注意：**  
如果在无GUI服务器环境运行，需设置环境变量：

```sh
export QT_QPA_PLATFORM=offscreen
```

---

## 2. 数据准备

### 2.1 拍摄视频 & 抽帧

- 使用手机或相机对建模物体环绕360°拍摄视频。
- 用`ffmpeg`抽取图片帧，例如每秒抽2帧：

```sh
mkdir frames1
ffmpeg -i /root/syyang/nn_dl/svox2/video.mp4 -vf fps=2 frames1/frame_%03d.jpg
```

### 2.2 COLMAP 预处理和格式转换

1. 调用处理脚本进行COLMAP SfM处理：

    ```sh
    cd /root/syyang/nn_dl/svox2/opt/scripts
    bash proc_colmap.sh /root/syyang/nn_dl/svox2/frames1 --noradial
    ```

2. 检查图像配对情况，输出不能为0：

    ```sh
    cd /root/syyang/nn_dl/svox2/frames1
    sqlite3 database.db "SELECT COUNT(*) FROM matches;"
    ```

---

## 3. Plenoxel 训练与推理

### 3.1 训练

建议先检查并修改`configs/custom.json`内的参数。

```sh
bash ./launch.sh test2 0 /root/syyang/nn_dl/svox2/frames1 -c configs/custom.json
```

### 3.2 训练进度监控

实时查看日志：

```sh
tail -f ckpt/test2/log
```

### 3.3 启动 Tensorboard

```sh
cd /root/syyang/nn_dl/svox2/opt/ckpt/test2
tensorboard --logdir="." --load_fast=false --host=127.0.0.1 --port=8008
```

### 3.4 测试模型渲染

```sh
python render_imgs.py /root/syyang/nn_dl/svox2/opt/ckpt/test2/ckpt.npz /root/syyang/nn_dl/svox2/frames1 --no_imsave
```

### 3.5 渲染动画/环拍

```sh
python render_imgs_circle.py /root/syyang/nn_dl/svox2/opt/ckpt/test2/ckpt.npz /root/syyang/nn_dl/svox2/frames1
```

---

## 4. NeRF 训练与推理

### 4.1 生成 LLFF 所需的 `poses_bounds.npy`

```sh
python /root/syyang/nn_dl/LLFF/imgs2poses.py /root/syyang/nn_dl/nerf-pytorch/data/frames1
```

### 4.2 配置 NeRF

- 修改 `run_nerf.py` 里的数据路径指向你的数据。
- 新建配置文件 `configs/frames.txt`，内容参考 `configs/fern.txt` 等文件。

### 4.3 训练

```sh
python run_nerf.py --config configs/frames.txt
```

### 4.4 渲染/测试

```sh
python run_nerf.py --config configs/frames.txt --render_only
```

---

## 备注与常见问题

- 充分利用GPU加速。
- 服务器没有GUI时，务必设置`export QT_QPA_PLATFORM=offscreen`，防止COLMAP报错。
- 路径请根据自己的实际情况适配。

---


（对应参考项目[Plenoxel](https://github.com/sxyu/svox2), [NeRF](https://github.com/bmild/nerf), [LLFF](https://github.com/Fyusion/LLFF), [COLMAP](https://colmap.github.io/)。）
