# 集成自动化部署          

i. 当你 push 到仓库是时候同时发送 deployment           
ii. 连接一个 integrator 去部署         
 

[Delivering deployments](https://developer.github.com/guides/delivering-deployments/) 指南描述了如何建立一个服务去使用 [Deployments API](https://developer.github.com/v3/repos/deployments/) 轻松的从 Github 获取你的代码并投入工作中使用。但如果你不想为了获取代码而建立一个单独的服务，或者你只是想 merge 一些代码，在不考虑兼容其它应用的情况下部署它呢？          

你可以使用 GitHub 自动部署服务去接收任何对仓库的修改，并且配置它部署集成。自动部署服务基于两个事件提供有效负载：当一个 push 发生或当 [CI的状态改变](https://developer.github.com/guides/building-a-ci-server/)。      

这里有一个过程示范图：          

```
+--------------------+        +--------+                    +-----------+
| GitHub Auto-Deploy |        | GitHub |                    |  Heroku   |
|      Service       |        +--------+                    +-----------+
+--------------------+         |                                  |
     |                         |                                  |
     |  Create Deployment      |                                  |
     |------------------------>|                                  |
     |                         |                                  |
     |                         |                                  |
     |                         |       Deployment Event           |
     |                         |--------------------------------->|
     |                         |                                  |
     |                         |    Deployment Status (pending)   |
     |                         |<---------------------------------|
     |                         |                                  |
     |                         |                                  |
     |                         |   Deployment Status (success)    |
     |                         |<---------------------------------|
     |                         |                                  |
```        

注意：自动部署服务只从你的默认分支获取修改内容，通常是 `master` 分支。
           

## 当你 push 到仓库的时候同时发送 deployment                   

自动部署服务负责在你的分支接收到一个 push 的时候创建一个 deployment。接着，我们会建立服务去接收那些 deployment 事件信息并且在你的工程中处理它。


1. 引导到你设置 deployment 的那个仓库。             

2. 在你仓库的右侧边栏, 点击 _Settings_ 。                        

3. 在左边，click __Webhooks & Services__ 。

4. 点击 __Add service__，然后键入 “GitHub Auto-Deployment.”         

5. 在 __GitHub token__ 下面，粘贴一个你已经创建的 access token 。里面至少有 repo 的范围。要了解更多内容，查看 “[Creating an access token for command-line use](https://help.github.com/articles/creating-an-access-token-for-command-line-use/).”                

6. 在 __Environments__ 下面，列出了一些环境供你选择去将 deployment 发送进去。可以使用 [你定义的任何字符串](https://developer.github.com/v3/repos/deployments/#parameters) 去描述你的环境。默认是 “production.”             

7. 如果你只是想建立成功通过连续测试的套件，请选择 __Deploy on status__ 。            

8. 如果你正在 Github 上运行这个服务, 你必须通过你设备的 [endpoint URL](https://developer.github.com/v3/enterprise/#endpoint-urls)。   

9. 点击 __Add service__ 。       

## 连接一个 integrator 去部署  

为了实现我们的 deployments，我们将使用 Heroku 作为示例服务器。 

1. 引导到你设置 deployment 的那个仓库。             

2. 在你仓库的右侧边栏, 点击 _Settings_ 。                        

3. 在左边，click __Webhooks & Services__ 。

4. 点击 __Add service__，然后键入 “Heroku.”

5. 将你需要部署的 GitHub 仓库命名为 Heroku 应用。

6. 输入你的 Heroku OAuth 令牌。你必须根据 Heroku 的文档说明来生成。

7. 在 GitHub token，粘贴你先前使用的令牌。

8. 点击 Add service。

现在起，所有提交到你 `master` 分支的 commit ，包括和 pull request 进行 merge 产生的，都会自动向你的 Heroku 应用触发一个 deployment。       