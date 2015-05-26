# 传递部署


i. 编写你的服务  
ii. 工作部署  
iii. 结论  


[Deployments API](https://developer.github.com/v3/repos/deployments/)可以让你托管在GitHub上的项目部署到你自己的服务器上。 结合 [the Status API](https://developer.github.com/guides/building-a-ci-server/)，你在提交代码到 master 分支的同时可以调整部署。               


本指南将为你演示一个你可以操作的步骤，在我们的脚本里，我们会：       

- Merge 一个 Pull Request         
- 当 CI 结束时，我们将相应地设置 Pull Request 的状态            
- 当 Pull Request 被 Merged，我们将运行部署到我们的服务器                
         
我们的 CI 系统和主机服务器是我们凭空想象出来的，他们可以是 Heroku，Amazon 或其他一些实体。本指南的关键就是设置和配置服务端去管理通信。                 

如果你还没有准备好，请确定你下载了[ngrok](https://ngrok.com/)，并且学会[使用它](https://developer.github.com/webhooks/configuring/#using-ngrok)。我们发现他是一个监测本地连接非常好用的工具。                    

提示：你可以从 [platform-samples repo](https://github.com/github/platform-samples/tree/master/api/ruby/delivering-deployments) 上下载这个项目完整的源码。                

## 编写你的服务        

我们将编写一个轻便的 Sinatra 应用程序来证明我们的本地连接正在工作。我们这样开始：       

	require 'sinatra'        
	require 'json'          

	post '/event_handler' do           
	payload = JSON.parse(params[:payload])          
	  "Well， it workded！"         
	end         


(如果你不熟悉 Sinatra 如何工作， 我们推荐你阅读 [Sinatra指南](http://www.sinatrarb.com/))             

启动服务器。默认情况下，Sinatra 从 `9393` 端口启动，所以你也会配置 ngrok 去监听这个端口。                   

为了让这台服务器的工作，我们需要创建一个带有一个 webhook 的仓库。无论一个 Pull Resuqest 是被 Merged 还是被创建，webhook 都应该被配置为 fire。继续并创建一个仓库，染护沉浸其中。也许你可以参考 [@octocat’s Spoon/Knife repository](https://github.com/octocat/Spoon-Knife)，之后，你会在你的仓库里创建一个新的 webhook，并将 ngrok 提供你的 URL 填充上去：              

![image](images/webhook_sample_url.png)                    

点击 **Update webhook**。你会看到消息响应中显示了 `Well, it workded!`，很好！点击 **Let me select individual events**.，然后选择下面的选项：             

- Deployment      
- Deployment status     
- Pull Request     

无论发生了什么事件 Github 都会将这些事件发送到我们的服务器上。我们配置服务器只在 Pull Request 正被 merged 的时候处理。                 

```
post '/event_handler' do      
     @payload = JSON.parse(params[:payload])      

     case request.env['HTTP_X_GITHUB_EVENT']       
     when "pull_request"         
       if @payload["action"] == "closed" && @payload["pull_request"]["merged"]             
       puts "A pull request was merged! A deployment should start now..."              
     end             
  end             
end       
```

接下来做什么？每个 Github 发出的事件会附上一个 HTTP Header `X-Github-Event`。我们现在只需要关心 PR 事件。当 pull request 被 merged（它的状态是 `closed`，并且 `merged` 的值为 `true`），我们将开始部署。       

要测试这个 proof-of-concept，在你测试的库的分支上做些修改，发起一个 pull request 并且 merge。你的服务器会作出相应的响应。                 

## 工作部署  
            
我们的服务已经到位，代码经过审查，同时我们的 pull request 被 merged，我们希望来部署我们的工程。                     

我们开始修改事件监听以在 pull requests 被 merged 时进行处理，然后开始关注下部署：   
           

```
when "pull_request"        
     if @payload["action"] == "closed" && @payload["pull_request"]["merged"]        
     start_deployment(@payload["pull_request"])          
     end         
when "deployment"         
     process_deployment(@payload)              
when "deployment_status"          
     update_deployment_status          
end            
```

基于 pull request 里的信息，我们开始实现 `start_deployment` 方法：  
            
```
def start_deployment(pull_request)
  user = pull_request['user']['login']
  payload = JSON.generate(:environment => 'production', :deploy_user => user)
  @client.create_deployment(pull_request['head']['repo']['full_name'], pull_request['head']['sha'], {:payload => payload, :description => "Deploying my sweet branch"})
end，
```

部署可以附加一些元数据，用一个 `payload` 和一个 `description` 的形式。尽管这些值是可选的，但是有助于我们 log 和展示信息。         

当一个新的部署被创建，一个单独的事件就会触发。这就是为什么我们在 `depolyment` 事件处理中有一个新的 `switch` case。当一个部署已经被触发的时候，你可以使用这些信息来通知。

部署可能会持续很长时间，所以我们需要去监听各种事件，比如什么时候部署被创建，和当前的状态。

让我们模拟一次做了某些工作的部署，并注意它的输出。首先，让我们完成 `process_deployment` 方法：


```
def process_deployment             
     payload = JSON.parse(@payload['payload'])             
     # you can send this information to your chat room, monitor, pager, e.t.c.              
     puts "Processing '#{@payload['description']}' for #{payload['deploy_user']} to #{payload['environment']}"          
     sleep 2 # simulate work            
     @client.create_deployment_status("repos/#{@payload['repository']['full_name']}/deployments/#{@payload['id']}", 'pending')              
     sleep 2 # simulate work              
     @client.create_deployment_status("repos/#{@payload['repository']['full_name']}/deployments/#{@payload['id']}", 'success')            
end           
```

最后，我们会模拟存储状态信息，控制台输出：

```，
def update_deployment_status          
     puts "Deployment status for #{@payload['id']} is #{@payload['state']}"          
end         
```

我们梳理一下发生的事件。一个新的用来负责触发 deoloyment 事件的 deployment 被 `start_delopment` 方法创建，从那里，我们调用 `process_eloyment` 方法去模拟接下来的工作。在处理过程中，我们同样调起 `create_deployment_status` 让接收方知道接下来会发生什么，正如我们设定状态为 `pending`。     

在 deployment 结束后，我们将状态设定为`success`，你会发现，这种模式和你的 CI 状态是相同的。                 

## 结论         

在 Github 中，我们多年来一直使用 [Heaven](https://github.com/atmos/heaven) 的一个版本去管理 deployment，本质上基本步骤和我们上面已经建立的服务端极其相似。在 GitHub， 我们:          

- 在 CI 状态下等待回应               
- 如果代码是绿色，我们 merge pull request          
- Heaven 持有这些 merge 后的代码，并且部署到我们的临时服务器和项目中       
- 同时，Heaven 还通知大家关于构建的消息，通过 Hubot 进入我们的聊天室                          

这就是它了，你不需要自己建立一个 deployment 来使用这个例子，你可以始终依赖于[第三方服务](https://github.com/integrations)。       

