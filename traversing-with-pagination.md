# 遍历分页页面

* i.	分页基础
* ii.	消费信息
* iii.	构造分页链接

GitHub API 提供了海量的信息给开发者消费。大多数时候，你甚至会发现你自己要求_太多_信息，为了防止我们劳累的服务器出现不测，API 会自动对[要求的数据进行分页](/v3/#pagination)。

在这个指南中，我们会调用 GitHub 搜索 API，并通过分页对结果进行迭代。您可以在 [platform-samples](https://github.com/github/platform-samples/tree/master/api/ruby/traversing-with-pagination) 存储库中找到本次示例工程的完整源代码。

## 分页基础 ##

在你开始之前，有几个关于接收分页数据的重要事实你需要了解：

1.  不同的 API 调用会得到不同的默认返回格式的值。举例来说，调用
    [列出 GitHub 所有公共存储库](https://developer.github.com/v3/repos/#list-all-public-repositories)
    会得到分页之后的数据，每页30个项目；但如果你调用 GitHub Search API，每个分页则会有100个项目。
2.  你可以指定一个页面有多少个项目（上限是100）；但是，
3.  由于技术原因, 不是所有端点都会有一致的结果。举例来说，[events](https://developer.github.com/v3/activity/events/) 就不会让你指定接收页面的最大项目数。所以请务必阅读目标端点的关于如何处理分页结果的文档。

关于分页的信息请参照一次 API 调用的[链接头](http://tools.ietf.org/html/rfc5988)。 例如，我们发一个 curl 请求给搜索 API，来查出 Mozilla 工程一共用了多少次短语 `addClass`：

```
curl -I "https://api.github.com/search/code?q=addClass+user:mozilla"
```

上文中的参数 `-I` 表示我们只关心链接头，不关心具体内容。在我们探讨结果的时候，您将会发现一些链接头内的信息会像下面这样：

	
	Link: <https://api.github.com/search/code?q=addClass+user%3Amozilla&page=2>; rel="next",
	  <https://api.github.com/search/code?q=addClass+user%3Amozilla&page=34>; rel="last"


我们把这个信息分解来看，`rel="next"` 部分说明了下一页是 `page=2`。 这讲得通，毕竟默认情况下，所有分页的请求都会从页面 `1` 开始 。而 `rel="last"` 部分则提供了更多信息，说明了结果的最后一页是在 `34` 页。也就是说，我们还有 33 页关于 `addClass` 的信息可供消费，真爽！

需要注意的是您应该**永远**依赖对方提供给你的链接关系，不要自己试图猜测或者构建 URL，例如[列出一个存储库内的所有 commit](https://developer.github.com/v3/repos/commits/#list-commits-on-a-repository) 中，分页结果是根据 SHA 散列值生成的，而不是页码。

### 在页面中导航 ###

现在你知道了有多少个页面需要接收，下一步可以开始导航页面来观看结果。你可以通过传递 `page` 参数来完成这件事。默认情况下， `page` 参数总是从 `1` 开始。让我们直接跳到第 14 页看看会发生什么事情：

```
curl -I "https://api.github.com/search/code?q=addClass+user:mozilla&page=14"
```

然后我们再一次得到了类似下文的链接头:


	Link: <https://api.github.com/search/code?q=addClass+user%3Amozilla&page=15>; rel="next",
	  <https://api.github.com/search/code?q=addClass+user%3Amozilla&page=34>; rel="last",
	  <https://api.github.com/search/code?q=addClass+user%3Amozilla&page=1>; rel="first",
	  <https://api.github.com/search/code?q=addClass+user%3Amozilla&page=13>; rel="prev"


和预想中的一样，`rel="next"` 指向第 15 页，而 `rel="last"` 则仍然指向第 34 页。不过这次我们得到了一些别的信息：`rel="first"`，提供了_第一页_的 URL；更重要的是，`rel="prev"` 能让你知道上一页的页码。有了这些信息，你就能构造一些用户界面来让用户在只需一次 API 调用的情况下就实现“第一页”、“最后一页”、“上一页”、“下一页”的随意跳转。

### 更改接收的项目数 ###

通过传递 `per_page` 参数，你可以指定每页返回多少个项目了，上限是 100。我们来试试要求关于 `addClass` 的 50 个项目的检索结果：

```
curl -I "https://api.github.com/search/code?q=addClass+user:mozilla&per_page=50"
```

请注意这个语句对链接头回应的影响：

	Link: <https://api.github.com/search/code?q=addClass+user%3Amozilla&per_page=50&page=2>; rel="next",
	  <https://api.github.com/search/code?q=addClass+user%3Amozilla&per_page=50&page=20>; rel="last"


也许您已经猜到，`rel="last"` 信息告诉我们，现在最后一页是第 20 页了。这显然是我们要求在每个页面内容纳更多的信息所导致的。

## 消费信息 ##

您一般不会仅仅为了处理数据分页而费力使用低级的 cURL 调用，所以我们来写一个小小的 Ruby 脚本来完成上文提到的所有事情。
和往常一样，首先我们会需要[GitHub 的 Octokit.rb](https://github.com/octokit/octokit.rb) Ruby 库，还要提供我们的[个人访问令牌](https://help.github.com/articles/creating-an-access-token-for-command-line-use)：


	require 'octokit'
	
	# !!! 在真正的应用内永远不要用硬编码把值写死 !!!
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV['MY_PERSONAL_TOKEN']

接下来，我们利用 Octokit的 `search_code` 方法来执行搜索。和使用 `curl` 不同，用 `search_code` 方法还能立刻得到结果的数量，立刻试试：


	results = client.search_code('addClass user:mozilla')
	total_count = results.total_count

现在我们来获取最后一页的页码。和链接头的 `page=34>; rel="last"` 信息相类似，Octokit.rb 支持通过一个叫做“[超媒体链接关系](https://github.com/octokit/octokit.rb#pagination)”的方法来返回分页信息，我们不会对这个东西作过多解释，不过，可以这么说，在 `results` 变量中的每个元素都有一个叫做 `rels` 的哈希值，这个哈希值可以包含和 `:next`、`:last`、`:first`、`:prev`相关的信息，这取决于你收到的结果类型。这些关系也包含了结果的 URL，称为 `rels[:last].href`。

知道这些之后，我们来获取最后一个结果所在的页码，然后将所有这些信息展现给用户：

	last_response = client.last_response
	number_of_pages = last_response.rels[:last].href.match(/page=(\d+)$/)[1]
	
	puts "There are #{total_count} results, on #{number_of_pages} pages!"

最后，我们来迭代结果。你虽然可以通过一个 `for i in 1..number_of_pages.to_i` 循环来做到这点，不过这次，不妨跟随 `rels[:next]` 头来接收每一页的信息。为了简便起见，这次只获取每一页中的第一个文件的路径。要实现这些需要使用循环，在每个循环的结尾，我们都可以通过跟随 `rels[:next]` 信息来得到下一个页面的数据集，这个循环会一直运行直到再也没有 `rels[:next]` 可供消费为止（换句话说，我们会执行到 `rels[:last]`）。代码应该看上去像下面这样：

	puts last_response.data.items.first.path
	until last_response.rels[:next].nil?
	  last_response = last_response.rels[:next].get
	  puts last_response.data.items.first.path
	end

在 Octokit.rb 中，更改每页显示的项目数量是极端简单的。只需要传递一个 `per_page` 选项 hash 给初始的客户端构造器。除此之外，您的代码应该还是原来的样子：

	require 'octokit'
	
	# !!! 在真正的应用内永远不要用硬编码把值写死 !!!
	# 而是设置环境变量并测试，和下例所示
	client = Octokit::Client.new :access_token => ENV['MY_PERSONAL_TOKEN']
	
	results = client.search_code('addClass user:mozilla', :per_page => 100)
	total_count = results.total_count
	
	last_response = client.last_response
	number_of_pages = last_response.rels[:last].href.match(/page=(\d+)$/)[1]
	
	puts last_response.rels[:last].href
	puts "There are #{total_count} results, on #{number_of_pages} pages!"
	
	puts "And here's the first path for every set"
	
	puts last_response.data.items.first.path
	until last_response.rels[:next].nil?
	  last_response = last_response.rels[:next].get
	  puts last_response.data.items.first.path
	end

## 构造分页链接 ##

正常情况下，当使用分页时，你的目标不是去串联所有可能的结果，而是去制作一套类似下图这样的导航器：

![Sample of pagination links](/images/pagination_sample.png)

我们来勾勒出这个功能的一个微型版本。

从上文提到的代码中，我们已经知道能够在第一次调用得到的分页结果中获取 `number_of_pages` 值：

	require 'octokit'
	
	# !!! DO NOT EVER USE HARD-CODED VALUES IN A REAL APP !!!
	# Instead, set and test environment variables, like below
	client = Octokit::Client.new :access_token => ENV['MY_PERSONAL_TOKEN']
	
	results = client.search_code('addClass user:mozilla')
	total_count = results.total_count
	
	last_response = client.last_response
	number_of_pages = last_response.rels[:last].href.match(/page=(\d+)$/)[1]
	
	puts last_response.rels[:last].href
	puts "There are #{total_count} results, on #{number_of_pages} pages!"

通过下面的代码，我们可以构造一个漂亮的 ASCII 风格数字框：

	numbers = ""
	for i in 1..number_of_pages.to_i
	  numbers << "[#{i}] "
	end
	puts numbers

我们通过随机数来模拟一个用户点击了其中任意一个数字框。

	random_page = Random.new
	random_page = random_page.rand(1..number_of_pages.to_i)
	
	puts "A User appeared, and clicked number #{random_page}!"

我们现在有了页码，可以使用 Octokit 通过传递 `:page` 选项来显式地接收目标页面了：

`clicked_results = client.search_code('addClass user:mozilla', :page => random_page)`

如果想做的更精致一些，我们也可以吧上一页和下一页链接也弄过来。要创建“上一页”(`<<`)和“下一页”(`>>`)元素，可以参考以下代码：

	prev_page_href = client.last_response.rels[:prev] ? client.last_response.rels[:prev].href : "(none)"
	next_page_href = client.last_response.rels[:next] ? client.last_response.rels[:next].href : "(none)"
	
	puts "The prev page link is #{prev_page_href}"
	puts "The next page link is #{next_page_href}"