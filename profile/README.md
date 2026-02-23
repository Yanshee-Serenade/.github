## Yanshee::Serenade

具身智能全链路实战项目，旨在验证具身智能全链路在 Yanshee 机器人上的可行性。本科毕设项目。

> [!WARNING]
> 演示视频将稍后上传。若要取得视频，可通过邮箱 subcat2077@gmail.com 联系作者。

### 组成部分

- [机器人部署方式](https://github.com/Yanshee-Serenade/Yanshee-Deployment)
- [Unity 仿真环境](https://github.com/Yanshee-Serenade/Yanshee-RL-Unity)
- [ROS1 节点](https://github.com/Yanshee-Serenade/ROS1-Nodes)
  - [Unity 桥接器](https://github.com/Unity-Technologies/ROS-TCP-Endpoint)
  - [SLAM 节点](https://github.com/Yanshee-Serenade/ORB-SLAM3-ROS)
  - [消息转换节点](https://github.com/Yanshee-Serenade/Serenade-ROS)
    - 基于 TCP 提供机器人角度设置的接口
    - 发布相机内参
    - 解压缩机器人发送的硬件压缩后的图像流
    - IMU Axis Remapping & Frame Rotation
- [ROS2 节点](https://github.com/Yanshee-Serenade/ROS2-Nodes)
  - [逆运动学节点](https://github.com/Yanshee-Serenade/Yanshee-IK-ROS2)
  - [AI 深度图节点](https://github.com/Yanshee-Serenade/Depth-Anything-3-ROS2)
  - [物体打框节点](https://github.com/Yanshee-Serenade/YOLO-World-ROS2)
  - [主项目各节点](https://github.com/Yanshee-Serenade/Serenade-ROS2)
    - [机器人步态、旋转、动作](https://github.com/Yanshee-Serenade/Serenade-ROS2/tree/main/serenade_walker)
    - [语音交流模块](https://github.com/Yanshee-Serenade/Serenade-ROS2/tree/main/serenade_chatbot)
    - [智能体模块](https://github.com/Yanshee-Serenade/Serenade-ROS2/tree/main/serenade_agent)

### 相关文档

- [部署经验文档](EXPERIENCE.md)