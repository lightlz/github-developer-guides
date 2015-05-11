# 集成自动化部署          

i. Sending deployments whenever you push to a repository          

ii. Hooking up an integrator to deployments          

“[Delivering deployments](https://developer.github.com/guides/delivering-deployments/)”指南描述了如何建立一个服务去使用[Deployments API](https://developer.github.com/v3/repos/deployments/)轻松的从Github获取你的编码并投入工作中使用。但如果你不想为了获取编码而建立一个单独的服务，或者你只是想merge一些编码，在不考虑兼容其它应用的情况下部署它呢？          

你可以使用GitHub自动部署服务去接收任何对仓库的修改，并且配置它部署集成。自动部署服务基于两个事件提供有效负载：当一个push发生或当[CI的状态改变](https://developer.github.com/guides/building-a-ci-server/)。        

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

注意：自动部署服务只从你的默认分支获取修改内容，通常是`master`分支            

## Sending deployments whenever you push to a repository          

自动部署服务负责在你的分支接收到一个push的时候创建一个deployment。接着，我们会建立服务员去接收那些deployment事件信息并且处理你工程的deployment。          


1 Navigate to the repository where you’re setting up your deployments。       

2 In your repository’s right sidebar, click       

3 On the left，click ""Webhooks & Services""。

4 Click ""Add service""，then type “GitHub Auto-Deployment.”

5 Under ""GitHub token""，paste an access token you’ve created。It must have at least the repo scope。For more information，see “[Creating an access token for command-line use](https://help.github.com/articles/creating-an-access-token-for-command-line-use/).”                

6 Under ""Environments""，optionally provide a list of environments you’d like to send your deployments to。This can be [any string you define](https://developer.github.com/v3/repos/deployments/#parameters) to describe your environment. The default is “production.”             

7 If you only want builds that successfully passed a continuous test suite， select ""Deploy on status""。            

8 If you’re running this service on GitHub Enterprise, you must pass in your appliance’s [endpoint URL](https://developer.github.com/v3/enterprise/#endpoint-urls)。         

9 点击 ""Add service""。         

## Hooking up an integrator to deployments       

现在起，所有提交到你`master分支`的commit，包括和pull request进行merge产生的，都会自动向你的Heroku应用触发一个deployment。