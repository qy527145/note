# note
记录一些通用的笔记
```
# 安装uv
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# 安装python
uv python install 3.13

# uv 设置全局镜像
# 在 ~/.config/uv/uv.toml 或者 /etc/uv/uv.toml 填写下面的内容：
[[index]]
url = "https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple/"
default = true

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
git config --global https.proxy http://127.0.0.1:10808

# git config --global --unset https.proxy
# 为某个host单独设置代理
git config --global http.https://172.16.0.120.proxy http://127.0.0.1:10808


# 设置基础git全局配置
git config --global user.name xxx
git config --global user.email xxx
git config --global http.sslVerify false

# ssh配置文件：
ServerAliveInterval 60
ServerAliveCountMax 5

Host github.com
    HostName github.com
    User git
    ProxyCommand connect.exe -S 127.0.0.1:10808 &h %p
```
