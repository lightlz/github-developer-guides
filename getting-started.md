# 开始

i. 概述  
ii. 身份验证  
iii. 版本库  
iv. Issues  
v. 条件请求（Conditional requests）

让我们来通过一些核心的 API 概念为我们处理一些日常用例。

## 概述

大部分应用都会使用你选择的语言的一个现有 [封装好的库](https://developer.github.com/libraries/),但是先熟悉内部的 API HTTP 方法是很重要的。

通过 [cURL](http://curl.haxx.se/)来进行试验是最简单的。

### Hello World

首先，我们来测试我们的设置是否正确。打开一个命令提示符，输入下面的命令（不需要 $ 符号）

    $ curl https://api.github.com/zen

    Keep it logically awesome.

返回的内容将会是我们的设计理念中随机抽取的一条。

接下来，我们向 [Chris Wanstrath's](https://github.com/defunkt) [GitHub profile](https://developer.github.com/v3/users/#get-a-single-user) 发送一个 GET 请求:

    # GET /users/defunkt
    $ curl https://api.github.com/users/defunkt

    {
      "login": "defunkt",
      "id": 2,
      "url": "https://api.github.com/users/defunkt",
      "html_url": "https://github.com/defunkt",
      ...
    }

Mmmmm，这看起来像 [JSON](http://en.wikipedia.org/wiki/JSON)。让我们添加 `-i` 参数来包含头部信息。

    $ curl -i https://api.github.com/users/defunkt

    HTTP/1.1 200 OK
    Server: GitHub.com
    Date: Sun, 11 Nov 2012 18:43:28 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Status: 200 OK
    ETag: "bfd85cbf23ac0b0c8a29bee02e7117c6"
    X-RateLimit-Limit: 60
    X-RateLimit-Remaining: 57
    X-RateLimit-Reset: 1352660008
    X-GitHub-Media-Type: github.v3
    Vary: Accept
    Cache-Control: public, max-age=60, s-maxage=60
    X-Content-Type-Options: nosniff
    Content-Length: 692
    Last-Modified: Tue, 30 Oct 2012 18:58:42 GMT

    {
      "login": "defunkt",
      "id": 2,
      "url": "https://api.github.com/users/defunkt",
      "html_url": "https://github.com/defunkt",
      ...
    }

在返回的头部信息中有一些有趣的数据。如预期那样，`Content-Type` 属性的值是 `application/json` 。

每一个以 `X-` 开头的头部都是自定义头，它们都不包含于 HTTP 标准。让我们看看其中的几个：

* `x-GitHub-Media-Type` 的值为 `github.v3`。这告诉我们返回值的[媒体类型](https://developer.github.com/v3/media/)。媒体类型帮助在 API v3 中确定我们的输出的版本信息。我们将在后面更深入探讨。

* 注意 `X-RateLimit-Limit` 和 `X-RateLimit-Remaining headers`。这一对头部信息标示出了在一个滚动时间周期内（一般是一个小时）[一个客户端能够发起多少请求](https://developer.github.com/v3/#rate-limiting)和这些请求多少已经完成。

### 身份认证

没认证的客户端每小时可以制造60个请求。如果想制造更多，我们需要进行认证。实际上，使用 GitHub API 做任何有趣一点的事情都会要求认证。

### 基础
GitHub API 最简单的认证方式是使用你的 GitHub 用户名和密码来通过基础认证。

    $ curl -i -u <your_username> https://api.github.com/users/defunkt

    Enter host password for user '<your_username>':

这个 `-u` 参数设置用户名，cURL 会提示你填写密码。你可以使用 `-u "username:password"` 来避免这个提醒，但这会使你的密码被记录在 shell 的历史记录中，所以并不推荐这种做法。验证时，你会看到你的速度限制会升到每小时 5000 个请求，这个会在 `X-RateLimit-Limit` 头部信息中标示。

## 双重认证
如果你启用了[双重认证](https://help.github.com/articles/about-two-factor-authentication)，这个 API 会在以上请求中返回 `401 Unauthorized` 错误码（其他的 API 请求也一样）：

    $ curl -i -u <your_username> https://api.github.com/users/defunkt

    Enter host password for user '<your_username>':

    HTTP/1.1 401 Unauthorized
    X-GitHub-OTP: required; :2fa-type

    {
      "message": "Must specify two-factor authentication OTP code.",
      "documentation_url": "https://developer.github.com/v3/auth#working-with-two-factor-authentication"
    }

避免这个错误最简单的方式是创建一个 OAuth 令牌和使用 OAuth 认证，而不是使用简单的基础认证。在下面的 OAuth 部分可以看到更详细的信息。

### 获取自己配置文件
当认证通过时，你可以利用与 GitHub 账户相关联的权限。例如，尝试获取你的用户配置文件：

    $ curl -i -u <your_username> https://api.github.com/user

    {
      ...
      "plan": {
        "space": 2516582,
        "collaborators": 10,
        "private_repos": 20,
        "name": "medium"
      }
      ...
    }

这一次，除了和早先我们获取 [@defunkt](https://github.com/defunkt) 同样的公共信息集合外，你还将看到你自己用户配置的非公共信息。比如，你将在返回值看到一个 `plan` 对象，它给出了账户的 GitHub 计划的细节。

### OAuth

基本认证虽然方便，但是并不理想，因为你不应该将你的 GitHub 用户名和密码共享给任何人。需要通过 API 读写另一个用户的私有信息时必须使用 OAuth。

OAuth 用令牌（token）替代了用户名和密码。令牌有两大特色：

* 可撤销访问：用户能够在任何时候撤销对第三方 app 的认证。
* 有限访问：用户能够在授权一个第三方 app 之前检验特定的准入权限。

一般来说，令牌能够通过一个 [web 流](https://developer.github.com/v3/oauth/#web-application-flow) 来创建。一个应用将用户跳转到 Github 登陆。然后 GitHub 会显示对话框来标明应用的名字，同时也显示应用曾经被授予的用户权限。当用户授权之后，GitHub 会将用户重定向跳转回应用：

![](images/oauth_prompt.png)

然而，开始使用 OAuth 令牌并不需要你配置整个 web 流。获取一个令牌的简单方法是通过[个人准入令牌设置页](https://github.com/settings/tokens) 创建一个[个人准入令牌](https://help.github.com/articles/creating-an-access-token-for-command-line-use):

![](images/personal_token.png)

同时，[授权 API](https://developer.github.com/v3/oauth_authorizations/#create-a-new-authorization) 使得通过基本授权创建一个 OAuth 令牌变得简单。试试粘贴并运行以下命令:

    $ curl -i -u <your_username> -d '{"scopes": ["repo", "user"], "note": "getting-started"}' \
    https://api.github.com/authorizations

    HTTP/1.1 201 Created
    Location: https://api.github.com/authorizations/2
    Content-Length: 384

    {
      "scopes": [
        "repo",
        "user"
      ],
      "token": "5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4",
      "updated_at": "2012-11-14T14:04:24Z",
      "url": "https://api.github.com/authorizations/2",
      "app": {
        "url": "https://developer.github.com/v3/oauth/#oauth-authorizations-api",
        "name": "GitHub API"
      },
      "created_at": "2012-11-14T14:04:24Z",
      "note_url": null,
      "id": 2,
      "note": "getting-started"
    }

这个小调用里面包含了很多过程，让我们来分解一下。首先，`-d` 标志表明我们正在做一个 `POST` 调用，使用的是 `application/x-www-form-urlencoded`内容类型(和 `GET` 相对)。所有对 GitHub API 发起的 `POST` 请求都必须用 JSON 编写。

接下来，让我们看看我们在这个请求播送的 `scopes` 字段。当创建一个新的令牌时，我们包含了一个可选的数组 _scopes_，或用来标示这个令牌能够获取的信息的准入等级。在这个例子中，我们设置该令牌拥有 `repo` 权限，这个权限将授予用户读写公共和私有仓库的权限；该令牌还有 `user` 域权限，这将授予用户读写公共和私有用户简介数据。查看
[区域文档](https://developer.github.com/v3/oauth/#scopes)可以获得所有区域的列表。为了避免因可能的侵入行为吓到用户，你应该只请求你的应用所实际需要的权限。 `201` 的状态码告诉我们调用是成功的，并且返回的 JSON 包含了新 OAuth 令牌的详细信息。

如果你已经打开[双重授权](https://help.github.com/articles/about-two-factor-authentication),API 将对上述请求返回前面所述的 `401 Unauthorize` 错误码。你能够通过在
[X-GitHub-OTP 请求头](https://developer.github.com/v3/auth/#working-with-two-factor-authentication) 包含一个 2FA OTP 码来回避这个错误:

    $ curl -i -u <your_username> -H "X-GitHub-OTP: <your_2fa_OTP_code>" \
        -d '{"scopes": ["repo", "user"], "note": "getting-started"}' \
        https://api.github.com/authorizations

如果你在一个移动应用打开了 2FA ，可以通过你的手机的一次性密码获取一个 OTP 码。如果你通过短信打开了 2FA，发起此请求后，你将受到一条包含你的 OTP 码的短信。

现在，我们能够在剩下的例子中使用这个40字节的令牌来替代用户名和密码。让我们再次获取我们的个人信息，这一次我们使用 OAuth：

    $ curl -i -H 'Authorization: token 5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4' \
        https://api.github.com/user

**请向对待密码一样对待 OAuth 令牌！**不要和其他用户分享令牌或者将令牌存储在不安全的地方。这些例子中的令牌是伪造的，名字也已经修改过，以免影响无关用户。

现在我们知道了如何进行授权请求，接下来我们来看[Repositories API](https://developer.github.com/v3/repos/)。

## Repositories

几乎所有有意义的 GitHub API 使用会包含了某种程度的 repositories 信息。我们能够像我们前面获取用户详细信息一样获取 repository 详情：

    $ curl -i https://api.github.com/repos/twbs/bootstrap

用同样的方式，我们能够[为授权用户查看 repositories](https://developer.github.com/v3/repos/#list-your-repositories)：

    $ curl -i -H 'Authorization: token 5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4' \
        https://api.github.com/user/repos

或者我们能够[查看另一个用户的 repositories](https://developer.github.com/v3/repos/#list-user-repositories)：

    $ curl -i https://api.github.com/users/technoweenie/repos

或者，我们能够[查看一个组织的 repositories](https://developer.github.com/v3/repos/#list-organization-repositories)：

    $ curl -i https://api.github.com/orgs/mozilla/repos

这些调用返回的信息将依赖于我们怎样被授权的：

* 使用基本授权，返回将包含所有用户在 github.com 能够查看的 repositories。
* 使用 OAuth 的话，如果 OAuth 令牌包含了 `repo` 域，则将只返回私有 repositories。

就像[文档](https://developer.github.com/v3/repos/)中指出的一样，这些方法包含了一个 `type` 参数以便根据用户对 repository 拥有的访问权限类型来过滤返回的 repositories。通过这种方式，我们能够单独获取直接拥有的 repositories，组织的 repositories，或者用户通过一个小组合作的 repositories。

    $ curl -i "https://api.github.com/users/technoweenie/repos?type=owner"

在这个例子中，我们获取了 technoweenie 用户拥有的 repositories，而不是那些他正在和其他人合作的 repositories。注意上面被引号包括的 URL。依赖于你的 shell 设置， cURL 有时候会要求 URL 被引号包括，否则将忽略查询字符串。

### 创建一个 repository

获取一个已存在的 repository 的信息是非常常见的应用，但是 GitHub API 也支持创建新的 repositories。为了 [创建一个 repository](https://developer.github.com/v3/repos/#create) ，我们需要 `POST` 一些包含细节和配置选项的 JSON ：

    $ curl -i -H 'Authorization: token 5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4' \
        -d '{ \
            "name": "blog", \
            "auto_init": true, \
            "private": true, \
            "gitignore_template": "nanoc" \
        }' \
        https://api.github.com/user/repos

在这里最小例子里面，我们为我们的博客（可能是架设在 [GitHub Pages](http://pages.github.com/) ）创建了一个新的 repository。 虽然博客将会是公共的，但是我们将 repository 设置为私有的。同样，在这个请求里，我们用一个 README 文件和一个 nanoc-flavored .gitignore 模板初始化这个 repository。

创建的 repository 将可以在 `https://github.com/<your_username>/blog` 找到。要创建一个你拥有的组织下的 repository，仅需要修改一个 API 方法，将 `/user/repos` 变更为 `/orgs/<org_name>/repos`。

接下来，让我们获取我们新创建的 repository ：

    $ curl -i https://api.github.com/repos/pengwynn/blog

    HTTP/1.1 404 Not Found

    {
        "message": "Not Found"
    }

这是怎么一回事？返回了一个 `404` 错误。因为我们将 repository 设置为私有的，我们需要授权才能查看它。如果你是一个 HTTP 的老用户，你可能会期望一个 `403` 而不是 `404`。但是因为我们不想泄露私有 repositories 的任何信息，所以 GitHub API 在这种情况下返回一个 `404`，表示我们不能确定或者否认这个 repository 的存在。

## 问题 （Issues）

GitHub Issues UI 目标在于当你远离你的方向的时候給你提供适合的工作流。通过 GitHub [Issues API](https://developer.github.com/v3/issues/)，你可以使用其他工具提取数据或者创建 Issues 来給你的团队创建一个工作流。

就像 github.com, 这个 API 給认证用户提供了一些查看 issues 的方法。查看[你的全部 issues](https://developer.github.com/v3/issues/#list-issues)，使用 `GET/issues`:

	$ curl -i -H 'Authorization: token 5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4' \
    https://api.github.com/issues
  
想要获取你[其中一个组织下的 issues](https://developer.github.com/v3/issues/#list-issues)，使用 `GET /orgs/<org>/issues`:

	$ curl -i -H 'Authorization: token 5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4' \
    https://api.github.com/orgs/rails/issues
 
我们也可以获取[单一 repository 中的全部 issues](https://developer.github.com/v3/issues/#list-issues-for-a-repository)：

	$ curl -i https://api.github.com/repos/rails/rails/issues

### 分页

一个项目的 Rails 大小拥有上千个 issues。我们将需要进行[分页](https://developer.github.com/v3/#pagination)处理，使用多种 API 来获取数据。重复最后一条命令，这时候需要注意回复的 headers：

	$ curl -i https://api.github.com/repos/rails/rails/issues

	HTTP/1.1 200 OK

	Link: <https://api.github.com/repos/rails/rails/issues?page=2>; rel="next",
	<https://api.github.com/repos/rails/rails/issues?page=14>; rel="last"


`Link`的[头部信息](http://www.w3.org/wiki/LinkHeader)将提供一个方法来响应链接外部资源，用这种方式来获取额外页面的数据。当我们请求获取超过30条（默认的页面大小）issues 的时候，这个 API 将会告诉我们哪里可以获取下一页和最后一页的结果。

### 创建一个 issue

我们已经知道怎么将 issues 分页，现在让我们[使用 API 来创建一个 issue](https://developer.github.com/v3/issues/#create-an-issue)。

要创建一个 issue，我们需要先通过认证，所以在请求的头部需要传递 OAuth 令牌。同时，我们需要传递 JSON body 中的 title，body 和 labels 到我们想要创建 issue 的 repository 下面的 `/issues` 路径：

	$ curl -i -H 'Authorization: token 5199831f4dd3b79e7c5b7e0ebe75d67aa66e79d4' \
    -d '{ \
         "title": "New logo", \
         "body": "We should have one", \
         "labels": ["design"] \
       }' \
    https://api.github.com/repos/pengwynn/api-sandbox/issues

	HTTP/1.1 201 Created
	Location: https://api.github.com/repos/pengwynn/api-sandbox/issues/17
	X-RateLimit-Limit: 5000

	{
	  "pull_request": {
	    "patch_url": null,
	    "html_url": null,
	    "diff_url": null
	  },
	  "created_at": "2012-11-14T15:25:33Z",
	  "comments": 0,
	  "milestone": null,
	  "title": "New logo",
	  "body": "We should have one",
	  "user": {
	    "login": "pengwynn",
	    "gravatar_id": "7e19cd5486b5d6dc1ef90e671ba52ae0",
	    "avatar_url": "https://secure.gravatar.com/avatar/7e19cd5486b5d6dc1ef90e671ba52ae0?d=https://a248.e.akamai.net/assets.github.com%2Fimages%2Fgravatars%2Fgravatar-user-420.png",
	    "id": 865,
	    "url": "https://api.github.com/users/pengwynn"
	  },
	  "closed_at": null,
	  "updated_at": "2012-11-14T15:25:33Z",
	  "number": 17,
	  "closed_by": null,
	  "html_url": "https://github.com/pengwynn/api-sandbox/issues/17",
	  "labels": [
	    {
	      "color": "ededed",
	      "name": "design",
	      "url": "https://api.github.com/repos/pengwynn/api-sandbox/labels/design"
	    }
	  ],
	  "id": 8356941,
	  "assignee": null,
	  "state": "open",
	  "url": "https://api.github.com/repos/pengwynn/api-sandbox/issues/17"
	}


这个 response 給我们回复了新创建 issue 的一系列指针（pointers），有响应头的`位置`还有 JSON response 的`url` 域。

### 条件请求

作为一个好的api用户很重要的一点就是缓存信息（从来不改变的）以配合api的速度限制。这个 API 提供条件请求和帮你做正确的事。试想一下我们用来获取 defunkt’s 配置的第一条命令：

	$ curl -i https://api.github.com/users/defunkt

	HTTP/1.1 200 OK
	ETag: "bfd85cbf23ac0b0c8a29bee02e7117c6"

除了 JSON body，我们注意 HTTP 的状态码200和 ETag 的头部信息。[ETag](http://en.wikipedia.org/wiki/HTTP_ETag)是 response 的“指纹”。如果我们在随后的访问中传递它，我们就可以告诉 API 让它再一次回复之前的 resource，前提是它已经发生改变。

	$ curl -i -H 'If-None-Match: "bfd85cbf23ac0b0c8a29bee02e7117c6"' \
    https://api.github.com/users/defunkt

	HTTP/1.1 304 Not Modified

状态码304表明这个资源在我们之前访问到现在时间内都没有发生改变，并且这个 response 将不包含 body。作为奖励，这个304响应将不会影响你的速率限制。

喔！现在我们知道了 GitHub API 的基础。

+ 基础 & OAuth 认证
+ Fetching 和 创建 repositories 和 issues
+ 条件请求

继续学习下一个 API 指南[Basics of Authentication](Basics-of-Authentication.md)!
 