# 集成自动化部署          

i. 当你push到仓库是时候同时发送deployment          

ii. 连接一个integrator去部署          

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
           

## 当你push到仓库是时候同时发送deployment            

自动部署服务负责在你的分支接收到一个push的时候创建一个deployment。接着，我们会建立服务员去接收那些deployment事件信息并且处理你工程的deployment。          


1 引导到你设置deployment的那个仓库             

2 在你仓库的右侧边栏, 点击 _Settings_                       

3 在左边，click __Webhooks & Services__。![image](https://github.com/Leolusir/github-developer-guides/blob/master/images/webhooks_and_services_menu.png?imageView/1/w/50/q/10)                  

4 点击 __Add service__，然后键入 “GitHub Auto-Deployment.” ![image](https://github.com/Leolusir/github-developer-guides/blob/master/images/add_heroku_autodeploy_service.png?imageView/1/w/100/q/20)        

5 在 __GitHub token__ 下面，粘贴一个你已经创建的 access token。里面至少有 repo 的范围。要了解更多内容，查看 “[Creating an access token for command-line use](https://help.github.com/articles/creating-an-access-token-for-command-line-use/).”                

6 在 __Environments__ 下面，列出了一些环境供你选择去将deployment发送进去。可以使用 [你定义的任何字符串](https://developer.github.com/v3/repos/deployments/#parameters) 去描述你的环境。 默认是 “production.”             

7 如果你只是想建立了成功通过连续测试的套件，请选择 __Deploy on status__。            

8 如果你正在Github上运行这个服务, 你必须通过你设备的 [endpoint URL](https://developer.github.com/v3/enterprise/#endpoint-urls)。         

9 点击 __Add service__。         

## 连接一个integrator去部署      

现在起，所有提交到你`master`分支的commit，包括和pull request进行merge产生的，都会自动向你的Heroku应用触发一个deployment。