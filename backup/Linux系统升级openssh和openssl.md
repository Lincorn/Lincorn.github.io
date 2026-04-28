# 具体脚本

## 1、获取openssh-rpms软件包
项目地址：https://github.com/boypt/openssh-deb
项目地址：https://github.com/boypt/openssh-rpms

## 2、接着执行如下操作：
```bash
yum install perl perl-IPC-Cmd
yum groupinstall -y "Development Tools"
yum install -y imake rpm-build pam-devel krb5-devel zlib-devel libXt-devel libX11-devel gtk2-devel initscripts chkconfig perl-Time-Piece 
```
```bash
# For CentOS7 and above:
yum install -y systemd-devel
```
```bash
# For CentOS5 only:
yum install -y gcc44
```


## 3、下载并编译openssh
下载软件包，主要是openssl和openssh
```bash
openssl: https://openssl-library.org/source/
openssh: https://mirrors.aliyun.com/pub/OpenBSD/OpenSSH/portable/
```
```bash
[root@openeurelr1233 ~]# cd openssh-rpms-main
amzn1 amzn2023  docker   downloads el6 pullsrc.sh version.env
amzn2 compile.sh docker.README.md el5  el7 README.md
```
```bash
#下面此步骤会自动下载所需源码包，也可以手动下载，放到downloads目录下
[root@openeurelr1233 openssh-rpms-main]# ./pullsrc.sh
上面脚本默认下载openssl和openssh的软件版本，会读取version.env文件，可修改此文件，选择需要下载和编译
的其他版本。
```
```bash
#编译生成rpm
[root@openeurelr1233 openssh-rpms-main]# bash compile.sh
[root@openeurelr1233 openssh-rpms-main]# cd el7/RPMS/x86_64
[root@openrner x86_64]# ls -al
-rw-r--r--. 1 root root 6263529 7月 13 22:24 openssh-9.8p1-1.x86_64.rpm
-rw-r--r--. 1 root root 6413937 7月 13 22:24 openssh-clients-9.8p1-1.x86_64.rpm
-rw-r--r--. 1 root root 5412869 7月 13 22:24 openssh-debuginfo-9.8p1-1.x86_64.rpm
-rw-r--r--. 1 root root  837413 7月 13 22:24 openssh-debugsource-9.8p1-1.x86_64.rpm
-rw-r--r--. 1 root root 3407697 7月 13 22:24 openssh-server-9.8p1-1.x86_64.rpm
```
## 4、rpm方式安装openssh软件包
安装脚本如下：
```bash
#!/bin/bash
cp /etc/pam.d/sshd /etc/pam.d/sshd.before
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.before
rpm -Uvh *rpm
cd /etc/ssh
ssh-keygen -t rsa -b 2048 -f ssh_host_rsa_key
ssh-keygen -t ecdsa -b 256 -f ssh_host_ecdsa_key
ssh-keygen -t ed25519 -f ssh_host_ed25519_key
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config
sed -i 's/#UsePAM no/UsePAM yes/g' /etc/ssh/sshd_config
/bin/cp /etc/pam.d/sshd.before /etc/pam.d/sshd
sed -i '/#KexAlgorithms/a KexAlgorithms curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1,diffie-hellman-group1-sha1' /etc/ssh/sshd_config
sed -i '/#Ciphers/a Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-cbc,3des-cbc,aes192-cbc,aes256-cbc,blowfish-cbc,cast128-cbc' /etc/ssh/sshd_config
sed -i '/#MACs/a MACs hmac-md5,hmac-sha1,umac-64@openssh.com,hmac-ripemd160,hmac-sha2-256,hmac-sha2-512,hmac-ripemd160@openssh.com' /etc/ssh/sshd_config
echo "HostKeyAlgorithms +ssh-rsa" >> /etc/ssh/sshd_config
systemctl restart sshd
 ```
# 遇到问题解决方法
## 问题一
如果提示脚本没有安装rpm-build就换成这个脚本compile.sh
```bash
#!/usr/bin/env bash
# Bash3 Boilerplate. Copyright (c) 2014, kvz.io
# 适配：CentOS 5/6/7/8/9、银河麒麟V10、Anolis 7/8、UOS 20+
# 功能：自动识别发行版，补全编译依赖检测，支持手动指定版本

set -o errexit
set -o pipefail
set -o nounset
# set -o xtrace # 调试时取消注释

trap 'echo -e "\nAborted, error $? in command: $BASH_COMMAND"; trap ERR; exit 1' ERR

# 基础路径变量定义
__dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
__file="${__dir}/$(basename "${BASH_SOURCE[0]}")"
__base="$(basename ${__file} .sh)"
__root="$(cd "$(dirname "${__dir}")" && pwd)"

arg1="${1:-}"
rpmtopdir=
WITH_OPENSSL= # 0:无openssl 1:系统openssl 2:静态编译openssl

# ====================== 1. 核心依赖检测 ======================
CHECK_DEPENDENCIES() {
    echo -e "\n[INFO] ===== 检查编译依赖工具 ====="
    local required_deps=(
        "rpmbuild" "gcc" "make" "rpm" "ldd" "install" 
        "find" "cat" "cut" "rev" "grep" "tr" "pushd" "popd"
    )

    for dep in "${required_deps[@]}"; do
        if ! command -v "$dep" &> /dev/null; then
            echo "[ERROR] 缺少编译依赖：$dep"
            echo "[TIPS] 执行以下命令安装所有依赖："
            echo "       yum install -y rpm-build gcc make coreutils findutils grep sed"
            exit 1
        fi
    done
    echo "[INFO] 所有编译依赖检测通过！"
}

# ====================== 2. 智能发行版识别（核心适配） ======================
GUESS_DIST() {
    echo -e "\n[INFO] ===== 自动识别发行版 ====="
    # 优先从os-release识别（通用方式）
    if [[ -f /etc/os-release ]]; then
        source /etc/os-release
        # 银河麒麟V10
        if [[ $NAME == "Kylin Linux Advanced Server" && $VERSION_ID == "V10" ]]; then
            echo "[INFO] 识别到：银河麒麟V10 → 适配为el7"
            echo "el7" && return 0
        fi
        # 统信UOS 20+
        if [[ $NAME == *"UOS"* || $NAME == *"UnionTech"* ]]; then
            echo "[INFO] 识别到：统信UOS → 适配为el8"
            echo "el8" && return 0
        fi
        # 龙蜥Anolis
        if [[ $NAME == *"Anolis"* ]]; then
            [[ $VERSION_ID == "8" ]] && echo "[INFO] 识别到：龙蜥8 → 适配为el8" && echo "el8" && return 0
            [[ $VERSION_ID == "7" ]] && echo "[INFO] 识别到：龙蜥7 → 适配为el7" && echo "el7" && return 0
        fi
        # CentOS/RHEL
        if [[ $NAME == *"CentOS"* || $NAME == *"Red Hat"* ]]; then
            case $VERSION_ID in
                5) echo "[INFO] 识别到：CentOS/RHEL 5 → 适配为el5" && echo "el5" && return 0 ;;
                6) echo "[INFO] 识别到：CentOS/RHEL 6 → 适配为el6" && echo "el6" && return 0 ;;
                7) echo "[INFO] 识别到：CentOS/RHEL 7 → 适配为el7" && echo "el7" && return 0 ;;
                8|9) echo "[INFO] 识别到：CentOS/RHEL 8+/9 → 适配为el8" && echo "el8" && return 0 ;;
            esac
        fi
    fi

    # 备用方案：通过rpm/dist识别
    if type -p rpm > /dev/null;then
        local dist=$(rpm --eval '%{?dist}' | tr -d '.')
        [[ $dist == "el9" ]] && echo "[INFO] 识别到el9 → 降级适配为el8" && echo "el8" && return 0
        [[ $dist == "el8" || $dist == "an8" ]] && echo "[INFO] 识别到el8/an8 → 适配为el8" && echo "el8" && return 0
        [[ $dist == "el7" || $dist == "an7" ]] && echo "[INFO] 识别到el7/an7 → 适配为el7" && echo "el7" && return 0
        [[ $dist == "el6" ]] && echo "[INFO] 识别到el6 → 适配为el6" && echo "el6" && return 0
        [[ $dist == "el5" ]] && echo "[INFO] 识别到el5 → 适配为el5" && echo "el5" && return 0
    fi

    # 最终兜底：通过glibc版本识别
    local glibcver=$(ldd --version | head -n1 | grep -Eo '[0-9]+' | tr -d '\n')
    [[ $glibcver -eq 25 ]] && echo "[INFO] glibc 2.5 → 适配为el5" && echo "el5" && return 0
    [[ $glibcver -eq 212 ]] && echo "[INFO] glibc 2.12 → 适配为el6" && echo "el6" && return 0
    [[ $glibcver -eq 217 ]] && echo "[INFO] glibc 2.17 → 适配为el7" && echo "el7" && return 0
    [[ $glibcver -ge 228 ]] && echo "[INFO] glibc ≥2.28 → 适配为el8" && echo "el8" && return 0

    # 所有识别失败
    echo "[ERROR] 无法识别发行版！"
    echo "[INFO] 当前系统信息："
    [[ -f /etc/os-release ]] && cat /etc/os-release | grep -E "NAME|VERSION"
    [[ -f /etc/redhat-release ]] && cat /etc/redhat-release
    echo "[TIPS] 请手动指定版本：./compile.sh el7 (或el5/el6/el8)"
    echo "unknown" && return 1
}

# ====================== 3. 版本目录选择 ======================
TOPDIR_SELECT() {
    local DISTVER=$(GUESS_DIST)
    [[ $DISTVER == "unknown" ]] && exit 1

    case $DISTVER in
        el8)
            rpmtopdir=el7  # el8兼容el7编译目录
            WITH_OPENSSL=${WITH_OPENSSL:-1}
            ;;
        el7)
            rpmtopdir=el7
            WITH_OPENSSL=${WITH_OPENSSL:-2}
            ;;
        el6)
            rpmtopdir=el6
            WITH_OPENSSL=${WITH_OPENSSL:-2}
            ;;
        el5)
            rpmtopdir=el5
            WITH_OPENSSL=${WITH_OPENSSL:-2}
            rpm -q gcc44 2>&1 >/dev/null && export CC=gcc44
            ;;
    esac
    echo -e "\n[INFO] 最终选择编译目录：$rpmtopdir"
}

# ====================== 4. 源码文件检查 ======================
CHECKEXISTS() {
  if [[ ! -f $__dir/downloads/$1 ]];then
    echo -e "\n[ERROR] 缺失源码文件：$1"
    echo "[TIPS] 执行 ./pullsrc.sh 拉取源码，或手动放入 downloads 目录"
    exit 1
  fi
}

# ====================== 5. RPM编译核心逻辑 ======================
BUILD_RPM() {
    echo -e "\n[INFO] ===== 开始编译RPM包 ====="
    # 加载版本配置
    if [[ ! -f $__dir/version.env ]]; then
        echo "[ERROR] 缺少版本配置文件：version.env" && exit 1
    fi
    source $__dir/version.env
    [[ -f $__dir/version-local.env ]] && source $__dir/version-local.env

    # 检查必要源码
    local SOURCES=($OPENSSHSRC $OPENSSLSRC $ASKPASSSRC)
    [[ $rpmtopdir == "el5" ]] && SOURCES+=($PERLSRC)

    for fn in ${SOURCES[@]}; do
      CHECKEXISTS $fn
    done

    # 编译参数配置
    local RPMBUILDOPTS=(
        --define "with_openssl ${WITH_OPENSSL}"
        --define "opensslver ${OPENSSLVER}"
        --define "opensshver ${OPENSSHVER}"
        --define "opensshpkgrel ${PKGREL}"
        --define 'debug_package %{nil}'
        --define 'no_gtk2 1'
        --define 'skip_gnome_askpass 1'
        --define 'skip_x11_askpass 1'
    )

    # EL5/EL7 特殊参数
    [[ $rpmtopdir == "el5" ]] && RPMBUILDOPTS+=('--define' "perlver ${PERLVER}" '--define' 'dist .el5')
    [[ $rpmtopdir == "el7" && -z $(rpm --eval '%{?dist}') ]] && RPMBUILDOPTS+=('--define' "dist .$(rpm -q glibc | rev | cut -d. -f2 | rev)")

    # 执行编译
    pushd $__dir/$rpmtopdir > /dev/null
    RPMBUILDOPTS+=('--define' "_topdir $PWD")
    for fn in ${SOURCES[@]}; do
      install -v -m666 $__dir/downloads/$fn ./SOURCES/
    done
    rpmbuild -ba ./SPECS/openssh.spec "${RPMBUILDOPTS[@]}"
    popd > /dev/null

    # 输出编译结果
    local RPMDIR=$__dir/${rpmtopdir}/RPMS/$(rpm --eval '%{_arch}')
    echo -e "\n[SUCCESS] 编译完成！RPM包路径："
    find $RPMDIR -type f -name '*.rpm'
}

# ====================== 6. 辅助功能（查版本/查RPM路径） ======================
LIST_RPMDIR(){
    local RPMDIR=$__dir/${rpmtopdir}/RPMS/$(rpm --eval '%{_arch}')
    [[ -d $RPMDIR ]] && echo $RPMDIR
}

LIST_RPMS() {
    local RPMDIR=$(LIST_RPMDIR)
    [[ -d $RPMDIR ]] && find $RPMDIR -type f -name '*.rpm'
}

# ====================== 7. 主逻辑入口 ======================
# 第一步：检测依赖
CHECK_DEPENDENCIES

# 第二步：处理命令行参数
case $arg1 in
    # 辅助子命令
    GETEL)
        GUESS_DIST && exit 0
        ;;
    GETRPM)
        TOPDIR_SELECT && LIST_RPMS && exit 0
        ;;
    RPMDIR)
        TOPDIR_SELECT && LIST_RPMDIR && exit 0
        ;;
    # 手动指定版本编译
    el5|el6|el7|el8)
        rpmtopdir=$arg1
        echo "[INFO] 手动指定编译版本：$rpmtopdir"
        BUILD_RPM && exit 0
        ;;
    # 无参数：自动识别版本编译
    "")
        TOPDIR_SELECT
        if [[ ! -d $__dir/$rpmtopdir ]]; then 
            echo "[ERROR] 编译目录不存在：$__dir/$rpmtopdir"
            echo "[TIPS] 请确认目录存在，或手动指定版本：./compile.sh el7"
            exit 1
        fi
        BUILD_RPM && exit 0
        ;;
    # 无效参数
    *)
        echo -e "\n[ERROR] 无效参数：$arg1"
        echo "使用说明："
        echo "  自动编译：./compile.sh"
        echo "  手动指定版本：./compile.sh el7 (el5/el6/el8)"
        echo "  辅助命令：./compile.sh GETEL (查适配版本) | GETRPM (查RPM路径) | RPMDIR (查RPM目录)"
        exit 1
        ;;
esac
 ```
## 问题二
#### 如果按照1操作不行的话就到version.env把注释取消
```bash
# 编译选项：2=静态编译openssl（默认） WITH_OPENSSL=2
```
#### 验证
```bash
source version.env
echo $WITH_OPENSSL  # 应输出2
echo $OPENSSHVER    # 应输出10.2p1（自己需要安装的版本）
```
## 问题三
如果运行install.sh脚本有问题就使用下面脚本
```bash
#!/bin/bash
set -e  # 遇到错误立即退出，避免执行后续错误操作

# 定义颜色输出，方便查看执行状态
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # 重置颜色

# 第一步：备份关键配置文件
echo -e "${YELLOW}1. 备份SSH关键配置文件...${NC}"
cp /etc/pam.d/sshd /etc/pam.d/sshd.before 2>/dev/null || echo "sshd pam配置文件已存在备份或无需备份"
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.before 2>/dev/null || echo "sshd_config配置文件已存在备份或无需备份"

# 第二步：处理冲突的openssh-help包
echo -e "${YELLOW}2. 检查并卸载冲突的openssh-help包...${NC}"
CONFLICT_PKG=$(rpm -qa | grep openssh-help)
if [ -n "$CONFLICT_PKG" ]; then
    echo -e "${YELLOW}发现冲突包：$CONFLICT_PKG，正在卸载...${NC}"
    rpm -e --nodeps "$CONFLICT_PKG"
    if [ $? -eq 0 ]; then
        echo -e "${GREEN}冲突包卸载成功！${NC}"
    else
        echo -e "${RED}冲突包卸载失败，请手动执行：rpm -e --nodeps $CONFLICT_PKG${NC}"
        exit 1
    fi
else
    echo -e "${GREEN}未检测到openssh-help冲突包，跳过卸载步骤${NC}"
fi

# 第三步：安装OpenSSH相关rpm包
echo -e "${YELLOW}3. 安装OpenSSH rpm包...${NC}"
rpm -Uvh *.rpm --replacefiles  # --replacefiles 强制替换冲突文件，避免遗漏
if [ $? -eq 0 ]; then
    echo -e "${GREEN}OpenSSH包安装成功！${NC}"
else
    echo -e "${RED}OpenSSH包安装失败，请检查rpm包是否存在或完整${NC}"
    exit 1
fi

# 第四步：配置SSH权限和参数
echo -e "${YELLOW}4. 配置SSH权限和参数...${NC}"
cd /etc/ssh/ || exit  # 切换目录失败则退出
chmod 400 ssh_host_ecdsa_key ssh_host_ed25519_key ssh_host_rsa_key 2>/dev/null || echo "部分SSH密钥文件不存在，权限配置跳过"

# 修改sshd_config配置
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
sed -i -e 's/#UsePAM no/UsePAM yes/g' /etc/ssh/sshd_config 2>/dev/null || echo "UsePAM配置项无需修改"

# 恢复pam配置
/bin/cp /etc/pam.d/sshd.before /etc/pam.d/sshd 2>/dev/null || echo "pam配置文件恢复失败，请手动检查"

# 清理重复的加密算法配置
sed -i '/KexAlgorithms/d' /etc/ssh/sshd_config
sed -i '/GSSAPIKexAlgorithms/d' /etc/ssh/sshd_config
sed -i '/HostKeyAlgorithms/d' /etc/ssh/sshd_config

# 添加加密算法配置
echo "Kexalgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1,diffie-hellman-group1-sha1,diffie-hellman-group1-sha1,diffie-hellman-group1-sha1,diffie-hellman-group14-sha1,diffie-hellman-group-exchange-sha1,diffie-hellman-group-exchange-sha256,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group1-sha1,curve25519-sha256@libssh.org" >>/etc/ssh/sshd_config
echo "HostKeyAlgorithms +ssh-rsa" >>/etc/ssh/sshd_config

# 第五步：重启SSH服务
echo -e "${YELLOW}5. 重启SSH服务...${NC}"
systemctl restart sshd
if [ $? -eq 0 ]; then
    echo -e "${GREEN}SSH服务重启成功，脚本执行完成！${NC}"
else
    echo -e "${RED}SSH服务重启失败，请检查配置文件！${NC}"
    exit 1
fi
```

# 一键部署
## 支持版本
• Ubuntu 24.04/22.04/20.04
• Debian 13/trixie 12/bookworm 11/bullseye
• UnionTech OS 桌面版 20 家庭版 (Debian GLIBC 2.28.21-1+deepin-1)
• Kylin V10 SP1 (Ubuntu GLIBC 2.31-0kylin9.2k0.1)
```bash
sudo bash -c "$(curl -L https://github.com/boypt/openssh-deb/raw/master/lazy_install.sh)"
```
## 一键回滚
恢复发行版默认版本
```bash
sudo apt update
V=$(apt-cache madison ssh | awk 'NR==1 {print $3}')
sudo apt install --allow-downgrades -y \
    ssh=$V openssh-client=$V openssh-server=$V openssh-sftp-server=$V
```


