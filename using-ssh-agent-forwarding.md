# 使用SSH agent 转发功能 #

- i.	设置 SSH agent 转发功能
- ii.	测试 SSH agent 转发功能
- iii.	SSH agent 转发功能的疑难解答

**SSH Agent 转发功能**可以让您在部署至服务器的过程更简便。比起让您的（无密码保护的）密钥存放在服务器上，SSH 代理转发功能可以让您使用本地的 SSH 密钥。

如果您已经设置好 SSH 密钥来和 Github 交换数据，可能已经对 ssh-agent 感到熟悉。 ssh-agent 是一个在后台运行的应用程序，它会缓存您已经加载到内存中的密钥，这样便不必每次使用这个密钥都输入密码了。更棒的是，您可以选择让服务器访问您的本地 ssh-agent ，效果如同 ssh-agent 已经在服务器上运行。这有点像当您借用朋友的电脑时，让朋友帮忙输入密码以便让您使用。

查看  [Steve Friedl 的技术提示指南](http://www.unixwiz.net/techtips/ssh-agent-forwarding.html) 来获得关于 SSH Agent 转发功能更详细的解释。

## 设置 SSH agent 转发功能 ##

请确保您自己的 SSH 密钥已经设置并且可用。 如果还没有完成这步，可以查看我们的 [生成 SSH 密钥指南](https://help.github.com/articles/generating-ssh-keys) 。

您可以在终端中使用命令 `ssh -T git@github.com` 来测试您的本地密钥是否可用：

    $ ssh -T git@github.com
    # 尝试通过SSH连接Github
    Hi username! You've successfully authenticated, but GitHub does not provide
    shell access.


我们现在有了一个良好的开端。让我们设置 SSH 来允许 agent 转发到您的服务器。



1.使用您最喜欢的文本编辑器, 打开文件 `~/.ssh/config`。 如果这个文件不存在，可以在终端中使用命令`touch ~/.ssh/config` 来创建一个。


2.将以下文本输入到文件中，用您的服务器的域名或者 IP 地址来替换 `example.com`：
    
    Host example.com
      ForwardAgent yes


> 警告: 您也许会想投机取巧地使用类似`Host *`这样的万金油语句来设置所有的 SSH 连接。这并不是一个好办法，因为这样做会分享您的本地 SSH 密钥给您用 SSH 连接过的所有服务器。 虽然他们不能直接访问您的本地密钥，但是他们可以以*您的名义*来使用这些密钥，只要连接尚未断开。 **所以请务必只输入您信任的并且打算使用 agent 转发功能的服务器。**

## 测试 SSH agent 转发功能##

您可以用 SSH 连接到您的服务器并再次执行命令 `ssh -T git@github.com` 来测试 agent 转发是否有效。如果情况顺利，您会得到和本地操作一致的提示。

如果您不能确定目前是否正在使用本地密钥，可以通过检视服务器上的 `SSH_AUTH_SOCK` 变量：

    $ echo "$SSH_AUTH_SOCK"
    # 打印输出 SSH_AUTH_SOCK 变量
    /tmp/ssh-4hNGMk8AZX/agent.79453

如果这个变量尚未设定，说明 agent 转发功能没有运作。

    $ echo "$SSH_AUTH_SOCK"
    # 打印输出 SSH_AUTH_SOCK 变量
    [No output]
    $ ssh -T git@github.com
    # 尝试通过 SSH 连接 Github
    Permission denied (publickey).

## SSH agent 转发功能的疑难解答 ##

如果您在 SSH agent 转发功能中遇到问题，以下几点可以助您排除故障。

**您必须使用 SSH URL 来检查代码**

SSH 转发功能只能在 SSH URL 下使用, 不包括 HTTP(s) URL 。 检查服务器上的 *.git/config* 文件并且确保 URL 是 SSH 格式的，类似下文：

    [remote "origin"]
      url = git@github.com:yourAccount/yourProject.git
      fetch = +refs/heads/*:refs/remotes/origin/*

**您的 SSH 密钥必须本地可用**

在您的密钥能通过 agent 转发功能使用之前，这些密钥必须在本地上也能正常使用。我们的 [生成SSH密钥指南](https://help.github.com/articles/generating-ssh-keys) 可以帮助您在本地设置您的 SSH 密钥。

**您的系统必须允许 SSH agent 转发功能**

某些时候，系统配置会不允许 SSH agent 转发功能。您可以通过终端输入以下命令来检查目前是否有系统配置文件正在生效：

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

在这个示例中，我们的 */etc/ssh_config* 文件显然写着 `ForwardAgent no`，这是阻止 agent 转发功能工作的方法之一。将这行从文件中删去应当可以让 agent 转发功能再次正常工作。

**您的服务器必须允许 SSH agent 转发功能可以用于入站连接上**

agent 转发功能也可能是被您的服务器阻止了。您可以通过 SSH 连接服务器并且执行 `sshd_config` 来检查 agent 转发功能是否被许可。该命令的输出应该指出 `AllowAgentForwarding` 已被设置.

**您的本地** ssh-agent **必须处于运行中状态**

在大多数电脑上，操作系统会自动为您启动 `ssh-agent`。然而在 Windows 上，您需要手动设置。我们有 [一个指南教您如何随打开 Git Bash 时启动`ssh-agent`](https://help.github.com/articles/working-with-ssh-key-passphrases#auto-launching-ssh-agent-on-msysgit).

如果想确认 `ssh-agent` 正在您的电脑上运行，请在终端中输入以下命令：

    $ echo "$SSH_AUTH_SOCK"
    # 打印输出 SSH_AUTH_SOCK 变量
    /tmp/launch-kNSlgU/Listeners

**您的密钥必须对于** ssh-agent **可见**

您可以通过以下命令检查您的密钥是否对 `ssh-agent` 可见：

    ssh-add -L

如果该命令指出没有身份可用，那么您需要通过以下命令添加您的密钥：

    ssh-add 您的密钥

> 在 Mac OS X 上，当系统重新启动后，ssh-agent 再次启动时会 “忘记” 这个密钥。不过您可以通过以下命令将您的 SSH 密钥导入到密钥链中：
> 
>     /usr/bin/ssh-add -K 您的密钥