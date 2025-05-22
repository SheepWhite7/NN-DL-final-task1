
# 使用自己拍摄的视频进行 Plenoxel 3D 重建

## 一、环境搭建

### （一）克隆 Plenoxel 代码并配置环境
```bash
# 克隆代码仓库
git clone https://github.com/sxyu/svox2.git
cd svox2

# 创建并激活 Conda 环境
conda env create -f environment.yml
conda activate plenoxel

# 安装库（含 CUDA 扩展）
pip install -e . --verbose
```

**注意**：若 CUDA 工具包版本 < 11，需先安装 CUB：  
```bash
conda install -c bottler nvidiacub
```


### （二）安装 COLMAP（用于相机位姿估计）
```bash
# 安装依赖项
sudo apt-get install -y \
    git cmake ninja-build build-essential \
    libboost-program-options-dev libboost-graph-dev libboost-system-dev \
    libeigen3-dev libflann-dev libfreeimage-dev libmetis-dev \
    libgoogle-glog-dev libgtest-dev libgmock-dev libsqlite3-dev \
    libglew-dev qtbase5-dev libqt5opengl5-dev libcgal-dev libceres-dev

# 安装 CUDA 工具包（若需 GPU 加速）
sudo apt-get install -y nvidia-cuda-toolkit nvidia-cuda-toolkit-gcc

# 克隆并编译 COLMAP
git clone https://github.com/colmap/colmap.git
cd colmap
mkdir build && cd build
cmake .. -GNinja  # 若需指定 CUDA 路径，取消注释下方三行
# cmake .. -GNinja \
# -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc \
# -DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda \-DCMAKE_CUDA_ARCHITECTURES=89
ninja
sudo ninja install

# 配置无 GUI 环境（服务器场景）
export QT_QPA_PLATFORM=offscreen
```


## 二、数据准备

### （一）拍摄与抽帧
1. **拍摄要求**：  
   - 围绕物体拍摄 **360 度环绕视频**，包含不同仰角视角。  
   - 确保物体静止、光照稳定，避免动态元素、高光反射和运动模糊。  
2. **视频抽帧**：  
   ```bash
   mkdir frames
   ffmpeg -i video.mp4 -vf fps=2 frames/frame_%03d.jpg  # 每秒抽取 2 帧（可调整 fps 值）
   ```


### （二）格式转换与检查
```bash
# 进入 Plenoxel 脚本目录
cd opt/scripts

# 使用 COLMAP 处理图像（禁用径向畸变校正）
bash proc_colmap.sh /path/to/frames --noradial

# 检查匹配结果（确保输出不为 0）
cd /path/to/frames
sqlite3 database.db "SELECT COUNT(*) FROM matches;"
```


## 三、模型训练

### （一）启动训练
```bash
# 在 opt 目录下执行（路径需根据实际调整）
bash ./launch.sh <实验名称> <GPU 编号> <数据目录> -c <配置文件>


**配置文件说明**：  
- `configs/syn.json`：适用于 NeRF-synthetic 合成数据  
- `configs/llff.json`：适用于前向视角真实数据  
- `configs/tnt.json`：适用于 Tanks and Temples 数据集  
- `configs/custom.json`：适用于自定义 360 度数据  


### （二）监控训练进度
1. **实时日志**：  
   ```bash
   tail -f ckpt/<实验名称>/log
   ```
2. **TensorBoard 可视化**：  
   ```bash
   cd opt/ckpt/<实验名称>
   tensorboard --logdir="." --load_fast=false --host=127.0.0.1 --port=8008
   ```
   访问 `http://localhost:8008` 查看训练曲线。


## 四、测试与渲染

### （一）测试推理
```bash
# 在 opt 目录下执行
python render_imgs.py <检查点路径> <数据目录> --no_imsave

# 示例
python render_imgs.py ckpt/test1/ckpt.npz /root/syyang/nn_dl/svox2/frames --no_imsave
```
**参数说明**：`--no_imsave` 用于跳过图像保存，加速测试。


### （二）渲染 360 度视角视频
```bash
python render_imgs_circle.py <检查点路径> <数据目录>

# 示例
python render_imgs_circle.py ckpt/test1/ckpt.npz /root/syyang/nn_dl/svox2/frames
```


## 五、注意事项
1. **动态物体**：不支持动态场景，拍摄时需确保物体静止。  
2. **光照与曝光**：保持光照稳定，锁定相机曝光参数。  
3. **图像质量**：避免高压缩率 JPEG 图像，建议使用 PNG 格式。  
4. **CUDA 版本**：推荐使用 CUDA 11+，旧版本需手动安装 CUB。  
5. **配置调整**：若重建效果不佳，可尝试切换 `custom.json` 与 `custom_alt.json`，或调整 TV 损失和稀疏性参数。

