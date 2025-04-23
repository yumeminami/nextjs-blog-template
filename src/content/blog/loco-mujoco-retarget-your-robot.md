---
title: "Loco Mujoco Retarget Your Robot"
date: "2025-04-23"
draft: false
keywords: ["robotics", "motion capture", "AI"]
summary: "介绍如何使用 Loco Mujoco 将动捕数据转换为机器人动作"
featured: true
---

## 项目介绍

这个项目实现了一个动捕数据到人形机器人 motion retargeting 的 pipeline，同时还提供了基于 JAX 的 AMP, GAIL, PPO 等算法, 支持imitation learning, reinforcement learning 等训练方式。

Project Link: https://github.com/robfiras/loco-mujoco

## motion retargeting pipeline

官方的 tutor 中并没有详细介绍 motion retargeting pipeline 的实现，这里主要介绍下其实现的细节, 以及如何使用这个 pipeline 来 retarget 动捕数据到您的机器人。

首先我们先介绍一下项目中用到的各种动捕数据集都会被转换成一个 `Trajectory`:

```bash
Trajectory的主要组成
---
TrajectoryInfo（轨迹静态信息）：
关节名称列表（joint_names）
轨迹频率（frequency）
身体部位名称（body_names）
站点名称（site_names）
简化的MuJoCo模型（model）
---
TrajectoryData（轨迹动态数据）：
关节位置（qpos）：每个时间步所有关节的位置状态
关节速度（qvel）：每个时间步所有关节的速度状态
base 位置（xpos）：物理引擎计算的身体部位在世界坐标系中的位置
base 旋转（xquat）：表示旋转的四元数
base 速度（cvel）：线速度和角速度信息
质心数据（subtree_com）：身体部位的质心信息
站点位置和旋转矩阵（site_xpos, site_xmat）
---
可选组件：
轨迹转换（transitions）：用于强化学习的状态-动作-奖励数据
观察容器（obs_container）：构建观察所需的信息
```

**retargeting pipeline** 的关键在于动捕的数据集 smpl 模型的 `Trajectory` 如何转换成**适配**人形机器人的 `Trajectory` 数据，或者如何将 A 机器人转换成 B 机器人的 `Trajectory` 数据。

接下来我们将详细介绍 `retargeting pipeline` 的实现细节。

主要由两个关键阶段组成：

- **机器人形状优化(Shape Fitting)** - 一次性优化，为机器人创建匹配的SMPL模型
- **动作拟合(Motion Fitting)** - 使用优化后的SMPL模型将动捕数据映射到机器人关节

### 机器人形状优化 (fit_smpl_shape)

项目中所使用的动捕数据集采用的模型都是 SMPL 模型。

1. 初始T型姿势准备：
   1. 设置机器人到标准T型姿势(to_t_pose), 大多数人形机器人默认 POSE 都是双手自然下垂, 因此需要 rotate 一下双肩的关节
   2. 为关键部位(如手、脚、头部)创建mocap标记点, 需要修改 `Mujoco` 的 `xml` 文件, 添加 `mimic_site` 标记点
2. SMPL模型优化：
   1. 优化SMPL的形状参数(shape_new)和整体比例(scale)
   2. 通过梯度下降使SMPL模型的关键点尽可能接近机器人的对应点
   3. 计算SMPL模型和机器人之间的偏移位置(smpl2robot_pos)和旋转矩阵(smpl2robot_rot_mat)
   4. 计算高度偏移(offset_z)和高度比例(height_scale)
3. 保存优化结果：
   1. 保存优化后的SMPL形状参数到shape_optimized.pkl，供后续动作拟合使用

### 动作拟合 (fit_smpl_motion)

使用优化后的SMPL模型将具体动捕数据转换为机器人动作：

1. 准备环境：
   1. 创建目标机器人环境，添加mocap跟踪点(mimic_site)
   2. 加载先前优化的SMPL形状参数(shape_optimized.pkl)
2. 运动数据处理：
    1. 处理原始SMPL动作数据(pose_aa, trans)
    2. 应用高度缩放和偏移，调整比例, 得到适配 shape_optimized.pkl 的 SMPL 动作数据
3. 姿势转换核心过程：
    1. 对每一帧动作数据：
        1. 通过SMPL模型计算全局位置和旋转
        2. 应用先前优化的变换矩阵(位置偏移和旋转矩阵)
        3. 将结果位置设置到机器人的mocap标记点(mimic_site)
        4. 通过MuJoCo的逆运动学(mj_step)计算机器人关节角度
        5. 可视化过程(可选)
4. 生成关节轨迹：
    1. 收集计算得到的关节角度(qpos)
    2. 计算关节速度(qvel)，通过有限差分法
    3. 构建完整轨迹数据结构(Trajectory)


### 机器人到机器人的重定向

LocoMuJoCo 还支持机器人之间的动作转换，通过 `motion_transfer_robot_to_robot` 函数实现：

1. 源机器人运动转换为SMPL动作
2. 使用优化算法拟合SMPL关节角度和位置
3. 将拟合后的SMPL动作应用到目标机器人

## retargeting your own robot

### ⚠️ 注意事项

   1. 请确保您的 amass dataset 已经下载完毕，并且按照 repo README 配置(https://github.com/robfiras/loco-mujoco/blob/master/loco_mujoco/smpl)
   2. 请确保您的 `python` 依赖已经按照 repo README 所需要安装完毕


### 准备您的机器人的 `Mujoco` 模型文件

   1. 将您的模型文件目录放置 `loco-mujoco/loco_mujoco/models` 目录下
   2. 修改您的 `mujoco` 的 `xml` 文件, 添加 `mimic_site` 标记点放在对应的 `<body>` 下

      ```xml
         <site name="pelvis_mimic" class="mimic" />
         <site name="upper_body_mimic" class="mimic" />
         <site name="left_hip_mimic" class="mimic"  />
         <site name="left_knee_mimic" class="mimic"  />
         <site name="left_foot_mimic" class="mimic"  />
         <site name="right_hip_mimic" class="mimic"  />
         <site name="right_knee_mimic" class="mimic"  />
         <site name="right_foot_mimic" class="mimic"  />
         <site name="left_shoulder_mimic" class="mimic"  />
         <site name="left_elbow_mimic" class="mimic"  />
         <site name="left_hand_mimic" class="mimic"  />
         <site name="right_shoulder_mimic" class="mimic"  />
         <site name="right_elbow_mimic" class="mimic"  />
         <site name="right_hand_mimic" class="mimic"  />
         <site name="head_mimic" class="mimic" />
      ```

   3. 您可以参考 `loco-mujoco/loco_mujoco/models/unitree_g1/g1_23dof.xml` 的写法, 添加您的机器人关节的 `mimic_site` 标记点

### **注册您的机器人** Mujoco Environment

   1. 在 `loco-mujoco/loco_mujoco/environments/humanoids` 文件中注册您的机器人 Mujoco Environment, 可以参考 `unitreeG1.py` 的写法

   2. 自定义禁用参数，在您的机器人 Environment 中的构造函数中添加参数，例如：

      ```python
      def __init__(self, disable_arms: bool = False, # custom disable arms
                  disable_back_joint: bool = False, # custom disable back joint
                  spec: Union[str, MjSpec] = None,
                  observation_spec: List[Observation] = None,
                  actuation_spec: List[str] = None,
                  **kwargs) -> None:
      ```

      其中 `disable_*` 参数用于是否禁用自定义的关节，在 `_get_spec_modifications` 函数中，remove 掉对应的 `joint` 以及 `actuator`。最后需要给对应的 `link` 添加一个固定的 `quat`。

      实现可以参考 `unitreeG1.py` 中的 `_get_spec_modifications` 函数以及 `_reorient_arms` 函数

   3. 在 `loco-mujoco/loco_mujoco/environments/humanoids/__init__.py` 文件中注册您的机器人 Environment

### 定义您的机器人 smpl robot conf

   1. 在 `loco-mujoco/loco_mujoco/smpl/robot_confs` 下添加一个您的机器人 Environment Class 同名的 yaml 文件, 例如 `unitreeG1.yaml`
   2. 对其 smpl 模型的 T Pose, 例如 `unitreeG1.yaml` 中的 `robot_pose_modifier` 参数(请根据您的机器人关节的实际情况修改 `robot_pose_modifier` 参数)

      ```yaml
         robot_pose_modifier:
         - left_shoulder_roll_joint: "np.pi/2"
         - left_elbow_joint: "np.pi/2"
         - right_shoulder_roll_joint: "-np.pi/2"
         - right_elbow_joint: "np.pi/2"
      ```

   3. 添加 env_params(请根据您的机器人关节的实际情况修改 `env_params` 参数)

      ```yaml
         env_params:
         disable_arms: False
         disable_back_joint: True
      ```

### 尝试 retargeting motion to your robot

   1. 修改 `loco-mujoco/examples/replay_datasets/smpl_example.py` 中 `ImitationFactory.make` 函数中的 `env_name` 参数为您的机器人 Environment Class 名称

   2. python `smpl_example.py` 运行
