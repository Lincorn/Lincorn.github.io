# Ubuntu 22.04.5 离线安装海光 K100 AI（DCU）驱动与 DTK 全流程指南

> 适用于无法联网的服务器环境，驱动版本：`rock-6.3.13-V1.12.0.run`，DTK 版本：`DTK-25.04-Ubuntu22.04-x86_64.tar.gz`  
> 本文适配系统：**Ubuntu 22.04.5 LTS (Jammy)** 桌面版或服务器版均可。

---

## 🧩 一、安装 Ubuntu 时的建议选项

在安装 Ubuntu 22.04.5 桌面版时，建议如下设置：

| 项目 | 是否勾选 | 原因 |
|------|-----------|------|
| ✅ **最小化安装 (Minimal installation)** | ✔️ 建议勾选 | 安装最少的软件，系统更干净。 |
| ✅ **安装第三方图形和 Wi-Fi 硬件及额外媒体格式** | ✔️ 一定要勾选 | 会自动安装 `dkms`、`build-essential` 等编译驱动所需依赖。 |
| ❌ **在安装时下载更新** | 不勾选 | 离线服务器无法联网，否则会卡住。 |
| ⚠️ **Secure Boot（安全启动）** | 建议关闭 | 内核模块签名会阻止海光驱动加载。 |

---

## ⚙️ 二、系统依赖准备（离线）

如果服务器不能联网，请在一台联网的 Ubuntu 22.04 机器上执行：

```bash
sudo apt update
sudo apt download dkms build-essential libelf-dev linux-headers-$(uname -r) pciutils cmake automake make libdrm-dev python3
```

会生成一堆 .deb 文件。

将这些文件拷贝到服务器（例如 /home/ubuntu/offline-debs），然后执行：
```bash
sudo dpkg -i /home/ubuntu/offline-debs/*.deb
```

若有依赖错误可运行：
```bash
sudo apt -f install
```
（离线环境不会下载，只修复已存在的依赖关系）

## 🧱 三、安装海光 K100 AI 驱动（rock-6.3.13）

### 1️⃣ 添加执行权限
```bash
chmod +x rock-6.3.13-V1.12.0.run
```

### 2️⃣ 执行安装
```bash
sudo ./rock-6.3.13-V1.12.0.run --no-dkms
```
     --no-dkms 表示使用本地 DKMS 编译，不会联网拉取依赖。
     安装过程中会自动编译内核模块，请耐心等待。

### 3️⃣ 安装完成后重启
```bash
sudo reboot
```

### 4️⃣ 验证驱动状态
```bash
/opt/rocm/bin/rocminfo
/opt/rocm/bin/rocm-smi
```

### 若能看到类似以下输出：
```bash
Device 0: DCU K100 AI
gfx version: gfx928
Memory: 32768 MB
```
  说明驱动安装成功 🎉

## 🧠 四、安装 DCU ToolKit（DTK）
### 1️⃣ 解压 DTK 安装包
```bash
tar -xzvf DTK-25.04-Ubuntu22.04-x86_64.tar.gz -C /opt
```

### 2️⃣ 设置环境变量
```bash
sudo tee /etc/profile.d/dtk_env.sh <<'EOF'
export PATH=/opt/DTK/bin:$PATH
export LD_LIBRARY_PATH=/opt/DTK/lib:$LD_LIBRARY_PATH
export HSA_OVERRIDE_GFX_VERSION=9.2.8
export ROCR_VISIBLE_DEVICES=0
EOF
```
立即生效：
```bash
source /etc/profile.d/dtk_env.sh
```
### 3️⃣ 验证 DTK 是否安装成功
```bash
/opt/DTK/bin/dtk-smi
```

## 🧪 五、验证 K100 AI 是否工作正常
```bash
lspci | grep -i dcu
/opt/rocm/bin/rocm-smi
rocminfo | grep gfx
```
若输出中包含设备型号（如 gfx928 / gfx940 等），表示驱动与工具链均工作正常。

## ✅ 六、离线安装流程总结
| 步骤 | 说明 | 
|------|-----------|
| 💿 安装 Ubuntu 22.04.5| 勾选“最小化安装”与“第三方驱动” | 
| ⚙️ 准备依赖包 | 安装 build-essential、dkms、libelf-dev 等 |
| 📦 安装 rock 驱动 | sudo ./rock-6.3.13-V1.12.0.run --no-dkms | 
| 🔁 重启系统| sudo reboot | 
| 🧩 安装 DTK 工具包| 解压并配置环境变量 | 
| 🧪 验证驱动状态| rocminfo / rocm-smi 查看设备信息 | 

## 💡 七、附加建议
- 若需要运行深度学习框架（如 PaddlePaddle、PyTorch 等），请使用海光官方适配的 DCU 版本。
- 不要启用 Secure Boot，否则驱动模块无法加载。
- 若驱动编译失败，请确认 linux-headers 版本与当前内核完全一致。