# macOS 基础开发环境搭建

## 1. 环境目标

一台新的 macOS 开发机建议先完成这些基础层：

| 层级 | 工具 | 作用 |
|------|------|------|
| 系统依赖 | Xcode Command Line Tools | 提供 `git`、`clang`、`make` 等基础编译工具 |
| 包管理 | Homebrew | 安装命令行工具、GUI App、服务和开发依赖 |
| Shell | Zsh / Oh My Zsh | 管理终端配置、主题、插件、补全 |
| Node 版本 | nvm | 管理多个 Node.js / npm 版本 |
| Ruby 版本 | rbenv | 管理多个 Ruby 版本 |
| Python 版本 | pyenv | 管理多个 Python 版本 |

安装顺序建议：

```text
Xcode Command Line Tools
        |
        v
Homebrew
        |
        v
Zsh / Oh My Zsh
        |
        v
nvm / rbenv / pyenv
```

## 2. 安装 Xcode Command Line Tools

Homebrew、rbenv、pyenv 编译语言运行时都可能依赖系统编译工具。

```bash
xcode-select --install
```

检查是否安装成功：

```bash
xcode-select -p
git --version
clang --version
```

如果系统提示已经安装，可以跳过。

## 3. 安装 Homebrew

Homebrew 是 macOS 最常用的软件包管理器，可以安装命令行工具、GUI 应用、后台服务和开发依赖。

官方安装命令：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Apple Silicon Mac 默认安装在：

```text
/opt/homebrew
```

Intel Mac 默认安装在：

```text
/usr/local
```

安装完成后，如果终端提示需要写入 shell 配置，按提示执行。Apple Silicon 常见配置如下：

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Intel Mac 常见配置如下：

```bash
echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/usr/local/bin/brew shellenv)"
```

检查安装：

```bash
brew --version
brew doctor
```

### 3.1 Homebrew 常用命令

```bash
# 搜索软件包
brew search git

# 查看软件包信息
brew info git

# 安装命令行工具
brew install git

# 安装 GUI 应用
brew install --cask visual-studio-code

# 查看已安装包
brew list
brew list --cask

# 更新 Homebrew 自身和软件源
brew update

# 升级已安装的软件包
brew upgrade

# 升级 GUI 应用，包含部分自更新应用
brew upgrade --cask --greedy

# 卸载软件包
brew uninstall git
brew uninstall --cask visual-studio-code

# 清理旧版本缓存
brew cleanup

# 查看环境诊断
brew doctor

# 查看包安装路径
brew --prefix
brew --prefix git
```

### 3.2 建议安装的基础工具

```bash
brew install git curl wget tree jq ripgrep fd fzf
```

常见 GUI 应用示例：

```bash
brew install --cask visual-studio-code iterm2
```

## 4. 配置 Zsh 与 Oh My Zsh

macOS 默认 Shell 通常已经是 Zsh。先确认当前 Shell：

```bash
echo $SHELL
zsh --version
```

如果不是 Zsh，可以切换：

```bash
chsh -s /bin/zsh
```

安装 Oh My Zsh：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

安装后主要配置文件是：

```text
~/.zshrc
```

### 4.1 常用配置项

修改主题：

```bash
ZSH_THEME="robbyrussell"
```

启用插件：

```bash
plugins=(
  git
  brew
  npm
  node
  python
  ruby
)
```

修改后重新加载配置：

```bash
source ~/.zshrc
```

### 4.2 推荐的 shell 配置分层

| 文件 | 适合放什么 |
|------|------------|
| `~/.zprofile` | 登录 shell 初始化，例如 Homebrew 的 `brew shellenv` |
| `~/.zshrc` | 交互 shell 配置，例如 alias、插件、nvm/rbenv/pyenv 初始化 |
| `~/.zshenv` | 极少量所有 zsh 都需要的环境变量 |

实践上，普通开发者主要维护 `~/.zprofile` 和 `~/.zshrc`。

## 5. Node.js 虚拟环境：nvm

nvm 用来管理多个 Node.js 版本。不同项目可以使用不同 Node 版本，避免全局 Node 互相污染。

### 5.1 安装 nvm

可以使用官方安装脚本。版本号需要以 nvm 官方 README 为准：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

安装完成后，把下面配置加入 `~/.zshrc`。如果安装脚本已自动写入，就不要重复添加。

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

重新加载：

```bash
source ~/.zshrc
```

检查：

```bash
nvm --version
```

### 5.2 nvm 常用命令

```bash
# 查看可安装的远程版本
nvm ls-remote

# 只查看 LTS 版本
nvm ls-remote --lts

# 安装最新版 Node
nvm install node

# 安装最新 LTS
nvm install --lts

# 安装指定版本
nvm install 20
nvm install 20.18.0

# 查看本机已安装版本
nvm ls

# 切换当前终端使用的版本
nvm use 20
nvm use 20.18.0

# 查看当前使用版本
nvm current
node -v
npm -v

# 设置默认版本
nvm alias default 20
nvm alias default 20.18.0

# 卸载指定版本
nvm uninstall 18

# 查看某个版本的 node 路径
nvm which 20
```

### 5.3 项目级 Node 版本

在项目根目录创建 `.nvmrc`：

```bash
20.18.0
```

进入项目后执行：

```bash
nvm use
```

如果该版本尚未安装：

```bash
nvm install
```

团队项目建议提交 `.nvmrc`，让所有人使用相同 Node 主版本或精确版本。

### 5.4 npm 全局包迁移

安装新 Node 版本时迁移旧版本全局包：

```bash
nvm install 22 --reinstall-packages-from=20
```

升级 npm：

```bash
nvm install-latest-npm
```

## 6. Ruby 虚拟环境：rbenv

rbenv 用来管理多个 Ruby 版本。它通过 shims 接管 `ruby`、`gem`、`bundle` 等命令，根据当前目录或全局配置选择版本。

### 6.1 安装 rbenv

通过 Homebrew 安装：

```bash
brew install rbenv ruby-build
```

初始化：

```bash
rbenv init
```

如果希望手动写入 Zsh 配置，可加入 `~/.zshrc`：

```bash
eval "$(rbenv init - zsh)"
```

重新加载：

```bash
source ~/.zshrc
```

检查：

```bash
rbenv --version
rbenv doctor
```

如果 `rbenv doctor` 不存在，可以跳过；不同安装方式可用命令略有差异。

### 6.2 rbenv 常用命令

```bash
# 查看可安装 Ruby 版本
rbenv install -l

# 查看全部可安装版本
rbenv install -L

# 安装指定版本
rbenv install 3.3.6

# 查看已安装版本
rbenv versions

# 查看当前版本
rbenv version
ruby -v

# 设置全局默认 Ruby
rbenv global 3.3.6

# 设置当前项目 Ruby
rbenv local 3.3.6

# 设置当前 shell 临时 Ruby
rbenv shell 3.3.6

# 取消当前 shell 临时版本
rbenv shell --unset

# 卸载版本
rbenv uninstall 3.2.2

# 重新生成 shims
rbenv rehash

# 查看命令实际路径
rbenv which ruby
rbenv which gem
```

### 6.3 项目级 Ruby 版本

执行：

```bash
rbenv local 3.3.6
```

会在当前目录生成：

```text
.ruby-version
```

常见内容：

```text
3.3.6
```

团队项目建议提交 `.ruby-version`。

### 6.4 Bundler 与 gem

安装 Bundler：

```bash
gem install bundler
rbenv rehash
```

安装项目依赖：

```bash
bundle install
```

查看 gem 环境：

```bash
gem env
bundle env
```

## 7. Python 虚拟环境：pyenv

pyenv 用来管理多个 Python 解释器版本。它解决的是 Python 版本管理问题，不等同于 `venv`、`virtualenv` 这类项目依赖隔离工具。

推荐组合：

```text
pyenv 管 Python 解释器版本
venv / virtualenv / poetry / uv 管项目依赖环境
```

### 7.1 安装 pyenv

通过 Homebrew 安装：

```bash
brew install pyenv
```

pyenv 编译 Python 时可能需要额外依赖：

```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

把初始化配置加入 `~/.zshrc`：

```bash
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"
```

重新加载：

```bash
source ~/.zshrc
```

检查：

```bash
pyenv --version
```

### 7.2 pyenv 常用命令

```bash
# 查看可安装 Python 版本
pyenv install -l

# 安装指定版本
pyenv install 3.12.8

# 查看已安装版本
pyenv versions

# 查看当前版本
pyenv version
python --version
python3 --version

# 设置全局默认 Python
pyenv global 3.12.8

# 设置当前项目 Python
pyenv local 3.12.8

# 设置当前 shell 临时 Python
pyenv shell 3.12.8

# 取消当前 shell 临时版本
pyenv shell --unset

# 卸载指定版本
pyenv uninstall 3.11.9

# 重新生成 shims
pyenv rehash

# 查看命令实际路径
pyenv which python
pyenv which pip
```

### 7.3 项目级 Python 版本

执行：

```bash
pyenv local 3.12.8
```

会在当前目录生成：

```text
.python-version
```

常见内容：

```text
3.12.8
```

团队项目可提交 `.python-version`。如果项目需要更细的依赖隔离，还应创建项目虚拟环境。

### 7.4 Python 项目虚拟环境

使用标准库 `venv`：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

退出虚拟环境：

```bash
deactivate
```

常用检查：

```bash
which python
which pip
python --version
pip --version
```

建议把 `.venv/` 加入 `.gitignore`。

## 8. 默认版本与项目版本的优先级

nvm、rbenv、pyenv 都有类似的版本选择思路：

```text
当前 shell 临时版本
        >
当前目录或父目录的项目版本文件
        >
全局默认版本
        >
系统版本
```

| 工具 | 全局默认 | 项目版本文件 | 临时 shell |
|------|----------|--------------|------------|
| nvm | `nvm alias default <version>` | `.nvmrc` | `nvm use <version>` |
| rbenv | `rbenv global <version>` | `.ruby-version` | `rbenv shell <version>` |
| pyenv | `pyenv global <version>` | `.python-version` | `pyenv shell <version>` |

## 9. 推荐初始化片段

下面是一个常见 `~/.zshrc` 片段。按需启用，不要重复添加。

```bash
# Oh My Zsh
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(git brew npm node python ruby)
source "$ZSH/oh-my-zsh.sh"

# nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# rbenv
eval "$(rbenv init - zsh)"

# pyenv
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - zsh)"
```

Homebrew 的 shellenv 更适合放到 `~/.zprofile`：

```bash
# Apple Silicon
eval "$(/opt/homebrew/bin/brew shellenv)"

# Intel Mac
# eval "$(/usr/local/bin/brew shellenv)"
```

## 10. 常见问题

### 10.1 command not found: brew

检查 Homebrew 路径是否写入 shell：

```bash
which brew
echo $PATH
```

Apple Silicon：

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Intel Mac：

```bash
eval "$(/usr/local/bin/brew shellenv)"
```

### 10.2 nvm 安装后找不到命令

检查 `~/.zshrc` 中是否有：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

重新加载：

```bash
source ~/.zshrc
```

### 10.3 rbenv / pyenv 设置版本后无效

通常是 shims 没有在 PATH 前面，或 shell 初始化没有生效。

检查：

```bash
which ruby
rbenv which ruby

which python
pyenv which python
```

重新加载配置并刷新 shims：

```bash
source ~/.zshrc
rbenv rehash
pyenv rehash
```

### 10.4 pyenv 编译 Python 失败

先安装编译依赖：

```bash
brew install openssl readline sqlite3 xz zlib tcl-tk
```

再重新安装：

```bash
pyenv install 3.12.8
```

如果仍失败，优先看错误日志中缺失的是哪一个系统库。

### 10.5 全局 npm / gem / pip 包是否应该大量安装

不建议大量安装全局包。更稳妥的原则：

| 生态 | 建议 |
|------|------|
| Node | CLI 工具可全局安装，项目依赖放 `package.json` |
| Ruby | 项目依赖放 `Gemfile`，用 Bundler 管理 |
| Python | 项目依赖放 `.venv`，用 `requirements.txt`、Poetry 或 uv 管理 |

## 11. 最小验证清单

完成搭建后执行：

```bash
brew --version
zsh --version
nvm --version
node -v
npm -v
rbenv --version
ruby -v
gem -v
pyenv --version
python --version
pip --version
```

确认路径：

```bash
which brew
which node
which ruby
which python
```

正常情况下：

- `brew` 应来自 `/opt/homebrew/bin` 或 `/usr/local/bin`
- `node` 应来自 `~/.nvm`
- `ruby` 应经过 `rbenv/shims`
- `python` 应经过 `pyenv/shims`，或来自当前项目 `.venv`

## 12. 参考资料

- Homebrew: https://brew.sh/
- Oh My Zsh: https://ohmyz.sh/
- nvm: https://github.com/nvm-sh/nvm
- rbenv: https://github.com/rbenv/rbenv
- pyenv: https://github.com/pyenv/pyenv

## 13. 总结

macOS 基础开发环境的核心是分层管理：

| 层级 | 管理方式 |
|------|----------|
| 系统工具 | Xcode Command Line Tools |
| 软件安装 | Homebrew |
| 终端体验 | Zsh + Oh My Zsh |
| Node 版本 | nvm + `.nvmrc` |
| Ruby 版本 | rbenv + `.ruby-version` |
| Python 版本 | pyenv + `.python-version` + `.venv` |

不要直接依赖系统自带的 Ruby 或 Python 做项目开发；系统版本主要服务 macOS 自身和系统脚本。项目开发应使用版本管理器和项目级版本文件固定运行时。
