# 使用SSH agent 转发功能 #

- i.	设置 SSH agent 转发功能
- ii.	测试 SSH agent 转发功能
- iii.	SSH agent 转发功能的疑难解答

**SSH Agent转发功能**可以让您在部署至服务器的过程更简便。比起让您的（无密码保护的）密钥存放在服务器上，SSH代理转发功能可以让您使用本地的SSH密钥。

如果您已经设置好SSH密钥来和Github交换数据，可能已经对ssh-agent感到熟悉。 ssh-agent是一个在后台运行的应用程序，它会缓存您已经加载到内存中的密钥，这样便不必每次使用这个密钥都输入密码了。更棒的是，您可以选择让服务器访问您的本地ssh-agent，效果如同ssh-agent已经在服务器上运行。这有点像当您借用朋友的电脑时，让朋友帮忙输入密码以便让您使用。

查看 [Steve Friedl 的技术提示指南](http://www.unixwiz.net/techtips/ssh-agent-forwarding.html)来获得关于 SSH Agent转发功能更详细的解释。

## 设置 SSH agent 转发功能 ##

请确保您自己的SSH密钥已经设置并且可用。 如果还没有完成这步，可以查看我们的[生成SSH密钥指南](https://help.github.com/articles/generating-ssh-keys)。

您可以在终端中使用命令 `ssh -T git@github.com` 来测试您的本地密钥是否可用：

    $ ssh -T git@github.com
    # 尝试通过SSH连接Github
    Hi username! You've successfully authenticated, but GitHub does not provide
    shell access.


我们现在有了一个良好的开端。让我们设置SSH来允许agent转发到您的服务器。



1.使用您最喜欢的文本编辑器, 打开文件 `~/.ssh/config`。 如果这个文件不存在，可以在终端中使用命令`touch ~/.ssh/config` 来创建一个。


2.将以下文本输入到文件中，用您的服务器的域名或者IP地址来替换 `example.com`：
    
    Host example.com
      ForwardAgent yes



## 测试 SSH agent 转发功能##

您可以用SSH连接到您的服务器并再次执行命令 `ssh -T git@github.com` 来测试agent转发是否有效。如果情况顺利，您会得到和本地操作一致的提示。

如果您不能确定目前是否正在使用本地密钥，可以通过检视服务器上的 `SSH_AUTH_SOCK` 变量：

    $ echo "$SSH_AUTH_SOCK"
    # 打印输出 SSH_AUTH_SOCK 变量
    /tmp/ssh-4hNGMk8AZX/agent.79453

如果这个变量尚未设定，说明agent转发功能没有运作。

    $ echo "$SSH_AUTH_SOCK"
    # 打印输出 SSH_AUTH_SOCK 变量
    [No output]
    $ ssh -T git@github.com
    # 尝试通过SSH连接Github
    Permission denied (publickey).

## SSH agent 转发功能的疑难解答 ##

如果您在SSH agent 转发功能中遇到问题，以下几点可以助您排除故障。

**您必须使用SSH URL来检查代码**

SSH转发功能只能在SSH URL下使用, 不包括HTTP(s) URL。 检查服务器上的 *.git/config* 文件并且确保URL是SSH格式的，类似下文：

    [remote "origin"]
      url = git@github.com:yourAccount/yourProject.git
      fetch = +refs/heads/*:refs/remotes/origin/*

**您的SSH密钥必须本地可用**

在您的密钥能通过agent转发功能使用之前，这些密钥必须在本地上也能正常使用。我们的[生成SSH密钥指南](https://help.github.com/articles/generating-ssh-keys)可以帮助您在本地设置您的SSH密钥。

**您的系统必须允许 SSH agent 转发功能**

某些时候，系统配置会不允许SSH agent转发功能。您可以通过终端输入以下命令来检查目前是否有系统配置文件正在生效：

    $ ssh -v example.com
    # 连接到 example.com，附带详细除错输出
    OpenSSH_5.6p1, OpenSSL 0.9.8r 8 Feb 2011
    debug1: Reading configuration data /Users/you/.ssh/config
    debug1: Applying options for example.com
    debug1: Reading configuration data /etc/ssh_config
    debug1: Applying options for *
    $ exit
    # 返回到您的本地命令提示符

在上面的示例中，*~/.ssh/config* 文件最先被载入，然后是 */etc/ssh_config* 文件。我们可以通过下面的命令来确认后来被载入的文件是否覆盖了我们的设置：

    $ cat /etc/ssh_config
    # 打印输出 /etc/ssh_config 文件
     Host *
       SendEnv LANG LC_*
       ForwardAgent no

在这个示例中，我们的 */etc/ssh_config* 文件显然写着 `ForwardAgent no`，这是阻止agent转发功能工作的方法之一。将这行从文件中删去应当可以让agent转发功能再次正常工作。

**您的服务器必须允许 SSH agent 转发功能可以用于入站连接上**

agent转发功能也可能是被您的服务器阻止了。您可以通过SSH连接服务器并且执行 `sshd_config` 来检查agent转发功能是否被许可。该命令的输出应该指出 `AllowAgentForwarding` 已被设置.

**您的本地** ssh-agent **必须处于运行中状态**

在大多数电脑上，操作系统会自动为您启动 `ssh-agent`。然而在Windows上，您需要手动设置。我们有[一个指南教您如何随打开 Git Bash 时启动`ssh-agent`](https://help.github.com/articles/working-with-ssh-key-passphrases#auto-launching-ssh-agent-on-msysgit).

如果想确认 `ssh-agent` 正在您的电脑上运行，请在终端中输入以下命令：

    $ echo "$SSH_AUTH_SOCK"
    # 打印输出 SSH_AUTH_SOCK 变量
    /tmp/launch-kNSlgU/Listeners

**您的密钥必须对于** ssh-agent **可见**

您可以通过以下命令检查您的密钥是否对 `ssh-agent` 可见：

    ssh-add -L

如果该命令指出没有身份可用，那么您需要通过以下命令添加您的密钥：

    ssh-add 您的密钥

> 在Mac OS X上，当系统重新启动后，ssh-agent再次启动时会“忘记”这个密钥。不过您可以通过以下命令将您的SSH密钥导入到密钥链中：
> 
>     /usr/bin/ssh-add -K 您的密钥