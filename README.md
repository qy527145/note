# note
记录一些通用的笔记
```
# 设置国内镜像
bash <(curl -sSL https://linuxmirrors.cn/main.sh)
bash <(curl -sSL https://linuxmirrors.cn/docker.sh)

# cmd设置代理
set http_proxy=http://127.0.0.1:10808
set https_proxy=http://127.0.0.1:10808

# 安装uv
set UV_INSTALLER_GITHUB_BASE_URL=https://ghfast.top/https://github.com
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# 使用代理安装
powershell -ExecutionPolicy Bypass -c "$p='http://127.0.0.1:10808';[Net.WebRequest]::DefaultWebProxy = New-Object Net.WebProxy($p); irm https://astral.sh/uv/install.ps1 | iex"

# linux 下安装uv
export http_proxy=http://127.0.0.1:10808
export https_proxy=http://127.0.0.1:10808
curl -LsSf https://astral.sh/uv/install.sh | sh

# 使用镜像安装uv
# macOS / Linux
curl -LsSf https://gitee.com/wangnov/uv-custom/releases/download/0.10.10/uv-installer-custom.sh | sh
# Windows (PowerShell)
powershell -ExecutionPolicy Bypass -c "irm https://gitee.com/wangnov/uv-custom/releases/download/0.10.10/uv-installer-custom.ps1 | iex"

# Linux 使用镜像安装uv（自动查询版本号）
apt install jq curl
curl -LsSf https://gitee.com/wangnov/uv-custom/releases/download/`curl -s https://gitee.com/api/v5/repos/wangnov/uv-custom/tags | jq -r 'sort_by(.commit.date)|.[-1].name'`/uv-installer-custom.sh | sh


# 安装python
uv python install 3.13

# uv 设置全局镜像
# Linux： ~/.config/uv/uv.toml 或者 /etc/uv/uv.toml
mkdir -p /etc/uv
cat > /etc/uv/uv.toml << EOF
python-install-mirror = "https://ghfast.top/https://github.com/astral-sh/python-build-standalone/releases/download"

[[index]]
url = "https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple/"
default = true
EOF

# Windows：%ALLUSERSPROFILE%\uv\uv.toml （C:\ProgramData\uv\uv.toml）（在 %ProgramData%\uv\uv.toml 或者 %AppData%\uv\uv.toml）
python-install-mirror = "https://ghfast.top/https://github.com/astral-sh/python-build-standalone/releases/download"

[[index]]
url = "https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple/"
default = true

# 项目级别 pyproject.toml
[[tool.uv.index]]
url = "https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple"
default = true

# 备用方法：环境变量
UV_DEFAULT_INDEX=https://pypi.tuna.tsinghua.edu.cn/simple

# pip 设置全局镜像
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple

# uv安装poetry 全局tool
uv tool install poetry

# 设置代理
set http_proxy=http://127.0.0.1:10808
set https_proxy=http://127.0.0.1:10808

# 安装git
winget install git

# 设置git全局代理
git config --global http.proxy http://127.0.0.1:10808

# git config --global --unset http.proxy
# 为某个host单独设置代理
git config --global http.https://172.16.0.120.proxy http://127.0.0.1:10808


# 设置基础git全局配置
git config --global user.name xxx
git config --global user.email xxx
git config --global http.sslVerify false
git config --global credential.helper store

# ~/.gitconfig
[user]
	name = xxx
	email = xxx
[credential]
	provider = generic
	helper = store

[http "https://github.com"]
	proxy = http://127.0.0.1:10808

[http "https://192.168.xx.xx"]
	sslVerify = false
	proxy = http://127.0.0.1:10819

# ssh配置文件：
StrictHostKeyChecking no
ServerAliveInterval 60
ServerAliveCountMax 5

Host github.com
    HostName github.com
    User git
    ProxyCommand connect.exe -S 127.0.0.1:10808 %h %p

# msys2
# node
pacman -S mingw-w64-x86_64-nodejs

# gcc
pacman -S mingw-w64-ucrt-x86_64-toolchain
# lcov
pacman -S lcov

# clangd
pacman -S mingw-w64-clang-x86_64-toolchain
# llvm-cov
pacman -S mingw-w64-x86_64-llvm

# cmake
pacman -S mingw-w64-ucrt-x86_64-cmake

%MSYS_HOME%\usr\bin
%MSYS_HOME%\ucrt64\bin
%MSYS_HOME%\mingw64\bin
%MSYS_HOME%\clang64\bin
确保在下面的值上方
%SystemRoot%\system32;

rd /s /q build
cmake -G "MinGW Makefiles" -B build
cmake --build build

build\GaugeModuleApp.exe
bash -c "lcov -d . -c -o coverage.info --rc lcov_branch_coverage=1"
bash -c "genhtml coverage.info -o report --rc lcov_branch_coverage=1"

筛选
lcov --extract coverage.info "*/module1.c" --output-file module1_coverage.info

# pip install 缺msvc编译器（不想安装VS那一坨）
# 首先安装PortableBuildTools
# https://github.com/Data-Oriented-House/PortableBuildTools
call D:\BuildTools\devcmd.bat
set DISTUTILS_USE_SDK=1
# 或者参考https://github.com/qy527145/msvclib
```
