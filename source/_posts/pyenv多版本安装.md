
title: pyenv多版本安装
date: 2017-07-04 11:18:32
tags: [youdaonote]
---

```
yum install -y git python-pip
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
 yum install readline readline-devel readline-static openssl openssl-devel openssl-static sqlite-devel bzip2-devel bzip2-libs -y
 pyenv install 3.6.1
 git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
 pyenv shell 3.6.1
 echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile
 pyenv virtualenv 3.6.1 will_3.6.1
 
 ```
 
 ```
 pyenv versions
 
 pyenv activate  will_3.6.1
 ```
 
 
 安装加速: 提前下载好对应python版本的tar.zx包，放到~/.pyenv/cache/下，然后再运行 pyenv install 3.6.1
