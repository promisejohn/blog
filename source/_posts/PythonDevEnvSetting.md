title: "Python开发环境搭建"
date: 2015-04-15 16:03:04
tags: [python, dev]
categories: [Tech]

---

# Python开发环境搭建

## 用阿里Yun的源给pip和easy_install加速
~/.bashrc：

```
alias easy_install='easy_install -i http://mirrors.aliyun.com/pypi/simple/'
```
安装pip：

```
$ easy_install pip
```
~/.pip/pip.conf：

```
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
```

## 多版本管理pyenv, virtualenv

```
$ git clone git://github.com/yyuu/pyenv.git ~/.pyenv
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
$ echo 'eval "$(pyenv init -)"' >> ~/.bashrc
$ exec $SHELL -l
$ export PYTHON_BUILD_MIRROR_URL="http://pyenv.qiniudn.com/pythons/" #加速, Official: http://yyuu.github.io/pythons/
```
安装pyenv-virtualenv plugin：

```
$ git clone https://github.com/yyuu/pyenv-virtualenv.git ~/.pyenv/plugins/pyenv-virtualenv
$ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile
$ exec "$SHELL"
```

安装centOS系统开发环境工具：

```
$ yum groupinstall -y "Development Tools"
$ yum install -y readline readline-devel readline-static openssl openssl-devel openssl-static sqlite-devel bzip2-devel bzip2-libs
```

安装隔离的python环境：

```
$ pyenv install --list
$ pyenv install 2.7.8
$ pyenv rehash
```
查看切换版本：

```
$ pyenv versions
$ pyenv global 2.7.8 #OR Use: pyenv local 2.7.8
```
新建环境，检查环境列表：

```
$ pyenv virtualenv 2.7.8 my-2.7.8 ＃OR from current version: pyenv virtualenv venv34
$ pyenv virtualenvs
```
Activate环境：

```
pyenv activate <name>
pyenv deactivate
```

卸载环境，与卸载某个python版本相同：

```
pyenv uninstall my-virtual-env
```

相关变量：

```
PYENV_VIRTUALENV_CACHE_PATH
VIRTUALENV_VERSION
EZ_SETUP/GET_PIP # use ez_setup.py and get_pip.py from the specified location.
EZ_SETUP_URL/GET_PIP_URL # download ez_setup.py and get_pip.py from the specified URL.
SETUPTOOLS_VERSION/PIP_VERSION # install the specified version of setuptools and pip.
```

## 建立自己的PyPi服务器

目前可以用Artifactory, Sonatype Nexus等作为Java的私有仓库和Mirros，虽然Artifactory Pro版也支持PyPi，甚至yum、npm等其他仓库，但目前看要$1000+，期望后续Sonatype Nexus有免费的套餐～
稍微麻烦点可以自己架设一个devpi：

```bash
$ pip install devpi-server
$ devpi-server --version # 查看版本
$ devpi-server --start
$ pip install -i http://localhost:3141/root/pypi/ simplejson #简单测试
$ devpi-server --status
$ devpi-server --stop
$ devpi-server --log
```
正式部署及其他方案可以参考下面。

## 参考：

1. [Python多版本共存之pyenv][python1]
2. [用pyenv 和 virtualenv 搭建单机多版本python 虚拟开发环境][python2]
3. [pyenv-virtualenv Official site][python3]
4. [Quickstart: running a pypi mirror on your laptop][devpy1]
5. [Quickstart: permanent install on server/laptop][devpy2]
6. [Quickstart: uploading, testing, pushing releases][devpy3]
7. [Survey of Existing PyPI Implementations][python4]

[python1]: http://seisman.info/python-pyenv.html "Python多版本共存之pyenv"
[python2]: http://www.cnblogs.com/npumenglei/p/3719412.html "用pyenv 和 virtualenv 搭建单机多版本python 虚拟开发环境"
[python3]: https://github.com/yyuu/pyenv-virtualenv "pyenv-virtualenv Official site"
[devpy1]: http://doc.devpi.net/latest/quickstart-pypimirror.html "Quickstart: running a pypi mirror on your laptop"
[devpy2]: http://doc.devpi.net/latest/quickstart-server.html "Quickstart: permanent install on server/laptop"
[devpy3]: http://doc.devpi.net/latest/quickstart-releaseprocess.html "Quickstart: uploading, testing, pushing releases"
[python4]: https://github.com/teamfruit/defend_against_fruit/wiki/Survey-of-Existing-PyPI-Implementations "Survey of Existing PyPI Implementations"