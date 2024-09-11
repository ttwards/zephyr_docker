# dockerfile: ubuntu-dev.dockerfile
FROM ubuntu:24.04

# 设置SHELL为bash
SHELL ["/bin/bash", "-c"]

# 安装必要的软件包，包括sudo和udev
RUN apt update && apt -y install openssh-server wget udev tree sudo perl

RUN deluser --remove-all-files devcontainer || true


# RUN mkdir test && pwd && ls && echo "aaahaome"&& fuckdocker
# 如果devcontainer用户不存在，则创建devcontainer用户并设置密码
# RUN id -u devcontainer &>/dev/null || (useradd --create-home --no-log-init --shell /bin/bash devcontainer && echo 'devcontainer:123456' | chpasswd)

# # 将devcontainer用户添加到sudoers组
# RUN echo 'devcontainer ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

RUN useradd --create-home --no-log-init --shell /bin/bash devcontainer \
    && echo 'devcontainer:123456' | chpasswd \
    && adduser devcontainer dialout \
    && echo 'devcontainer ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers



# 切换到devcontainer用户
USER devcontainer:devcontainer
RUN sudo usermod -aG dialout devcontainer \
    && sudo usermod -aG plugdev devcontainer \
    && sudo gpasswd -a devcontainer dialout \
    && sudo gpasswd -a devcontainer plugdev

# 授予devcontainer用户/home/devcontainer权限
RUN sudo chmod 777 /home/devcontainer
RUN sudo chown -R devcontainer:devcontainer /home/devcontainer

# 创建临时文件夹供build使用，避开挂载文件夹以免被覆盖，并设为工作目录
RUN mkdir -p /home/devcontainer/workspace && \
    mkdir /home/devcontainer/tmp

WORKDIR /home/devcontainer/workspace

# 下载和运行Kitware存档脚本
RUN cd /home/devcontainer/workspace && \
    wget https://apt.kitware.com/kitware-archive.sh && \
    sudo bash kitware-archive.sh

# 安装所需的软件包
RUN export DEBIAN_FRONTEND=noninteractive && \
    export TZ=Asia/Shanghai && \
    sudo apt install -y --no-install-recommends \
    git cmake ninja-build gperf ccache dfu-util device-tree-compiler \
    wget python3-dev python3-pip python3-setuptools python3-tk python3-wheel \
    xz-utils file make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1 python3-venv

# 清理sudo apt缓存
RUN sudo apt clean

RUN sudo mv /usr/lib/python3.12/EXTERNALLY-MANAGED /usr/lib/python3.12/EXTERNALLY-MANAGED.old

# 初始化west并更新
RUN pip install --no-cache-dir west

# 将pip安装的可执行文件添加到PATH
ENV PATH="/home/devcontainer/.local/bin:${PATH}"

RUN pip show west && west init .
   
# 将west1.yml文件复制到west.yml
ADD west2.yml zephyr/west.yml
RUN west update

# 将west2.yml文件复制到west.yml
ADD west1.yml zephyr/west.yml
RUN west update

# 导出Zephyr环境
RUN cd zephyr \
    && west zephyr-export

# 安装Zephyr依赖
RUN pip install -r zephyr/scripts/requirements.txt

# 安装ESP32和Arduino支持
RUN cd zephyr && west blobs fetch hal_espressif
RUN cd zephyr && pyocd pack install r4fa4m1ab3cfm

# pip换源
RUN pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 安装Zephyr依赖
RUN pip install --no-cache-dir -q -r zephyr/scripts/requirements.txt

# 安装curl
RUN sudo apt install -y curl coreutils

# 安装Zephyr SDK
RUN wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.8/zephyr-sdk-0.16.8_linux-x86_64.tar.xz
RUN wget -O - https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.8/sha256.sum \
    | shasum --check --ignore-missing
RUN tar xvf zephyr-sdk-0.16.8_linux-x86_64.tar.xz
RUN mv zephyr-sdk-0.16.8 /home/devcontainer/zephyr-sdk
RUN yes y | bash /home/devcontainer/zephyr-sdk/setup.sh
RUN rm zephyr-sdk-0.16.8_linux-x86_64.tar.xz



# Clean up stale packages
RUN sudo apt-get clean -y && \
    sudo apt-get autoremove --purge -y

RUN mv -f /home/devcontainer/workspace/* /home/devcontainer/tmp
RUN mv -f /home/devcontainer/workspace/.[^.]* /home/devcontainer/tmp
RUN rm kitware-archive.sh || true
RUN sudo cp /home/devcontainer/zephyr-sdk/sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules \
    /etc/udev/rules.d

# 写入entrypoint.sh，若/home/devcontainer/tmp存在则将其中文件及文件夹移动到/home/devcontainer/workspace，之后删除/home/devcontainer/tmp
RUN echo '#!/bin/bash' > /home/devcontainer/entrypoint.sh \
    && echo 'if [ -d "/home/devcontainer/tmp" ]; then' >> /home/devcontainer/entrypoint.sh \
    && echo 'cp -r /home/devcontainer/tmp/* /home/devcontainer/workspace/' >> /home/devcontainer/entrypoint.sh \
    && echo 'cp -r /home/devcontainer/tmp/.[^.]* /home/devcontainer/workspace/' >> /home/devcontainer/entrypoint.sh \
    && echo 'rm -rf /home/devcontainer/tmp' >> /home/devcontainer/entrypoint.sh \
    && echo 'fi' >> /home/devcontainer/entrypoint.sh \
    && echo 'exec "$@"' >> /home/devcontainer/entrypoint.sh \
    && echo 'rm -rf /home/devcontainer/tmp' >> /home/devcontainer/entrypoint.sh
# 设置权限
RUN sudo chmod +x /home/devcontainer/entrypoint.sh

# RUN sudo apt install -y linux-tools-virtual hwdata usbutils \
#     kmod libusb-1.0-0-dev

# RUN sudo cp /home/devcontainer/zephyr-sdk/sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules \
#     /etc/udev/rules.d && \
#     sudo udevadm control --reload

# RUN chown -R devcontainer:devcontainer /home/devcontainer

# 设置入口点
ENTRYPOINT ["bash -c /home/devcontainer/entrypoint.sh"]

