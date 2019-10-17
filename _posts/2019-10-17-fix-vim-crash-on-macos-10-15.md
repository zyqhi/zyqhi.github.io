---
title: macOS 10.15 Vim crash修复之路
tags: Vim Python
typora-root-url: "/Users/mutsu/project/blog/zyqhi.github.io"
---

macOS 10.15 Catalina更新以后，我第一时间更新了系统。但这次系统升级遇到的问题，比我预期的要多。对我而言，其中影响比较大的是MacVim不能用了。

# Crash 1

每次启动是都会报如下错误：

```bash
dyld: Library not loaded: /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/libruby.2.3.0.dylib
  Referenced from: /usr/local/Cellar/macvim/8.1-157/MacVim.app/Contents/MacOS/Vim
  Reason: image not found
fish: 'vim' terminated by signal SIGABRT (Abort)
```

Google一下，这个crash很快便解决了（https://github.com/macvim-dev/macvim/issues/947）。这个问题应该是MacVim自身的问题，解决方案是安装最新版本：

``` bash
brew install macvim --HEAD
```

# Crash 2

第一个crash解决之后，每次打开vim仍然会crash，日志如下：

```bash
➜  client git:(master) ✗ Vim: Caught deadly signal ABRT
Error detected while processing function <SNR>144_OnVimLeave:Vim: Finished.

line    6:
Traceback (most recent call last):
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/base_request.py", line 193, in Session
    return cls.session
AttributeError: type object 'BaseRequest' has no attribute 'session'
During handling of the above exception, another exception occurred:
Traceback (most recent call last):
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/base_request.py", line 193, in Session
    return cls.session
AttributeError: type object 'BaseRequest' has no attribute 'session'
During handling of the above exception, another exception occurred:
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/youcompleteme.py", line 498, in OnVimLeave
    self._ShutdownServer()
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/youcompleteme.py", line 283, in _ShutdownServer
    SendShutdownRequest()
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/shutdown_request.py", line 45, in SendShutdownRequest
    request.Start()
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/shutdown_request.py", line 39, in Start
    display_message = False )
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/base_request.py", line 127, in PostDataToHandler
    BaseRequest.PostDataToHandlerAsync( data, handler, timeout ),
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/base_request.py", line 137, in PostDataToHandlerAsync
    return BaseRequest._TalkToHandlerAsync( data, handler, 'POST', timeout )
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/base_request.py", line 152, in _TalkToHandlerAsync
    return BaseRequest.Session().post(
  File "/Users/mutsu/.vim/bundle/YouCompleteMe/python/ycm/client/base_request.py", line 196, in Session
    from requests_futures.sessions import FuturesSession
ImportError: cannot import name 'FuturesSession' from 'requests_futures.sessions' (/Users/mutsu/.vim/bundle/YouCompleteMe/third_party/requests-futures/requests_futures/sessions.py)
```

从日志来看，是YouCompleteMe插件导致的crash，关于这个crash，在该插件的issue面板也有讨论：https://github.com/ycm-core/YouCompleteMe/issues/3504。从讨论来看，该crash出现并不具有普遍性，开发者回应称并非YouCompleteMe本身的bug，而是本机Python环境的原因。

于是，我做了以下尝试：

- 重装YouCompleteMe插件
- 卸载MacVim，安装Vim
- 重新安装Python2/3
- 安装pyenv，精确控制Python版本
- 尝试Python3.7.4/3.7.3版本
- ...

皆不能解决我的问题。我都有点抑郁了，感觉遇到了束手无策的事情。转机出现在**[@puremourning](https://github.com/puremourning)**的建议：

> Try removing the site packages you have in your homebrew python.
>
> Closing as not YCM issue.

他建议我移除homebrew版本Python的site packages，我检查了一下，我系统上存在3个Python site packages目录，分别为：

- /usr/local/lib/python2.7/site-packages
- /usr/local/lib/python3.6/site-packages
- /usr/local/lib/python3.7/site-packages

于是我先将3个目录先备份，然后全部移除，问题解决了🎉🎉🎉。目前我不知道我这种做法会带来那种风险，但是可以确定此次crash是Python导致。

此处只是移除了homebrew版本Python的第三方库，在此之前我通过pyenv安装了Python 3.7.4，并将其作为全局默认的Python。
{:.info}

# pyenv

这次升级遇到问题主要是因为Python环境导致，Python长期以来都是2.x和3.x两个版本共存，由于平常不做Python开发，所以也没有对版本过多的关注。之前建立此博客时，也遇到Ruby版本问题，导致博客部署失败，后来通过安装rbenv才终得解决。

pyenv和rbenv采用类似的管理方式。首先安装pyenv：

```bash
brew install pyenv
```

通过pyenv安装Python：

```bash
export PYTHON_CONFIGURE_OPTS="--enable-framework"
pyenv install 3.7.3
```

>  因为YouCompleteMe需要链接Python动态库，所以安装之前需要添加配置`--enable-framework`，否则安装YouCompleteMe时会报如下错误：
>
> ```bash
> $ ./install.py --clangd-completer
> Searching Python 3.7 libraries...
> ERROR: found static Python library (/Users/mutsu/.pyenv/versions/3.7.4/lib/python3.7/config-3.7m-darwin/libpython3.7m.a) but a dynamic one is required. You must use a Python compiled with the --enable-framework flag. If using pyenv, you need to run the command:
>   export PYTHON_CONFIGURE_OPTS="--enable-framework"
> before installing a Python version.
> ```

在`.zshrc`中添加pyenv初始化代码：

```bash
# ~/.zshrc
eval "$(pyenv init -)"
source ~/.zshrc
```

可以通过pyenv查看当前的Python及pip的版本以及安装路径：

```bash
$ pyenv versions
  system
  3.7.3
* 3.7.4 (set by /Users/mutsu/.pyenv/version)

$ which pip
/Users/mutsu/.pyenv/shims/pip

$ which python
/Users/mutsu/.pyenv/shims/python
```

上述表明当前系统安装了3.7.3和3.7.4两个版本，且当前环境的默认版本为3.7.4。

# 关于排查

排查期间由于不清楚原因，做了许多排查，也增长一些知识，记录如下：

- 不加载任何配置，启动Vim：

  ```bash
  vim --clean
  ```

- 查看Vim的Python版本，在Vim ex mode下输入命令：

  ```
  :py3 import sys; print(sys.version)
  ```

- Vim内执行Python代码：

  ```
  :py3 from requests_futures.sessions import FuturesSession
  ```

- shell内执行Python代码：

  ```bash
  python -c 'from requests_futures.sessions import FuturesSession'
  ```



此次crash问题本质还是Python环境的问题，但问题具体出在哪里我不是特别清楚，能想到的解决方式也是这种暴力的重置环境的方式。

但是crash可能的原因有Vim/YouCompleteMe/Python三种，最终定位到是Python导致的，期间做过的测试、调查、求助还是有代表意义的。

# 参考

- [pyenv](https://opensource.com/article/19/5/python-3-default-mac)
- [MacVim crash](https://github.com/macvim-dev/macvim/issues/947)
- [YouCompleteMe crash](https://github.com/ycm-core/YouCompleteMe/issues/3504)
- [reset site packages](https://stackoverflow.com/questions/7387453/how-to-reset-site-packages)