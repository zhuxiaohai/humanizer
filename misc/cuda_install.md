

# 服务器配置记录

## 公司 8 卡 4090 服务器配置

### 20251119 更新

### 显卡驱动

1. 安装 NVIDIA 驱动 580 版本, 最高支持 CUDA13.0 (官网指引, 含内核模块及后续配置)

### Docker 配置

1. 安装基础依赖及阿里云镜像源
2. 安装 Docker、Docker Compose
3. 容器的 GPU 支持 :

### 显卡驱动

切换到 root 用户下执行

```
sudo apt-get remove --purge nvidia*
```

安装依赖

```
sudo apt-get update  
sudo apt-get install g++  
sudo apt-get install gcc  
sudo apt-get install make
```

确保 lsmod | grep nouveau 没有输出

按照 NVIDIA 官网更新最新驱动

<https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/ubuntu.html#ubuntu-installation-local>

- 使用 Distribution : ubuntu2204
- 使用 Network Repository Installation (amd64)
- 使用 Selecting a Branch or a Specific Driver version : 580
- 使用 Proprietary Kernel Modules
- 执行 Post-installation Actions

### Docker 配置

切换到普通用户下执行

```
# 系统依赖  
sudo apt update  
sudo apt upgrade  
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```

### # 阿里云配置

```
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository  ¥
"deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# 安装 docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
# 安装后处理
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
# 验证
docker --version
docker run hello-world
# 安装 docker compose
sudo apt-get install docker-compose -y
docker-compose --version
```

### 容器中使用 GPU

```
# Configure the repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg ¥
    && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | ¥
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | ¥
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list ¥
    && ¥
    sudo apt-get update

# Install the NVIDIA Container Toolkit packages
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker

# Configure the container runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify NVIDIA Container Toolkit
docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```