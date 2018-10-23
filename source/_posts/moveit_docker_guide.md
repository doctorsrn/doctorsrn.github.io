---
title: Moveit Docker的安装与配置过程
date: 2018-10-14 16:52:22
updated: 2018-10-18 23:44:22
layout: post
tags: [ros, moveit, docker]
comments: true
categories: 
- ros
---

# 使用Docker版本的moveit
最近打算学习使用一下moveit，在安装moveit时发现官网提供docker版本，之前对docker有一些浅显的了解，苦于没有应用场景，这次就尝试使用docker版的moveit练练手。在折腾了两天之后，完成支持gui的moveit的docker环境配置，主要内容包括两部分：
+ docker的安装和简单使用，以及nvidia-docker的安装
+ docker对moveit gui的支持
  
测试主机的环境是： 
+ 系统：Ubuntu16.04
+ 显卡：1060 6G

<!--more-->
# Docker的安装和简单使用，以及nvidia-docker的安装
## Docker的安装
按照[moveit官网](!http://moveit.ros.org/install/docker/)的教程安装：
官网的命令：
```
sudo apt-get install curl
curl -sSL https://get.docker.com/ | sh
sudo usermod -aG docker $(whoami)
```

将官网命令中的链接改为国内链接后使用：
```
sudo apt-get install curl
curl -sSL https://get.daocloud.io/docker | sh
sudo usermod -aG docker $USER
```
安装完成后记得log out然后log in使用户组权限配置生效，然后执行docker命令时就不需要敲`sudo`了。  
**常用docker命令:** 
另开shell进入已打开的容器：`docker exec -it docker_id /bin/bash`
将打开的容器作为镜像存储：`docker commit docker_id moveit/moveit_own:0.1`
容器和宿主机之间拷贝文件：`docker cp container:path  host`  和 `docker cp host container:path`
exit退出并关闭容器后再重启容器：`docker restart containerID`
利用Dockerfile新建镜像：` docker build -t moveit_test .`
参考网站：https://www.simapple.com/docker-tutorial

常用bash命令：https://github.com/zhangzju/myDockerBashrc/blob/master/.bashrc

## 拉取moveit镜像并运行一个支持图形界面的moveit容器
下面是[moveit官网](!http://moveit.ros.org/install/docker/)中使用gui的命令：
```
wget https://raw.githubusercontent.com/ros-planning/moveit/kinetic-devel/.docker/gui-docker gui-docker && chmod +x gui-docker

./gui-docker -it --rm moveit/moveit:kinetic-release /bin/bash
```
这两条命令是拉取moveit仓库的gui命令脚本`gui-docker`,并给执行权限，然后使用`gui-docker`脚本运行。
`gui-docker`的具体内容是：
```
#!/usr/bin/env bash

# This script is used to create and run a docker container with settings for gui interactions
# All arguments to this script will be appended to a docker run command.
# Example command line:
# ./gui-docker -ti --rm moveit/moveit /bin/bash

# XAUTH=/tmp/.docker.xauth
# xauth nlist :0 | sed -e 's/^..../ffff/' | xauth -f $XAUTH nmerge -
if [ ! -f /tmp/.docker.xauth ]
then
  export XAUTH=/tmp/.docker.xauth
  xauth nlist :0 | sed -e 's/^..../ffff/' | xauth -f $XAUTH nmerge -
fi

# Use lspci to check for the presence of an nvidia graphics card
has_nvidia=`lspci | grep -i nvidia | wc -l`
has_nvidia_docker=`which nvidia-docker > /dev/null 2>&1; echo $?`

# Check if nvidia-docker is avaialble
if [ ${has_nvidia_docker} -eq 0 ] && [ ${has_nvidia} -gt 0 ]
then
  DOCKER_COMMAND=nvidia-docker
  # Set docker gpu parameters
  # check if nvidia-modprobe is installed
  if ! which nvidia-modprobe > /dev/null
  then
    echo nvidia-docker-plugin requires nvidia-modprobe
    echo please install nvidia-modprobe
    exit -1
  fi
  # check if nvidia-docker-plugin is installed
  if curl -s http://localhost:3476/docker/cli > /dev/null
  then
    DOCKER_GPU_PARAMS=" $(curl -s http://localhost:3476/docker/cli)"
  else
    echo nvidia-docker-plugin not responding on http://localhost:3476/docker/cli
    echo please install nvidia-docker-plugin
    echo https://github.com/NVIDIA/nvidia-docker/wiki/Installation
    exit -1
  fi
else
  DOCKER_COMMAND=docker
  DOCKER_GPU_PARAMS=""
fi

DISPLAY="${DISPLAY:-:0}"
echo ${DOCKER_COMMAND}
echo ${DOCKER_GPU_PARAMS}
echo ${DISPLAY}
${DOCKER_COMMAND} run \
  --privileged \
  -e DISPLAY=unix$DISPLAY \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -v "/etc/localtime:/etc/localtime:ro" \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v "/tmp/.docker.xauth:/tmp/.docker.xauth" \
  ${DOCKER_GPU_PARAMS} \
  $@

```
**注意：** 这个脚本直接使用是无法支持gui的，我们还需要安装nvidia-docker和启用xhost。

这个脚本的实现过程是检查主机是否安装nvidia-docker，若未安装则直接使用不支持gui的原生docker命令运行；否则调用nvidia-docker执行命令，同时会检查是否安装nvidia-docker-plugin。  
需要注意的是nvidia-docker-plugin是nvidia-docker1专有的一个插件，用于支持gui，而nvidia-docker2版本中已经内置了nvidia-docker-plugin，所以不需要额外安装这个插件，**也就是说这个脚本是针对nvidia-docker1的**。  
我在使用nvidia-docker2时没有折腾成功moveit的gui，而使用nvidia-docker1成功支持moveit的gui。所以接下来介绍nvidia-docker1的安装。

## nvidia-docker1的安装
nvidia-docker1的安装方法参考[nvidia-docker官网wiki](!https://github.com/NVIDIA/nvidia-docker/wiki/Installation-(version-1.0))。 
安装完成后使用如下命令测试：
`docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi`

可能会遇到的问题：
+  `docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi`出现如下错误：
  >docker: Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:402: container init caused \"process_linux.go:385: running prestart hook 1 caused \\\"error running hook: exit status 1, stdout: , stderr: exec command: [/usr/bin/nvidia-container-cli --load-kmods configure --ldconfig=@/sbin/ldconfig.real --device=all --compute --utility --require=cuda>=10.0 brand=tesla,driver>=384,driver<385 --pid=8170 /var/lib/docker/overlay2/47e939b1bbc070d3ce4ec0920318a08c0cafb13d80d1716dd42251547bc6e64e/merged]\\\\nnvidia-container-cli: requirement error: unsatisfied condition: brand = tesla\\\\n\\\"\"": unknown.

**解决办法：** 出现这个错误的原因是pull下来的cuda版本和宿主机的cuda版本不一致导致的，宿主机使用的是cuda8.0，所以删除pull下来的镜像，使用下面指令重新pull：
`docker run --runtime=nvidia --rm nvidia/cuda:8.0-runtime nvidia-smi`即可以正常显示显卡的使用信息。

+ `docker run --runtime=nvidia --rm nvidia/cuda nvidia-smi`运行如果出错：
  >docker: Error response from daemon: Unknown runtime specified nvidiawhile running nvidia docker

**解决办法：** 上述错误是因为没有注册nvidia-docker-plugin的原因，
解决办法具体参考这个网站:https://github.com/nvidia/nvidia-container-runtime#docker-engine-setup
主要命令是：
```
  sudo mkdir -p /etc/systemd/system/docker.service.d
  sudo tee /etc/systemd/system/docker.service.d/override.conf <<EOF
  [Service]
  ExecStart=
  ExecStart=/usr/bin/dockerd --host=fd:// --add-runtime=nvidia=/usr/bin/nvidia-container-runtime
  EOF
  sudo systemctl daemon-reload
  sudo systemctl restart docker
```

至此nvidia-docker的安装和配置基本完成，使用mesa-utils包进行测试。
参考ROS官网[Docker GUI](!http://wiki.ros.org/docker/Tutorials/GUI)的方法使用`xhost +local:root`的方式运行moveit/moveit-release镜像：
```
#暴露显示端口给主机
xhost +local:root

#从moveit/moveit-release镜像创建一个容器
nvidia-docker run -it --rm\
  --env="DISPLAY" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  moveit/moveit-release bash
```
进入容器环境后，在容器中安装mesa-utils包：
```
sudo apt-get update
sudo apt-get install -y mesa-utils
```
然后在命令行运行：`glxgear`。正常的话可以看到转动的齿轮。这说明nvidia-docker工作正常，如果没有显示转动的齿轮，可以先跳过这个测试继续下面的配置。即使这个齿轮测试正常，想让moveit支持gui还没有完成配置，还需要对moveit的docker镜像添加配置信息。
可以尝试运行`roscore & rviz`（记得source ros的配置：`source /opt/ros/kinetic/setup.bash`）,这个时候肯定会报如下错误：
>libGL error: No matching fbConfigs or visuals found libGL error: failed to load driver: swrast Could not initialize OpenGL for RasterGLSurface, reverting to RasterSurface. libGL error: No matching fbConfigs or visuals found libGL error: failed to load driver: swrast Segmentation fault (core dumped)

**解决办法：** 见下一章节。


# Docker对moveit gui的支持
上一章节最后介绍了运行`roscore & rviz`（记得source ros的配置：`source /opt/ros/kinetic/setup.bash`）,这个时候肯定会报如下错误：
>libGL error: No matching fbConfigs or visuals found libGL error: failed to load driver: swrast Could not initialize OpenGL for RasterGLSurface, reverting to RasterSurface. libGL error: No matching fbConfigs or visuals found libGL error: failed to load driver: swrast Segmentation fault (core dumped)
这是因为当前使用的docker镜像没加添加nvidia-docker的相关支持信息，我们需要重新生成新的docker镜像。

插播一个知识点：之前有提到使用xhost支持gui显示，更多信息参考[ros docker gui](!http://wiki.ros.org/docker/Tutorials/GUI)的wiki，从这个wiki可以看到是docker支持gui主要有使用xhost和使用vnc等方式，我们就使用最简单的xhost方式实现，这种方式大概原理就是把宿主机的显示端口暴露给容器，然后显示gui。这种方法不仅适用于moveit镜像，也通用于ros系列的其他docker镜像。所以为了安全，记得退出容器时使用如下命令关闭暴露的端口：`xhost +local:root`

## 重新生成docker镜像
这一步主要为了解决这个错误：
>libGL error: No matching fbConfigs or visuals found libGL error: failed to load driver: swrast Could not initialize OpenGL for RasterGLSurface, reverting to RasterSurface. libGL error: No matching fbConfigs or visuals found libGL error: failed to load driver: swrast Segmentation fault (core dumped)

参考网站：
+ https://github.com/diegoferigo/dockerfiles/issues/6
+ https://github.com/NVIDIA/nvidia-docker/issues/136#issuecomment-232755805
  
具体步骤是：
+ **在moveit/moveit-release镜像的基础上重新生成镜像，加入nvidia-docker相关参数**，对应生成镜像的Dockerfile为：
  ```
  FROM moveit/moveit_own:0.2

  # install GLX-Gears
  RUN apt-get update && apt-get install -y \
      mesa-utils && \
      rm -rf /var/lib/apt/lists/*

  # nvidia-docker hooks
  LABEL com.nvidia.volumes.needed="nvidia_driver"
  ENV PATH /usr/local/nvidia/bin:${PATH}
  ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64:${LD_LIBRARY_PATH}
  ```
  将上述命令存为Dockerfile
+ biuld上面的Dockerfile：`docker build -t moveit_gui .`
  将会生成一个名为moveit_gui的docker镜像，可以通过`docker image ls`查看
+ 运行：`xhost +local:root`
+ 从之前生成的docker镜像生成一个容器：
  ```
  nvidia-docker run -it \
    --env="DISPLAY" \
    --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    moveit_gui bash
  ```
  比较方便的做法是将上面的命令写入一个bash文件，比如run.bash，然后给定执行权限：``chmod +x run.bash`，然后执行：`./run.bash`。

+ 在容器中测试gui是否正常：首先运行常用的测试：`glxgear`，正常的话会有一个转动的齿轮。然后运行`roscore & rviz`看能否正常启动。依然要考虑source ros配置脚本的问题。
  
## 解决Segmentation fault (core dumped)问题
如果上一步一切正常则表示当前的docker换进已经完全支持moveit的gui了。很大概率在运行rviz时会出现如下错误：
>[ INFO] [1539315753.889247768]: rviz version 1.12.16
[ INFO] [1539315753.889276892]: compiled against Qt version 5.5.1
[ INFO] [1539315753.889300776]: compiled against OGRE version 1.9.0 (Ghadamon)
[ INFO] [1539315754.465995908]: Stereo is NOT SUPPORTED
[ INFO] [1539315754.466066729]: OpenGl version: 4.5 (GLSL 4.5).
Segmentation fault (core dumped)

**解决办法：** 参考这两个issue可以解决这个问题：
https://github.com/ros-visualization/rviz/issues/975
http://ubuntuhandbook.org/index.php/2018/01/how-to-install-mesa-17-3-3-in-ubuntu-16-04-17-10/

解决的思路是更新MESA 3D graphics library，具体过程：
+ `sudo add-apt-repository ppa:ubuntu-x-swat/updates`
  这一步出错的话参考：https://www.aliyun.com/jiaocheng/137952.html
  ```
  sudo apt-get install python-software-properties
  sudo apt-get install software-properties-common
  ```
  然后再运行。
+ 接下来
  ```
  sudo apt-get update
  sudo apt-get dist-upgrade
  ```

如果一切正常的话，**现在就可以愉快的运行GUI了。**
所以Docker对moveit gui的支持总结来说，关键的步骤有两步：
+ 安装nvidia-docker1
+ 基于moveit/moveit-release镜像生成支持GUI的镜像

## 重写Dockerfile生成一个功能完善的moveit镜像
现在已经完成docker对moveit gui的支持的配置，我们将上面的一些配置过程和一些比较重要的环境配置整合至Dockerfile，可以基于官方moveit/moveit：release镜像生成一个功能完善的moveit镜像。主要的配置内容包括：
+ 修改apt更新源为国内的源
+ 安装常用的工具包vim、bash-completion、mesa-utils包等
+ 添加[moveit教程](!https://ros-planning.github.io/moveit_tutorials/doc/getting_started/getting_started.html)的示例代码并编译
+ 添加ROS更新环境变量指令
+ 支持命令行自动补全
+ 其他...

Dockerfile的书写：https://blog.csdn.net/wo18237095579/article/details/80540571

Dockerfile的具体内容如下：
```
FROM moveit/moveit:kinetic-release

# MAINTAINER
MAINTAINER sam.1223@live.cn

# running required command
# update source and install some useful packages
# bash-complete

RUN cd /etc/apt/ \
    # && mv sources.list sources.list.bkp \
    # && touch sources.list \
    # && echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse" >> sources.list \
    # && echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse" >> sources.list \
    # && echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse" >> sources.list \
    # && echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse" >> sources.list \
    # && echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse" >> sources.list \
    # && echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse" >> sources.list \
    # && echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse" >> sources.list \
    # && echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse" >> sources.list \
    # && echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse" >> sources.list \
    # && echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse" >> sources.list \
    && apt-get update  \
    && apt-get install -y -f apt-utils mesa-utils bash-completion tree vim  \
    python-software-properties software-properties-common \
    && /bin/bash -c "add-apt-repository -y ppa:ubuntu-x-swat/updates" \
    && apt-get update && /bin/bash -c "apt-get dist-upgrade -y" \
    && /bin/bash -c "source /etc/bash_completion" \
    # && echo "if [ -f /etc/bash_completion ] && ! shopt " >> ~/.bashrc \
    # && echo "-oq posix; then" >> ~/.bashrc \
    # && echo "    . /etc/bash_completion" >> ~/.bashrc \
    # && echo "fi" >> ~/.bashrc \
    # && source ~/.bashrc \
    && rm -rf /var/lib/apt/lists/*

# install moveit example code and build
# add setup.bash command to .bashrc
# reference: https://ros-planning.github.io/moveit_tutorials/doc/getting_started/getting_started.html

RUN /bin/bash -c "source /opt/ros/kinetic/setup.bash" \
    && echo "source /opt/ros/kinetic/setup.bash" >> ~/.bashrc \
    # && source ~/.bashrc \
    && cd ~/ws_moveit/src \
    && git clone https://github.com/ros-planning/moveit_tutorials.git \
    && git clone https://github.com/ros-planning/panda_moveit_config.git \
    && cd ~/ws_moveit/src \
    && apt-get update \
    && rosdep install -y --from-paths . --ignore-src --rosdistro kinetic \
    && cd ~/ws_moveit \
    && catkin config --extend /opt/ros/kinetic \
    && catkin build \
    && /bin/bash -c "source ~/ws_moveit/devel/setup.bash" \
    && echo 'source ~/ws_moveit/devel/setup.bash' >> ~/.bashrc \
    # && source ~/.bashrc \
    && rm -rf /var/lib/apt/lists/*

# RUN /bin/bash -c "apt-get update" \
#     && /bin/bash -c "apt-get install -y -f python-software-properties software-properties-common" \
#     && /bin/bash -c "add-apt-repository -y ppa:ubuntu-x-swat/updates" \
#     && /bin/bash -c "apt-get update && apt-get dist-upgrade -y" \

# nvidia-docker hooks
LABEL com.nvidia.volumes.needed="nvidia_driver"
ENV PATH /usr/local/nvidia/bin:${PATH}
ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64:${LD_LIBRARY_PATH}
```
以上就是Dockerfile的内容，很多内容注释掉了是因为测试一直出错，**上面的这个Dockerfile在本机build生成的镜像是可以正常使用**。如果还需要其他包或者配置，可以自行添加。
创建对应的Docker镜像： `docker build -t moveit/moveit_gui:release .`
或者直接pull我push到Docker Hub的镜像：`docker pull doctorsrn/moveit-gui`
对应的Docker Hub链接：https://hub.docker.com/r/doctorsrn/moveit-gui/
和github链接:https://github.com/doctorsrn/moveit-gui
之后使用生成的**moveit/moveit_gui:release**镜像创建容器和使用，接下来就可以愉快的按照[Moveit官网教程](!https://ros-planning.github.io/moveit_tutorials/doc/getting_started/getting_started.html)学习moveit的使用。比如载入rviz：`roslaunch panda_moveit_config demo.launch rviz_tutorial:=true`  
使用生成镜像创建容器：
```
xhost +local:root
nvidia-docker run -it \
  --env="DISPLAY" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  moveit_gui:release bash
```
再次建议：比较方便的做法是将上面的命令写入一个bash文件，比如run.bash，然后给定执行权限：`chmod +x run.bash`，然后执行：`./run.bash`。

# 待探索的内容和附录
这次主要使用了nvidia-docker1和xhost方式实现moveit的gui使用，还可以尝试以下内容：
+ nvidia-docker2
+ 基于VNC使moveit在docker容器中支持GUI：
  + https://github.com/fcwu/docker-ubuntu-vnc-desktop
  + http://wiki.ros.org/docker/Tutorials/GUI
  + https://github.com/bpinaya/robond-docker

其他的参考网站：
+ 基于nvidia-docker1的ros gui
  + https://github.com/turlucode/ros-docker-gui
  + http://wiki.ros.org/docker/Tutorials/GUI
  + https://docs.nvidia.com/ngc/ngc-titan-setup-guide/index.html
+ 有助于docker理解的资料：
  https://zhuanlan.zhihu.com/p/26418829?utm_medium=social&utm_source=weibo

这次配置docker moveit环境前后花了两天时间，中间遇到无数坑，google了无数问题，可能因为自己是Docker的初学者，对很多概念理解的不深，所以走了岔路。总之，通过这次环境的配置使自己对Docker的认识更深刻，Docker真的是一大神器。
