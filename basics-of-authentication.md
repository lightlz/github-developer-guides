#身份认证基础

在这一节，我们将重点讲身份认证的基础知识。明确地说，我们将使用
[Sinatra](http://www.sinatrarb.com/)创建一个ruby服务，该服务将用几种不
同的方式来实现一个应用的web流程。

你能够从[平台范例仓库](https://github.com/github/platform-samples/tree/master/api/)
下载这个工程的完整源代码

##注册你的应用

首先，你需要
[注册你的应用](https://github.com/settings/applications/new)。每一个已
注册的 OAuth 应用将被指定一个唯一的 Client ID 和 Client Secret 。注意不要共
享你的 Client Secret ！包括将该字符串提交到你的 repo 中。

你能够根据你的喜好任意填写每一个信息，除了__授权回调 URL__ 。它无疑是配置你的应用最重要的部分。它是 Github 在成功认证用户之后返回的回调URL。

因为我们是运行一个普通的 Sinatra 服务，本地实例的地址被设置为`http://localhost:4567`。所以让我们将回调URL填写为`http://localhost:4567/callback`。

##接受用户授权

现在，让我们开始编写我们简单的服务。创建名为 _server.rb_ 的文件并且将以下内容粘贴到文件中：

    require 'sinatra'
    require 'rest-client'
    require 'json'

    CLIENT_ID = ENV('GH_BASIC_CLIENT_ID']
    CLIENT_SECRET = ENV('GH_BASIC_SECRET_ID']

    get '/' do
      erb :index, :locals => {:client_id => CLIENT_ID}
    end

你的 client ID 和 client secret 密匙来自你的
[应用配置页](https://github.com/settings/applications)。你绝不应该将这
些值存储于 Github 或其他公共区域。我们建议将它们保存为
[环境变量](http://en.wikipedia.org/wiki/Environment_variable#Getting_and_setting_environment_variables)
，这也是我们这里所做的。

接下来，在 _views/index.erb_ 中，粘贴以下内容：

    <html>
      <head>
      </head>
      <body>
        <p>
          Well, hello there!
        </p>
        <p>
          We're going to now talk to the GitHub API. Ready?
          <a href="https://github.com/login/oauth/authorize?scope=user:email&client_id=<%= client_id %>">Click here</a> to begin!</a>
        </p>
        <p>
          If that link doesn't work, remember to provide your own <a href="/v3/oauth/#web-application-flow">Client ID</a>!
        </p>
      </body>
    </html>

（如果你不熟悉 Sinatra 是如何工作的，我们建议
[阅读 Sinatra 指南](https://github.com/sinatra/sinatra-book/blob/master/book/Introduction.markdown#hello-world-application)
）

同样的，注意代码中的 URL 使用 `scope` 查询参数来定义应用程序所要求的
[权限区域](https://developer.github.com/v3/oauth/#scopes)(scopes)。对于我
们的应用，我们请求`user:email`权限区域来读取私有email地址。

在你的浏览器中打开`http://localhost:4567`。点击该链接后，你将跳转至 GitHub ，并且显示类似以下对话框：

![](/images/oauth_prompt.png)

如果你信任你自己，点击 __Authorize App__ 。哇哦， Sinatra 跳出来一个`404`错误。这是怎么回事？

好吧，记得我们指定了一个回调 URL 为`callback`吗？我们并没有为它提供路由，所以 GitHub 在验证 app 之后，不知道把用户往哪里丢。现在我们来解决这个问题！

###提供一个 callback

    在 _server.rb_ 中，加入一个 route 来指明 callback 应该做什么：

    get '/callback' do
      # get temporary GitHub code...
      session_code = request.env['rack.request.query_hash']['code']

      # ... and POST it back to GitHub
      result = RestClient.post('https://github.com/login/oauth/access_token',
                              {:client_id => CLIENT_ID,
                               :client_secret => CLIENT_SECRET,
                               :code => session_code},
                               :accept => :json)

      # extract the token and granted scopes
      access_token = JSON.parse(result)['access_token']
    end

在一次成功的 app 授权认证之后， GitHub 提供了一个临时的`code`值。你将需要
将这个值`POST`回 GitHub 以交换一个`access_token`。我们使用
[rest-client](https://github.com/archiloque/rest-client)来简化我们的
 GET 和 POST HTTP 请求。注意，你可能永远不会通过 REST 来访问这些 API 。对于一
个更加正式的应用，你很可能使用
[一个你选择的语言所写的库](https://developer.github.com/libraries/)。

###确认被授予的权限区域

在此之后，用户将能够
[编辑你请求的权限区域](https://developer.github.com/changes/2013-10-04-oauth-changes-coming/)
，你的应用也可能被授予少于你默认请求的数量的权限区域。所以，在你使
用该 token 进行任何请求前，你应该确定用户授予了该 token 哪些权限区域。

被授予的权限区域被作为交换token时返回值的一部分被返回。

    # check if we were granted user:email scope
    scopes = JSON.parse(result)['scope'].split(',')
    has_user_email_scope = scopes.include? 'user:email'

在我们的应用中，我们使用`scopes.include?`来检查我们是否被授予了
`user:email`区域的权限，我们需要使用该权限来获取授权用户的私人 email 地
址。如果应用程序要求更多其他区域的权限，我们也可以以同样的方式检查。

还有，因为权限区域之间有着继承的关系，你必须检查你被授予了所请求的最低
级别的权限。比如说，如果应用请求了`user`区域权限，但是它可能只被授予了
`user:email`区域权限。在这种情况下，应用将不会被授予它所请求的权限，但
是已经被授予的区域权限仍然是有效的。

仅在进行请求之前检查区域授权情况是不够的，因为用户可能在你检查授权情况
和进行实际请求之间改变了区域权限。如果这种情况发生了，原本你预计成功的
请求，可能会失败并返回一个`404`或`401`状态，或者返回一个不同的信息子集。

为了让你能够更优雅地处理这些情况，所有有效 token 发起的 API 请求的返回值都
包含一个`X-OAuth-Scopes`头部。这个头部包含了该 token 用来发起请求的区域
列表。除此之外，授权 API 还提供了一个终端来
[检查一个token的有效性](https://developer.github.com/v3/oauth_authorizations/#check-an-authorization)
。使用这个信息来检测token授权区域的改变，并且告知你的用户可用应用功能
的改变。

###发起已认证请求

最后，使用这个 access token ，你将能够作为一个已登录用户发起已认证的请求。

    # fetch user information
    auth_result = JSON.parse(RestClient.get('https://api.github.com/user',
                                            {:params => {:access_token => access_token}}))

    # if the user authorized it, fetch private emails
    if has_user_email_scope
      auth_result['private_emails'] =
        JSON.parse(RestClient.get('https://api.github.com/user/emails',
                                  {:params => {:access_token => access_token}}))

    erb :basic, :locals => auth_result

我们能够使用我们的结果做任何我们想要的事。在这里，我们仅仅是将它们直接输出到 _basic.erb_ 中：

    <p>Hello, <%= login %>!</p>
    <p>
      <% if !email.nil? && !email.empty? %> It looks like your public email address is <%= email %>.
      <% else %> It looks like you don't have a public email. That's cool.
      <% end %>
    </p>
    <p>
      <% if defined? private_emails %>
      With your permission, we were also able to dig up your private email addresses:
      <%= private_emails.map{ |private_email_address| private_email_address["email"] }.join(', ') %>
      <% else %>
      Also, you're a bit secretive about your private email addresses.
      <% end %>
    </p>

##实现持久授权

如果我们要求用户每次进入网页的时候都需要登录 app ，那是非常糟糕的。例如，尝试直接打开`http://localhost:4567/basic`。你将看到一个报错。

假如我们能够跳过整个“点击这里”的过程，而是仅仅_记住_它，只要用户登录了 GitHub ，它们就能够使用这个应用，那会怎样？请保持淡定，因为这就是接下来我们要做的。

我们上面缩写的小服务器是非常简单的。为了能够嵌入一些智能的认证机制，我们将转而使用回话来保存 token 。这将使得认证对用户来说是透明的。

另外，因为我们要在会话中保持授权区域，我们需要处理用户在我们检查之后更
新了区域，或者撤消了标识的情况。为了做到这一点，我们将使用一个
`rescue`区块并检查第一个成功的API调用，这确认了 token 还是有效的。然后我
们会检查`X-OAuth-Scopes`应答头来确认用户还没有撤消`user:email`区域。

创建一个名为 _advanced_server.rb_ 的文件，并且将下面的代码粘贴到其中：

    require 'sinatra'
    require 'rest_client'
    require 'json'

    # !!! DO NOT EVER USE HARD-CODED VALUES IN A REAL APP !!!
    # Instead, set and test environment variables, like below
    # if ENV['GITHUB_CLIENT_ID'] && ENV['GITHUB_CLIENT_SECRET']
    #  CLIENT_ID        = ENV['GITHUB_CLIENT_ID']
    #  CLIENT_SECRET    = ENV['GITHUB_CLIENT_SECRET']
    # end

    CLIENT_ID = ENV['GH_BASIC_CLIENT_ID']
    CLIENT_SECRET = ENV['GH_BASIC_SECRET_ID']

    use Rack::Session::Pool, :cookie_only => false

    def authenticated?
      session[:access_token]
    end

    def authenticate!
      erb :index, :locals => {:client_id => CLIENT_ID}
    end

    get '/' do
      if !authenticated?
        authenticate!
      else
        access_token = session[:access_token]
        scopes = []

        begin
          auth_result = RestClient.get('https://api.github.com/user',
                                       {:params => {:access_token => access_token},
                                        :accept => :json})
        rescue => e
          # request didn't succeed because the token was revoked so we
          # invalidate the token stored in the session and render the
          # index page so that the user can start the OAuth flow again

          session[:access_token] = nil
          return authenticate!
        end

        # the request succeeded, so we check the list of current scopes
        if auth_result.headers.include? :x_oauth_scopes
          scopes = auth_result.headers[:x_oauth_scopes].split(', ')
        end

        auth_result = JSON.parse(auth_result)

        if scopes.include? 'user:email'
          auth_result['private_emails'] =
            JSON.parse(RestClient.get('https://api.github.com/user/emails',
                           {:params => {:access_token => access_token},
                            :accept => :json}))
        end

        erb :advanced, :locals => auth_result
      end
    end

    get '/callback' do
      session_code = request.env['rack.request.query_hash']['code']

      result = RestClient.post('https://github.com/login/oauth/access_token',
                              {:client_id => CLIENT_ID,
                               :client_secret => CLIENT_SECRET,
                               :code => session_code},
                               :accept => :json)

      session[:access_token] = JSON.parse(result)['access_token']

      redirect '/'
    end

大部分的代码看起来都很熟悉。比如说，我们仍然使用`RestClient.get`来调用
 GitHub API ，并且我们仍然将我们的结果传给一个ERB模板来渲染。（这回，文
件名是`advanced.erb`）

而且，我们现在使用`authenticated?`方法来检查用户是否已经认证过了。如果
没有，`authenticate!`方法将被调用，这个方法将执行OAuth流程并且使用被授
予的标识和区域来更新回话。

接下来，在 _views_ 中创建一个文件 _advanced.erb_ ，并将以下内容粘贴进去：

    <html>
      <head>
      </head>
      <body>
        <p>Well, well, well, <%= login %>!</p>
        <p>
          <% if !email.empty? %> It looks like your public email address is <%= email %>.
          <% else %> It looks like you don't have a public email. That's cool.
          <% end %>
        </p>
        <p>
          <% if defined? private_emails %>
          With your permission, we were also able to dig up your private email addresses:
          <%= private_emails.map{ |private_email_address| private_email_address["email"] }.join(', ') %>
          <% else %>
          Also, you're a bit secretive about your private email addresses.
          <% end %>
        </p>
      </body>
    </html>

从命令行调用`ruby advanced_server.rb`，将在 4567 端口启动你的服务端，和
我们使用简单的 Sinatra app 时同样的端口。当你浏览
`http://localhost:4567`时， app 调用`authenticate!`将你重定向到
`/callback`。然后`/callback`将我们又送回了`/`，由于现在我们已经认证了，
页面将渲染 _advanced.erb_ 。

我们能够通过在 GitHub 将我们的回调 URL 指定为`/`来完全简化这个往返的过程。
但是，因为`server.rb`和`advanced.rb`都依赖于同一个回调 URL ，我们必须多
绕点弯来让它正确工作。

而且，如果我们从来没有授权这个应用去活取我们的GitHub数据，我们将从更早的弹出窗口看到相同的确认对话框和警告。

如果你有兴趣，你可以查看
[yet another Sinatra-GitHub auth example](https://github.com/atmos/sinatra-auth-github-test)
作为另一个工程进行实验。
