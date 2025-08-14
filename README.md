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


# claude code
npm install -g @anthropic-ai/claude-code
修改配置文件 ~/.claude/settings.json：
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "key",
    "ANTHROPIC_BASE_URL": "xxx",
    "ANTHROPIC_MODEL": "claude-sonnet-4-20250514",
    "ANTHROPIC_SMALL_FAST_MODEL": "claude-3-5-haiku-20241022",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "32000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": 1
  },
  "permissions": {
    "allow": [],
    "deny": []
  }
}

```
