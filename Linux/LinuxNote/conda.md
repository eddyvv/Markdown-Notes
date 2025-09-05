```bash
# 查看虚拟环境列表
conda env list
conda info -e

# 创建虚拟环境
conda create -n <env_name> [python_version] [package_name]
conda create -n pyenv python=3.6 requests
conda create -n pyenv

# 复制虚拟环境
conda create --name <new_env_name> --clone <old_env_name>

# 删除虚拟环境
conda remove -n <env_name> --all

# 激活虚拟环境
conda activate <env_name> 

# 退出虚拟环境(进入环境状态下才可使用)
conda deactivate 

# conda 设置不自动进入base(最基础)环境
conda config --set auto_activate_base false
```









[Miniconda 安装及使用 - 掘金 (juejin.cn)](https://juejin.cn/post/7078965942968909854)