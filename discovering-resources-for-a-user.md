# 为用户发现资源

* i.	准备开始
* ii.	发现应用程序可让用户访问的存储库
* iii.	发现应用程序可让用户访问的组织

当发出经过认证的请求给 GitHub API 时，应用程序通常会需要获取当前用户的存储库和组织。这个指南会向您解释如何可靠地发现这类资源。

要和 GitHub API 互动，我们将使用[Octokit.rb](https://github.com/octokit/octokit.rb)。您可以在[platform-samples](https://github.com/github/platform-samples/tree/master/api/ruby/discovering-resources-for-a-user)存储库中找到这个示例项目的完整源代码。

## 准备开始

您应该先阅读[认证基础指南](/basics-of-authentication/)(如果您还没有的话)，再来尝试本指南所提供的示例。下面的示例默认您已经[注册了一个 OAuth 应用程序](/basics-of-authentication/#registering-your-app)并且[该程序有一个为用户提供的 OAuth 令牌](/basics-of-authentication/#making-authenticated-requests)。

## 发现应用程序可让用户访问的存储库

除了拥有自己的个人存储库之外，一个用户还可能是其他用户或组织的存储库的一个合作者。总而言之，刚才提到的都是用户拥有访问特权的存储库，要不就是用户拥有读写权限的私人存储库，要不就是用户拥有写权限的公共存储库。

[OAuth 域](https://developer.github.com/v3/oauth/#scopes)和[组织应用策略](https://developer.github.com/changes/2015-01-19-an-integrators-guide-to-organization-application-policies/)会决定哪些存储库可以通过您的应用让用户访问。使用下面的工作流来发现这些存储库。

和往常一样，我们会 require [GitHub 的 Octokit.rb](https://github.com/octokit/octokit.rb) Ruby 库。然后将其配置为自动为我们处理[分页](https://github.com/v3/#pagination)。


		require 'octokit'
		
		Octokit.auto_paginate = true

接下来，我们选择加入[列出存储库的 API 的未来改进](/v3/repos/#list-your-repositories)存储库。并设置媒体类别让我们得以访问那个功能。

	language-ruby
	Octokit.default_media_type = "application/vnd.github.moondragon+json"

现在，将我们应用程序的[指定用户的 OAuth 令牌](/guides/basics-of-authentication/#making-authenticated-requests)传过去：

	# 在真正的应用内永远不要用硬编码把值写死 !
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV["OAUTH_ACCESS_TOKEN"]

接下来，我们就可以获取[应用程序可以为用户获取的存储库](/v3/repos/#list-your-repositories)了：

	client.repositories.each do |repository|
	  full_name = repository[:full_name]
	  has_push_access = repository[:permissions][:push]
	
	  access_type = if has_push_access
	                  "write"
	                else
	                  "read-only"
	                end
	
	  puts "User has #{access_type} access to #{full_name}."
	end

## 发现应用程序可让用户访问的组织

应用程序可以为用户开展一系列组织相关的任务。要开展这些任务，程序需要一个[OAuth 认证](/v3/oauth/#scopes)和足够的权限。举例来说， `read:org` 域允许您[列出队伍](/v3/orgs/teams/#list-teams)，而 `user` 域则可以让你[公布用户的组织会员身份](/v3/orgs/members/#publicize-a-users-membership)。 当一个用户授予一个或多个这样的域给你的应用程序时，你就可以获取该用户的组织信息了。

就和我们发现存储库一样，我们还是从 require [GitHub’s Octokit.rb](https://github.com/octokit/octokit.rb) Ruby库开始，并将其设置为为我们自动操作[分页](/v3/#pagination)： 

	require 'octokit'
	
	Octokit.auto_paginate = true

接下来，我们会加入[列出组织信息的 API 的未来改进](/v3/orgs/#list-your-organizations)存储库。并设置媒体类别让我们得以访问那个功能。 

	Octokit.default_media_type = "application/vnd.github.moondragon+json"

现在，将我们应用程序的[指定用户的 OAuth 令牌](/guides/basics-of-authentication/#making-authenticated-requests)传过去，来初始化我们的 API 客户端：

	# 在真正的应用内永远不要用硬编码把值写死 !
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV["OAUTH_ACCESS_TOKEN"]


然后，我们可以[列出应用程序可以为用户访问的组织](/v3/orgs/#list-your-organizations)了：

	client.organizations.each do |organization|
	puts "User belongs to the #{organization[:login]} organization."
	end

### 不要依赖公共组织 API

如果你完整阅读了文档，也许会已经发现一个[列出用户公共组织会员身份的 API 方法](/v3/orgs/#list-user-organizations)。大多数应用程序应该避开这个 API 方法。因为这个方法只返回用户的公共组织会员身份，不包括私人组织会员身份。

作为一个应用程序，你通常会希望用户加入的所有组织（包括公共的和私人的）都能授权给你的应用程序访问。而本文所描述的工作流正是让你实现初衷的。