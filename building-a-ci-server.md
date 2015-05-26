# 架设一个 CI 服务器 

* i.	编写您的服务器
* ii.	处理各种状态
* iii.	总结

[状态 API](https://developer.github.com/v3/repos/statuses/) 通过一个测试服务来负责将 commit 捆在一起，所以您作出的每一次 push 都能在 pull request 内测试和表示。

这个指南会用上面提到的 API 来演示如何架设一个能用的服务器。
在这次演示中，我们会：

* 当一个 pull request 被打开时，运行我们的 CI 套件（在此我们会将 CI 状态设置为待定）。
* 当 CI 完成工作时，我们会按情况设置 pull request 的状态。

这里提到的 CI 系统和宿主服务器是虚构的，他们可能是 Travis，或者 Jenkins，也可能是一些完全不同的东西。
这个指南的重点是教您如何架设并且配置负责通讯的服务器。 

请务必[下载 ngrok](https://ngrok.com/) 并且学会[如何使用它](https://developer.github.com/webhooks/configuring/#using-ngrok)，如果您还没有的话。这工具在显示本地连接上非常有用。

注意: 您可以从 [platform-samples 存储库](https://github.com/github/platform-samples/tree/master/api/ruby/building-a-ci-server)这里下载该示例工程的完整源代码。

## 编写您的服务器

我们会编写一个简单的 Sinatra 应用来证明我们的本地连接是可用的。
从这里开始：

	require 'sinatra'
	require 'json'
	
	post '/event_handler' do
	  payload = JSON.parse(params[:payload])
	  "不错, 一切正常!"
	end

（如果您对 Sinatra 的工作原理不甚了解，我们建议您[阅读 Sinatra 官方指南](http://www.sinatrarb.com/)）

现在启动这个服务器。默认情况下，Sinatra 会在端口 `9393` 启动，所以您需要对 ngrok 进行配置，对该端口进行监听。

为了让这个服务器工作，我们需要使用 webhook 来架设一个存储库。 应该将 Webhook 配置成每当一个 pull request 创建或者合并时执行。然后依照您的意愿来架设一个合适的存储库，我们弱弱地建议参考 [@octocat 的 Spoon/Knife 存储库](https://github.com/octocat/Spoon-Knife)。接下来，您应该在您的存储库内创建一个新的 webhook，并交给它 ngrok 给您的 URL：

![A new ngrok URL](images/webhook_sample_url.png)

点击 **Update webhook**（更新 webhook）。您应该会看见一个正文回应：`不错, 一切正常!` 。
很好！接下来点击 **Let me select individual events**（让我选择个别的事件），然后选择以下选项:

* Status （状态）
* Pull Request

只要有一些相关的操作发生，GitHub 就会自动发送一些特别的事件给我们的服务器。我们来更新服务器，暂时让它只处理和 pull request 有关的事件：

	post '/event_handler' do
	  @payload = JSON.parse(params[:payload])
	
	  case request.env['HTTP_X_GITHUB_EVENT']
	  when "pull_request"
	    if @payload["action"] == "opened"
	      process_pull_request(@payload["pull_request"])
	    end
	  end
	end
	
	helpers do
	  def process_pull_request(pull_request)
	    puts "It's #{pull_request['title']}"
	  end
	end

到底发生了什么呢？每一个 GitHub 发出去的事件都会附带一个 `X-GitHub-Event` HTTP头。目前我们只关心 PR 事件，取出信息中的负载(payload)，然后将标题域返回。在理想情况下，除了 pull request 被打开的情况，服务器还应该关心每一次 pull request 的更新，这将会确保每一次新的 push 都会通过 CI 测试。不过在这个示例中我们只会关心当 pull request 打开时的情况。

为了测试这个理论是否可行，您可以对您的测试用存储库的分支做一些变动，然后打开一个 pull request。正常情况下您的服务器会做出合适的回应。

## 处理各种状态 

我们的服务器上线了，接下来可以准备开始第一个必备工作，那就是设置（和更新）CI 状态。请注意，当您每次更新服务器时，可以通过点击 **Redeliver**(重新发送) 来发出相同的负载。没有必要每次做出变更时都发出一个新的 pull request！

由于我们正在操作 GitHub API，所以会一同使用 [Octokit.rb](https://github.com/octokit/octokit.rb) 来管理我们的操作。并且将一个[个人访问令牌](https://help.github.com/articles/creating-an-access-token-for-command-line-use)配置给客户端：

	# 在真正的应用内永远不要用硬编码把值写死 !
	# 而是设置环境变量并测试，和下例所示
	ACCESS_TOKEN = ENV['MY_PERSONAL_TOKEN']
	
	before do
	  @client ||= Octokit::Client.new(:access_token => ACCESS_TOKEN)
	end

在这之后，只要将在 GitHub 上的 pull request 进行更新，就能弄清楚我们正在 CI 上作处理:

	def process_pull_request(pull_request)
	  puts "Processing pull request..."
	  @client.create_status(pull_request['base']['repo']['full_name'], pull_request['head']['sha'], 'pending')
	end

我们在这里做了三件非常基础的事情：

* 查询存储库的全名
* 查询最后一次 pull request 的 SHA 散列值
* 把状态设置为“待定”（pending）

这样就行了！从现在开始，您可以运行任何您需要的进程来执行您的测试用套件。也许您还会想将您的代码传给 Jenkins，亦或是通过相应的 API 来调用另外一个 web 服务，像 [Travis](https://api.travis-ci.org/docs/) 这样。在这之后，您还要确保再一次更新状态，在我们这个示例中，将直接将其设置为 `"success"`：

	def process_pull_request(pull_request)
	  @client.create_status(pull_request['base']['repo']['full_name'], pull_request['head']['sha'], 'pending')
	  sleep 2 # do busy work...
	  @client.create_status(pull_request['base']['repo']['full_name'], pull_request['head']['sha'], 'success')
	  puts "Pull request processed!"
	end

## 总结 

多年来，在 GitHub，我们使用了 [Janky](https://github.com/github/janky) 的其中一个版本来管理 CI。它的运作流程根本上就和上文所架设的服务器是完全相同的。

在 GitHub：

* 每当一个 pull request 被创建或者被更新，都会通过 Janky 发送给 Jenkins
* 然后等待 CI 的状态回应
* 如果代码通过测试，就将其合并入 pull request

这类通信都会完整回传到我们的聊天室。您不需要建立自己的 CI 架构来使用这个示例，因为您在什么时候都可以依赖[第三方服务](https://github.com/integrations)。