title: "WebDevEnvSetting"
date: 2015-04-15 16:03:04
tags: [dev]
categories: [dev]

---

### Python setting

用豆瓣的源给pip和easy_install加速，~/.bashrc：

```
alias easy_install='easy_install -i http://pypi.douban.com/simple'
```
安装pip：

```
$ easy_install pip
```
~/.pip/pip.conf：

```
[global]
index-url = http://pypi.douban.com/simple
```

安装pyenv

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


参考：

1. [Python多版本共存之pyenv][python1]
2. [用pyenv 和 virtualenv 搭建单机多版本python 虚拟开发环境][python2]
3. [pyenv-virtualenv Official site][python3]

[python1]: http://seisman.info/python-pyenv.html "Python多版本共存之pyenv"
[python2]: http://www.cnblogs.com/npumenglei/p/3719412.html "用pyenv 和 virtualenv 搭建单机多版本python 虚拟开发环境"
[python3]: https://github.com/yyuu/pyenv-virtualenv "pyenv-virtualenv Official site"

### NodeJS setting


### GCC setting