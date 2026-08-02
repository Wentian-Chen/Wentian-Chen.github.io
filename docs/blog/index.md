# 教程博客

<div id="km-home">

  <p class="km-lead">这里是我一路走来的技术笔记：环境配置的踩坑记录、ROS 与移动机器人的学习笔记、计算机视觉与深度学习的实践指南。所有内容都来自真实经历，希望它们能帮你在同一条路上少踩几个坑。</p>

  <!-- 环境配置 -->
  <section class="km-section">
    <h2 class="km-section-title">环境配置</h2>
    <p class="km-section-desc">配环境是最磨人的事，这里记录了我遇到过的各种问题与解决方案。</p>
    <div class="km-grid km-grid-2">
      <a class="km-card" href="envir_config/fix_some_common_linux_bugs.md">
        <h3 class="km-card-title">常见 Linux 小问题的解决方法</h3>
        <p class="km-card-text">蓝牙掉线、以太网失联、开机引导项、双系统时间不同步——四个高频问题的对症下药。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_apt.md">
        <h3 class="km-card-title">配置国内 APT 镜像源</h3>
        <p class="km-card-text">中科大、清华、阿里、华为四大源一键替换，覆盖 amd64 / arm64 与 Ubuntu 24.04 前后两个时代。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_pip.md">
        <h3 class="km-card-title">配置 Pip</h3>
        <p class="km-card-text">临时换源、永久换源（Linux / Windows）、代理配置，让 pip 下载不再龟速。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_nvidia_driver.md">
        <h3 class="km-card-title">配置 NVIDIA Driver</h3>
        <p class="km-card-text">从选择驱动版本到完整的问题排除流程，含 Secure Boot / MOK 签名注意事项。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_cuda.md">
        <h3 class="km-card-title">配置 CUDA</h3>
        <p class="km-card-text">安装前检查、deb 本地安装、路径配置与 update-alternatives 多版本管理。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_ssh.md">
        <h3 class="km-card-title">配置 SSH</h3>
        <p class="km-card-text">SSH 安装与密钥登录、rsync 断点续传、tmux 后台训练、代理与端口转发。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_docker.md">
        <h3 class="km-card-title">配置 Docker</h3>
        <p class="km-card-text">安装、免 sudo、镜像加速、X11 转发、NVIDIA Container Toolkit 与存储迁移。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="envir_config/config_thor.md">
        <h3 class="km-card-title">Jetson Thor 真机部署指南</h3>
        <p class="km-card-text">黑客松实战沉淀：在 Jetson AGX Thor 上部署 GR00T N1.5 并驱动 LeRobot 机械臂的完整流程。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
    </div>
  </section>

  <!-- ROS -->
  <section class="km-section">
    <h2 class="km-section-title">ROS</h2>
    <p class="km-section-desc">从安装配置到核心概念，再到建图导航的完整链路。</p>
    <div class="km-grid km-grid-2">
      <a class="km-card" href="ros/config_ros.md">
        <h3 class="km-card-title">配置 ROS</h3>
        <p class="km-card-text">ROS1 Noetic 与 ROS2 Humble 的安装，以及五种解决 rosdep init / update 失败的方法。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="ros/dive_into_ros2.md">
        <h3 class="km-card-title">ROS2 的深入理解</h3>
        <p class="km-card-text">从节点、话题到动作与参数，配合代码示例把 ROS2 的通信机制和工程结构一次讲清楚。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="ros/slam_and_navigation.md">
        <h3 class="km-card-title">SLAM 和 Navigation</h3>
        <p class="km-card-text">围绕「我是谁、我在哪、我要去哪」梳理建图、定位与自主导航的完整知识链。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="ros/cmakelists_guide.md">
        <h3 class="km-card-title">CMakeLists 详解</h3>
        <p class="km-card-text">find_package 的查找原理、add_dependencies 与 target_link_libraries 的工程实践。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
    </div>
  </section>

  <!-- Computer Vision -->
  <section class="km-section">
    <h2 class="km-section-title">Computer Vision</h2>
    <div class="km-grid km-grid-2">
      <a class="km-card" href="cv/yolo_guide.md">
        <h3 class="km-card-title">YOLO 的全流程指南</h3>
        <p class="km-card-text">从环境安装、数据标注、训练评估到模型导出的端到端教程，覆盖最常见的实战问题。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
    </div>
  </section>

  <!-- 大模型 -->
  <section class="km-section">
    <h2 class="km-section-title">大模型</h2>
    <div class="km-grid km-grid-2">
      <a class="km-card" href="model/dive_into_pytorch.md">
        <h3 class="km-card-title">PyTorch 的深入理解</h3>
        <p class="km-card-text">从张量与自动求导，到模型构建、数据管道与完整的训练循环范式。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
    </div>
  </section>

  <!-- 仿真 -->
  <section class="km-section">
    <h2 class="km-section-title">仿真</h2>
    <p class="km-section-desc">具身智能实验离不开仿真环境，这两篇是相关工具的使用笔记。</p>
    <div class="km-grid km-grid-2">
      <a class="km-card" href="../sim/isaacsim.md">
        <h3 class="km-card-title">Isaac Sim 使用指南与实战教程</h3>
        <p class="km-card-text">从环境搭建、资产管理到运动生成与 ROS2 联合仿真，一份面向具身智能实验的全流程指南。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
      <a class="km-card" href="../sim/isaaclab.md">
        <h3 class="km-card-title">Isaac Lab</h3>
        <p class="km-card-text">基于 Isaac Sim 的强化学习框架，面向机器人学习任务的使用笔记。</p>
        <span class="km-card-link">阅读笔记 →</span>
      </a>
    </div>
  </section>

</div>
