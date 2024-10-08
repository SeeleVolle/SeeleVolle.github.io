# PackageManager Notes

闲暇之余记录各种包管理器使用......

### 软件安装包的架构版本

+ `x86`: 32位系统专用
+ `x86_64`: x86架构64位系统专用
+ `amd64`: 等于`x86_64`
+ `aarch64`: arm架构64位系统专用
+ `arm64`: 等于`aarch64`


### Conda

+ conda环境安装:

按照[官网](https://docs.anaconda.com/miniconda/), 建议安装体积更小的miniconda
```sh
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm -rf ~/miniconda3/miniconda.sh
```

+ `conda env create -f environment.yml`: 根据当前目录下的`environment.yml` 文件创建conda环境

+ 克隆环境(从A->B) ：conda create -n B --clone A
+ 删除环境：conda remove -n your_env_name --all或者conda create -n B --clone A
+ 添加特定版本环境：conda create --name Python35 python=3.5
+ 设置非自动启动：conda config --set auto_activate_base false # 设置非自动启动

### Pip

+ `pip install -r requirements.txt`: pip的环境配置文件一般是`requirements.txt`

+ `pip install -i https://mirrors.zju.edu.cn/pypi/web/simple some-package --extra-index-url https://pypi.tuna.tsinghua.edu.cn/simple https://mirrors.aliyun.com/pypi/simple`: 用于指定一个索引仓库和备用索引仓库，可以指定多个备用索引仓库
  
    或者编辑`~/.pip/pip.conf`：

    ```bash
        [global]
        index-url = https://pypi.douban.com/simple
        extra-index-url =
            https://pypi.tuna.tsinghua.edu.cn/simple
            https://mirrors.aliyun.com/pypi/simple
            https://pypi.mirrors.ustc.edu.cn/simple
    ```

+ `pip install -e .`: `-e`代表以可编辑模式进行python包的安装，`.`代表安装到当前路径，通常会有一个名为`setup.py`的文件定义了包的元数据和安装要求
具体原理是该命令会创建一个指向源代码目录的符号链接，因此对源代码的任何更改都会立即反映在安装的包中，不需要重新install

    > `pip install -e .`带来的奇怪问题，今天将python包目录和工作目录放在同一文件夹时，发现instaLl后的python包无法正常导入，将python源代码目录放在另一个目录中重新install便恢复
    原因在于`pip install`将包安装到site-packages之后，工作目录中的`run.py`仍然会从当前目录的`vllm/`文件夹导入模块，但是实际模块在`vllm/vllm` 中

+ pip和conda的共存问题，在一个Python环境中安装包时，两者的安装包确实是可能会出现冲突的。

    一般来说，首先使用conda进行安装，找不到对应的包就使用pip安装，一般默认conda环境会有pip，否则需要执行`conda install pip`

    两者指定版本语法区别:

    ```bash
    　　conda install numpy=1.93
    　　pip  install numpy==1.93
    ```

+ pip cache purge: 清除缓存目录文件，一般位于`~/.cache/pip`中，可用`pip cache dir`查看具体缓存路径
### Apt/Dpkg

