# 具身智能项目实机部署经验总结

## 概述

本文档总结了在具身智能机器人项目开发与实机部署过程中遇到的关键问题、解决方案及实用技巧。内容涵盖系统配置、网络设置、机器人通信、SLAM建图、模型部署与优化等多个方面，旨在为后续开发提供参考。

## 系统与网络配置

### IPv6 问题处理
在某些网络环境下，IPv6可能导致连接问题，可临时禁用：
```bash
# 禁用 IPv6
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1

# 启用 IPv6
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=0
```

### 高精度时间同步
多机协作需要精确时间同步，推荐使用 `chrony`。

> [!WARNING]
> 若使用 Tailscale 等 VPN 工具，如果你连续几天没有启动机器人，机器人可能会在本地持有错误的时间，导致 HTTPS 连接出现问题，Tailscale 无法登录成功，从而无法同步时间，形成死循环。因此建议在内网环境中进行时间同步。

**服务器端 (Arch Linux) 配置**：
1. 编辑 `/etc/chrony.conf`（内网以 10.0.0.0/8 为例）：
   ```
   pool 2.arch.pool.ntp.org iburst
   allow 10.0.0.0/8
   local stratum 10
   ```
2. 重启服务：`sudo systemctl restart chronyd`

**客户端 (Raspbian) 配置**：
1. 安装：`sudo apt install chrony`
2. 编辑 `/etc/chrony/chrony.conf`，指向服务器：
   ```
   server [服务器IP] iburst minpoll 0 maxpoll 5
   ```
3. 重启服务：`sudo systemctl restart chrony`

**验证命令**：
- `chronyc tracking`
- `chronyc sources -v`
- `sudo chronyc clients`

### 校园网认证
（哈工深专用）使用旧版认证界面 `basic2` 可避免 SSO 登录问题，适用于多设备登录：
```
https://net.hitsz.edu.cn/srun_portal_pc?ac_id=1&theme=basic2&wlanacname=&wlanuserip=10.249.1.14
```

## 机器人系统配置

### 机器人环境配置

请参考 [Yanshee-Deployment](https://github.com/Yanshee-Serenade/Yanshee-Deployment) 仓库。

### ROS 环境与连接
- 机器人运行 ROS Kinetic，经测试可直接与 Docker 容器中的 ROS Noetic 通信。
    - [Reddit 帖子](https://www.reddit.com/r/ROS/comments/t5ns08/possible_to_communicate_between_melodic_and_noetic/)中也反馈可以正常通信
- 机器人 ROS 环境初始化：`source /opt/yanshee/setup.bash`
- 切换 ROS 主节点：执行 `~/scripts/switch_master.sh remote`

### 机器人服务管理
```bash
# 这些是硬件相关服务，现在的 setup 程序应该能正确处理这两个服务的启动，但如果发现硬件出现问题可检查这两个服务运行是否正常。这两个服务异常可能导致 IMU 数据无法读取或音响异常
sudo systemctl start dm_deviceapp
sudo systemctl start dm_devmanagementapp

# 重启机器人全部 ROS 服务，若 ROS1 Master 重启，机器人的服务也必须重启
sudo systemctl stop ubtedu-ros
sudo systemctl start ubtedu-ros

# 查看日志，跟踪启动情况
journalctl -u ubtedu-ros
```

### 摄像机服务
- 硬件加速启动脚本（我们使用的）：`python /home/pi/scripts/hardware_compressor.py &`
- 启动内置视频流服务：
  ```bash
  curl -X POST http://localhost:9090/v1/visions/streams \
    -H "Content-Type: application/json" \
    -d '{"resolution":"640x480"}'
  ```
- 关闭内置视频流服务：
  ```bash
  curl -X DELETE http://localhost:9090/v1/visions/streams
  ```

### 相机内参
机器人相机参数（YAML 格式）：
```yaml
cam0:
  cam_overlaps: []
  camera_model: pinhole
  distortion_coeffs: [0.148509, -0.255395, 0.003505, 0.001639, 0.0]
  distortion_model: radtan
  intrinsics: [503.640273, 502.167721, 312.565456, 244.436855]
  resolution: [640, 480]
  rostopic: /cam0/image_raw
```

### 机器人开放服务
- 实时视频流：`http://[机器人IP]:8000/stream.mjpg`
- API 文档：`http://[机器人IP]:9090/v1/ui/#/servos`
- 调试接口：`http://[机器人IP]:2017`

## SLAM 与视觉处理

### ORB-SLAM3 部署
- 在 Docker 容器中运行 ORB-SLAM3 ROS 版本。
- 查看跟踪点云数量：`rostopic echo /orb_slam3/tracked_points/width`

### 坐标系说明
- **原始 IMU 坐标系**：左-后-上
- **欧拉角与陀螺仪**：面向轴正方向顺时针旋转为正
- **加速度计**：x-, y-, z- 轴反向（右-前-下）
- **ORB-SLAM3 坐标系**：右-下-前

### Camera-IMU 标定
1. 录制数据包：
   ```bash
   rosbag record -O calibration_data.bag /camera/image_raw /imu/data
   ```
2. 使用 Kalibr 工具标定（需提前准备各 yaml，参考 [Kalibr Wiki](https://github.com/ethz-asl/kalibr/wiki/installation)）：
   ```bash
   rosrun kalibr kalibr_calibrate_imu_camera \
     --bag calibration_data.bag \
     --cam camchain.yaml \
     --imu imu.yaml \
     --target aprilgrid.yaml
   ```

## 模型部署与优化

目前仓库中已有 DevContainer 可直接构建开发、测试和生产环境。

### Flash Attention 安装
- 仅支持 Ampere 架构及以上显卡（如 RTX 30/40 系列），RTX 2080（Turing 架构）无法使用。
- ABI 检查：
  ```bash
  # 检查 PyTorch ABI 兼容性
  python3 -c "import torch; print('ABI is TRUE' if torch._C._GLIBCXX_USE_CXX11_ABI else 'ABI is FALSE')"
  ```
- 下载地址：GitHub Releases 页面寻找匹配版本 whl

### 模型下载技巧
Huggingface 模型下载易中断，推荐以下方法：

1. **手动下载**：
   - 启动下载后立即中断，在 `.cache/huggingface/hub/blobs/` 找到 `.incomplete` 文件。
   - 从模型仓库网页获取文件直链，使用 `wget` 下载。
   - 重命名为对应的 SHA256 校验和文件名。

2. **Git LFS 方法**：
   ```bash
   GIT_LFS_SKIP_SMUDGE=1 git clone <模型仓库>
   cd <模型目录>
   git lfs pull
   ```

3. **监控流量**：使用 `sudo iftop` 监控下载过程，防止卡死。

### 模型推理加速
- **量化加载**：设置 `load_in_8bit=True` 可显著提升加载速度。
- **编译优化**：`torch.compile` 效果有限，可考虑使用 vLLM 部署推理服务器。
- **TensorRT 优化**：转换模型至 TensorRT 格式提升推理性能，部署时需参考 [NVIDIA 深度学习框架支持矩阵](https://docs.nvidia.com/deeplearning/frameworks/support-matrix/index.html)。

### Depth Anything V3 (DA3) 部署
DevContainer 已经搞定全部依赖。关键依赖是 `xformers==0.0.32.post2`，需与 PyTorch 2.8.0 匹配。
- 启动命令：
  ```bash
  ros2 launch depth_anything_3_ros2 depth_anything_3.launch.py \
    image_topic:=/camera/image_slow \
    camera_info_topic:=/camera/camera_info \
    model_name:=depth-anything/DA3-SMALL \
    device:=cuda
  ```

### YOLO-World ROS2 部署
```bash
ros2 launch yolo_world_ros2 yolo_world_ros2_launch.py \
  color_image:=/camera/image_slow \
  depth_image:=/depth_anything_3/depth \
  color_camerainfo:=/camera/camera_info

# 设置检测类别
ros2 service call /yolo_world/classes yolo_world_interfaces/srv/SetClasses \
  "{classes: [wheel, shoe, text, box, chair, phone, tablet, pencil, monitor], conf: 0.25}"
```

### VLM 服务器 (Serenade) 部署
```bash
# 启动视觉语言模型服务
ros2 launch serenade_ros2 vlm_server.launch.py \
  image_topic:=/yolo_world/annotated_image \
  model_name:=Qwen/Qwen3-VL-4B-Instruct \
  max_new_tokens:=256

# 启动对话与导航模块
ros2 launch serenade_ros2 chatbot.launch.py
ros2 launch serenade_ros2 walker.launch.py

# 发送指令测试
ros2 topic pub /question std_msgs/msg/String \
  "data: '请走向离你最近的显示器'" -1
```

## 系统工具与技巧

### 远程图形界面
- 启用 SSH X11 Forwarding：配置 `/etc/ssh/sshd_config`：
  ```
  X11Forwarding yes
  X11DisplayOffset 10
  X11UseLocalhost yes
  ```
- Arch Linux 安装权限包：`paru -S xorg-xauth`
- macOS 使用 XQuartz：`open -a XQuartz`，连接时加 `-Y` 选项。
- 虚拟显示器：
  ```bash
  Xvfb :1 -screen 0 1920x1080x24  # 虚拟
  Xnest :1 -geometry 640x500       # 轻量远程
  ```

### ROS 版本桥接
DevContainer 已经搞定 ROS1-Bridge. 使用 `ros-jazzy-ros1-bridge` 或 `ros-humble-ros1-bridge` 连接 ROS1 与 ROS2。
- 构建时使用 Host 网络：`docker build --network=host .`
- 复制编译好的桥接到目标容器。

### LVM 快照管理
```bash
# 创建快照
sudo lvcreate -l 100%FREE -s -n lvsnap /dev/vg0/lv0

# 删除快照（永久应用更改）
sudo lvremove vg0/lvsnap

# 恢复快照（谨慎操作！）
sudo lvconvert --merge vg0/lvsnap
```

### 关节末端位置测定
若缺乏仿真引擎，可使用在线 URDF 可视化工具（如 [RobotsFan Viewer](https://viewer.robotsfan.com)）辅助测定。
- 在关节处添加可视化圆柱体辅助定位。
- 通过调整 `origin` 参数确定末端执行器位置。
  ```
    <visual>
    <origin rpy="1.57 -0.0 0.0" xyz="0.039 0 -0.056"/>
    <geometry>
      <cylinder radius="0.005" length="0.2"/>
    </geometry>
    <material name="default_material">
      <color rgba="0.2 0.2 0.2 1.0"/>
    </material>
  </visual>
  ```

## 问题与解决方案

### 避障实现思路
1. **数据源**：Depth Anything V3 提供实时深度图。
2. **处理流程**：
   - 密集点云 → 网格化（如 5cm×5cm 分辨率）
   - 障碍物分类与标注
   - 路径规划（A*/TEB 算法）
   - 执行器控制
3. **备选方案**：直接基于点云分类障碍，动态规划局部路径。

### SLAM 里程计融合
**可用数据源**：
- ORB-SLAM3：30Hz 里程计（质量一般）
- IMU：实际姿态数据
- Depth Anything V3：10Hz 深度图
- 动作模拟里程计

**融合目标**：
1. 存储物体大致坐标（通过 DA3 + YOLO-World 跟踪）
2. 构建可靠的融合里程计
3. 提供避障与导航支持

### VLM 交互设计
- **工具集**：
  - 增加/移除跟踪物体
  - 设置跟踪目标（描述、编号、距离）
  - 人机交互（问候、问答）
  - 机器人动作控制（旋转、移动）
- **记忆系统**：
  - 保存每次交互的场景描述与事件
  - 定期压缩记忆以节省资源
- **状态反馈**：向 VLM 提供实时状态（位置、传感器数据、任务进度）