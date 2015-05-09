# 使用评论 #

* i.	Pull Request 评论
* ii.	Pull Request 单行评论
* iii.	Commit 评论

对于任何一个 Pull Request，GitHub 都提供三种不同的评论 (comment) 视图：
[针对整个 Pull Request 的评论](https://github.com/octocat/Spoon-Knife/pull/1176#issuecomment-24114792) , 在 Pull Request 内[针对单行代码的评论](https://github.com/octocat/Spoon-Knife/pull/1176#discussion_r6252889) 和 [针对某个具体 commit 的评论](https://github.com/octocat/Spoon-Knife/commit/cbc28e7c8caee26febc8c013b0adfb97a4edd96e#commitcomment-4049848) 。

每种评论都经由不同的 GitHub API 组件发出。在这个指南中，我们会带您探索如何访问并且操纵每一种评论。本文所有示例都会使用在“octocat”存储库中的[这个 Pull Request 样本](https://github.com/octocat/Spoon-Knife/pull/1176)。和往常一样，样本都能在[我们的 platform-samples 存储库](https://github.com/github/platform-samples/tree/master/api/ruby/working-with-comments)中找到。

## Pull Request 评论 ##

为了能访问 Pull Request 上的评论，您需要仔细阅读 [Issue API](https://developer.github.com/v3/issues/comments/)。也许这一开始看起来违反直觉，不过当您一旦了解一个 Pull Request 的本质只是一个带着代码的 Issue 时，使用 Issue API 来创建 Pull Request 上的评论便显得十分合理。

我们将演示如何用 Ruby 脚本 + [Octokit.rb](https://github.com/octokit/octokit.rb) 来获取 Pull Request 评论。您也许还会需要创建一个[个人访问令牌](https://help.github.com/articles/creating-an-access-token-for-command-line-use)。

下述代码能帮您起步，通过 Octokit.rb 来访问 Pull Request 上的评论：
	
	require 'octokit'
	
	# !!! 在真正的应用内永远不要用硬编码把值写死 !!!
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV['MY_PERSONAL_TOKEN']
	
	client.issue_comments("octocat/Spoon-Knife", 1176).each do |comment|
	  username = comment[:user][:login]
	  post_date = comment[:created_at]
	  content = comment[:body]
	
	  puts "#{username} 在 #{post_date} 时发出了一个评论. 内容是:\n'#{content}'\n"
	end

在此，我们特地调用 Issue API 并提供存储库的名称 (`octocat/Spoon-Knife`)和我们感兴趣的 Pull Request 的 ID (`1176`)，来得到评论 (`issue_comments`)。接下来，问题就剩下迭代所有的评论来获取每一个评论的信息内容了。

## Pull Request 单行评论 ##

在 diff 视图内, 您可以就某个在 Pull Request 内的单独的变更来展开讨论，这些评论将会针对在被更改的文件内的特定行。这次讨论的端点  URL 来自于 [Pull Request Review API](https://developer.github.com/v3/pulls/comments/)。

下述代码的功能是根据一个 Pull Request 号码来获取所有在文件上发出的 Pull Request 评论：

	require 'octokit'
	
	# !!! 在真正的应用内永远不要用硬编码把值写死 !!!
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV['MY_PERSONAL_TOKEN']
	
	client.pull_request_comments("octocat/Spoon-Knife", 1176).each do |comment|
	  username = comment[:user][:login]
	  post_date = comment[:created_at]
	  content = comment[:body]
	  path = comment[:path]
	  position = comment[:position]
	
	  puts "#{username} 在 #{post_date} 时发出了一个评论，针对文件 #{path} 的第 #{position} 行. 内容是:\n'#{content}'\n"
	end


您会注意到这代码和上一个示例非常的接近。该示例和 Pull Request 评论示例的差别只在对话的焦点上，一个在 Pull Request 上发出的评论应该用来针对代码的大方向进行讨论或者提出新想法；作为 Pull Request review 的组成部分的单行评论，则应该特别针对一个文件内发生的一个特定改动。 


## Commit 评论 ##

最后一种评论，就是针对每个 commit 发出的。因为如此，我们需要使用 [commit 评论 API](https://developer.github.com/v3/repos/comments/#get-a-single-commit-comment)。
为了获取在 commit 上的评论，您会需要这个 commit 的 SHA1 散列值。换句话说，您不会需要任何和 Pull Request 相关联的识别码。
接下来是示例：

	require 'octokit'
	
	# !!! 在真正的应用内永远不要用硬编码把值写死 !!!
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV['MY_PERSONAL_TOKEN']
	
	client.commit_comments("octocat/Spoon-Knife", "cbc28e7c8caee26febc8c013b0adfb97a4edd96e").each do |comment|
	  username = comment[:user][:login]
	  post_date = comment[:created_at]
	  content = comment[:body]
	
	  puts "#{username} 在 #{post_date} 时发出了一个评论. 内容是:\n'#{content}'\n"
	end

值得注意的是这个 API 调用会同时得到单行评论和针对整个 commit 的评论。