# Conda and Pip Notes

记录一些常用的Conda和Pip指令含义：

### Conda

+ `conda env create -f environment.yml`: 根据当前目录下的`environment.yml` 文件创建conda环境

1. 克隆环境(从A->B) ：conda create -n B --clone A
2. 删除环境：conda remove -n your_env_name --all或者conda create -n B --clone A
3. 添加特定版本环境：conda create --name Python35 python=3.5
4. 设置非自动启动：conda config --set auto_activate_base false # 设置非自动启动

### Pip

+ `pip install -e .`: `-e`代表以可编辑模式进行python包的安装，`.`代表安装到当前路径，通常会有一个名为`setup.py`的文件定义了包的元数据和安装要求
  
具体原理是该命令会创建一个指向源代码目录的符号链接，因此对源代码的任何更改都会立即反映在安装的包中。