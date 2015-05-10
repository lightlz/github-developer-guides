#提供部署

i. 编写你的服务
ii. 工作部署
iii. 结论

[Deployments API](https://developer.github.com/v3/repos/deployments/)提供功能将你的工程发送到你自己的服务器上，就像托管在GitHub一样。　　　　
结合[the Status API](https://developer.github.com/guides/building-a-ci-server/)，you’ll be able to coordinate your deployments the moment your code lands on master.

本指南将为你演示一个你可以操作的步骤，在我们的脚本里，我们会：     

Merge 一个 Pull Request      
当CI结束时, 我们将相应地设置Pull Request的状态。   
当Pull Request被Merge, 我们运行将部署到我们的服务器。    

我们的CI系统和主机服务器将被虚拟成我们的假想，他们可以是Heroku, Amazon或其他一些整体。 本指南的关键就是设置和配置服务端去管理通信。     

如果你还没有准备好，请确定你下载了[ngrok](https://ngrok.com/),并且学会使用它。我们发现他是一个揭露本地连接非常好用的工具。     

提示：你可以从[platform-samples repo](https://github.com/github/platform-samples/tree/master/api/ruby/delivering-deployments)上下载完整的资源。      


##编写你的服务

我们将编写一个快速的应用程序Sinatra证明我们的本地连接正在起作用。我们这样开始：

	require 'sinatra'
	require 'json'

	post '/event_handler' do
  	payload = JSON.parse(params[:payload])
  	"Well, it workded！"
	end

(如果你不熟悉Sinatra如何工作, 我们推荐你阅读[Sinatra指南](http://www.sinatrarb.com/))

服务已经启动，默认情况下，Sinatra在9393端口开始，所以你也会希望配置ngrok去监听它。

为了让这台服务器的工作，我们需要用一个webhook创建一个库。无论何时Pull Resuqest或者merged被创建，webhook都应该被配置为fire。       
继续并创建一个库，你正在沉浸其中。Might we suggest @octocat’s Spoon/Knife repository?，之后，你会在你的repository里创建一        
个新的webhook，将ngrok提供你的URL填充上去。

![image](https://github.com/jikexueyuanwiki/github-developer-guides/blob/master/images/webhook_sample_url.png)           

点击Update webhook。你会看到消息中显示了“Well, it workded!”，不错，点击Let me select individual events.，然后选择下面的选项         

Deployment
Deployment status
Pull Request

当进行相关操作时Github会将这些事件发送到我们的服务器上。我们配置服务器只在Pull Request被merged的时候处理。

post '/event_handler' do
  @payload = JSON.parse(params[:payload])

  case request.env['HTTP_X_GITHUB_EVENT']
  when "pull_request"
    if @payload["action"] == "closed" && @payload["pull_request"]["merged"]
      puts "A pull request was merged! A deployment should start now..."
    end
  end
end

接下来做什么？每个Github发出的时间会附上一个Http头X-Github-Event。我们现在只需要关心PR时间。当pull request被merged（它的状态是closed，并且merged为true），我们将揭开部署。

下面测试这个proof-of-concept，在你的test repository里做些修改，发起一个pull request并且将它merge。你的服务器会作出相应的反应。

##工作部署
我们的服务已经到位，代码经过审查，同时我们的pull request被merged，我们希望来配置我们的工程。     

我们修改时间监听以在pull requests被merged时进行处理，然后开始关注下部署。    

when "pull_request"
  if @payload["action"] == "closed" && @payload["pull_request"]["merged"]
    start_deployment(@payload["pull_request"])
  end
when "deployment"
  process_deployment(@payload)
when "deployment_status"
  update_deployment_status
end

基于pull request里的信息，我们开始实现start_deployment方法：   

def process_deployment
  payload = JSON.parse(@payload['payload'])
  # you can send this information to your chat room, monitor, pager, e.t.c.
  puts "Processing '#{@payload['description']}' for #{payload['deploy_user']} to #{payload['environment']}"
  sleep 2 # simulate work
  @client.create_deployment_status("repos/#{@payload['repository']['full_name']}/deployments/#{@payload['id']}", 'pending')
  sleep 2 # simulate work
  @client.create_deployment_status("repos/#{@payload['repository']['full_name']}/deployments/#{@payload['id']}", 'success')
end

最后，我们会模拟存储状态信息，控制台输出：
def update_deployment_status
  puts "Deployment status for #{@payload['id']} is #{@payload['state']}"
end

我们梳理一下发生的事件，一个新的负责触发deoloyment事件的deployment被start_delopment方法创建，从那里，我们调用process_deloyment方法去模拟接下来的工作。在处理过程中，

##结论

