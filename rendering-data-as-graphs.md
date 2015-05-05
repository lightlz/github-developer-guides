# 将数据渲染成图表 #

- i.	[设置一个 OAuth 应用程序]()
- ii.	[获取储存库信息]()
- iii.	[可视化编程语言计数]()
- iv.	[结合不同的 API 调用]()

在这个指南中，我们将要用 API 来获取我们拥有的储存库(repository)的信息以及编写它们所用的编程语言。然后，我们将会用 [D3.js](http://d3js.org/) 库把那些信息用几种不同的方式进行可视化。在这里我们用一个极好的 Ruby 库 [Octokit](https://github.com/octokit/octokit.rb) 来和 GitHub API 交流。

如果您还没准备好，你应该先阅读[“认证基础”](../basics-of-authentication/)指南再尝试这个示例。
您能在 [platform-samples](https://github.com/github/platform-samples/tree/master/api/ruby/rendering-data-as-graphs) 存储库中找到这个示例项目的完整源代码。

是时候开始了！

## 设置一个 OAuth 应用程序 ##

首先, 在 GitHub 上[注册一个新应用程序](https://github.com/settings/applications/new)。 设置主 URL 和回调 URL 为 `http://localhost:4567/`。 和[之前](../basics-of-authentication/)一样，我们通过 [sinatra-auth-github](https://github.com/atmos/sinatra_auth_github) 来应用 Rack 中间件来处理 API 的认证：

	require 'sinatra/auth/github'
	
	module Example
	  class MyGraphApp < Sinatra::Base
	    # !!! 在真正的应用内永远不要用硬编码把值写死 !!!
	    # 而是设置环境变量并测试，和下例所示
	    # if ENV['GITHUB_CLIENT_ID'] && ENV['GITHUB_CLIENT_SECRET']
	    #  CLIENT_ID        = ENV['GITHUB_CLIENT_ID']
	    #  CLIENT_SECRET    = ENV['GITHUB_CLIENT_SECRET']
	    # end
	
	    CLIENT_ID = ENV['GH_GRAPH_CLIENT_ID']
	    CLIENT_SECRET = ENV['GH_GRAPH_SECRET_ID']
	
	    enable :sessions
	
	    set :github_options, {
	      :scopes    => "repo",
	      :secret    => CLIENT_SECRET,
	      :client_id => CLIENT_ID,
	      :callback_url => "/"
	    }
	
	    register Sinatra::Auth::Github
	
	    get '/' do
	      if !authenticated?
	        authenticate!
	      else
	        access_token = github_user["token"]
	      end
	    end
	  end
	end



和之前的例子一样，设置一个类似的 _config.ru_ 文件：
	
	language-ruby
	ENV['RACK_ENV'] ||= 'development'
	require "rubygems"
	require "bundler/setup"
	
	require File.expand_path(File.join(File.dirname(__FILE__), 'server'))
	
	run Example::MyGraphApp


## 获取存储库信息 ##

这次，为了能和 GitHub API 交流，我们将用 [Octokit
Ruby 库](https://github.com/octokit/octokit.rb).这要比直接进行一大堆 REST 调用简单许多 ， 更别说 Octokit 就是由 GitHub 用户开发和活跃维护的，所以你能指望这库好用。

通过 API 和 Octokit 库来进行认证是很简单的，只需要将你的登陆信息和令牌(token)作为参数穿给  `Octokit::Client` 构造器即可：

	if !authenticated?
	  authenticate!
	else
	  octokit_client = Octokit::Client.new(:login => github_user.login, :oauth_token => github_user.token)
	end

我们来对我们的存储库信息做点有趣的事情吧，例如查看他们使用的各种编程语言，并且数出哪一种是最常用的。要达成这个目的，首先我们需要通过 API 获取一个存储库列表，用 Octokit 的话语句会是下面这样子：

	repos = client.repositories


接下来，我们会迭代每一个存储库，然后对 GitHub 所关联的编程语言进行计数：

	language_obj = {}
	repos.each do |repo|
	  # 某些情况下，language 可能为0
	  if repo.language
	    if !language_obj[repo.language]
	      language_obj[repo.language] = 1
	    else
	      language_obj[repo.language] += 1
	    end
	  end
	end
	
	languages.to_s

当您重启您的服务器时，网页应该会显示类似下面这样的内容：

	{"JavaScript"=>13, "PHP"=>1, "Perl"=>1, "CoffeeScript"=>2, "Python"=>1, "Java"=>3, "Ruby"=>3, "Go"=>1, "C++"=>1}

到目前为止都十分顺利，不过这样的表现方法对人类不是十分友好。一个可视化图可以很好地帮助我们了解这些语言计数的分布情况。我们把计数值传给 D3 来获得一个整洁的条形图，显示我们使用的编程语言的流行程度。

## 可视化编程语言计数 ##

D3.js，或者直接叫 D3，是一个功能全面的库，专门用于创建许多不同种类的图表和互动可视化图。D3 的详细使用方法已经超出本文范畴，如果你想要一篇不错的入门文章，不妨去看看[“凡人用 D3”](http://www.recursion.org/d3-for-mere-mortals/)。

D3 是一个 JavaScript 库，并且喜欢把数据以数组形式处理。所以，我们先把 Ruby Hash 转换成一个 JSON 数组，以便在浏览器内被 JavaScript 使用。

	languages = []
	language_obj.each do |lang, count|
	  languages.push :language => lang, :count => count
	end
	
	erb :lang_freq, :locals => { :languages => languages.to_json}

我们简单地迭代对象中的每个*键-值对*并将他们写入一个新的数组。之所以在早些时候没做这步，是为了避免在建立 `language_obj` 对象的过程中对其进行迭代。

这时，lang_freq.erb 会需要一些 JavaScript 语句来渲染条形图，在本例中你可以直接使用下方提供的代码，如果你想了解 D3 如何工作，还能同时参照上方的资源链接：

	<!DOCTYPE html>
	<meta charset="utf-8">
	<html>
	  <head>
	    <script src="//cdnjs.cloudflare.com/ajax/libs/d3/3.0.1/d3.v3.min.js"></script>
	    <style>
	    svg {
	      padding: 20px;
	    }
	    rect {
	      fill: #2d578b
	    }
	    text {
	      fill: white;
	    }
	    text.yAxis {
	      font-size: 12px;
	      font-family: Helvetica, sans-serif;
	      fill: black;
	    }
	    </style>
	  </head>
	  <body>
	    <p>Check this sweet data out:</p>
	    <div id="lang_freq"></div>
	
	  </body>
	  <script>
	    var data = <%= languages %>;
	
	    var barWidth = 40;
	    var width = (barWidth + 10) * data.length;
	    var height = 300;
	
	    var x = d3.scale.linear().domain([0, data.length]).range([0, width]);
	    var y = d3.scale.linear().domain([0, d3.max(data, function(datum) { return datum.count; })]).
	      rangeRound([0, height]);
	
	    // 为 DOM 添加 canvas
	    var languageBars = d3.select("#lang_freq").
	      append("svg:svg").
	      attr("width", width).
	      attr("height", height);
	
	    languageBars.selectAll("rect").
	      data(data).
	      enter().
	      append("svg:rect").
	      attr("x", function(datum, index) { return x(index); }).
	      attr("y", function(datum) { return height - y(datum.count); }).
	      attr("height", function(datum) { return y(datum.count); }).
	      attr("width", barWidth);
	
	    languageBars.selectAll("text").
	      data(data).
	      enter().
	      append("svg:text").
	      attr("x", function(datum, index) { return x(index) + barWidth; }).
	      attr("y", function(datum) { return height - y(datum.count); }).
	      attr("dx", -barWidth/2).
	      attr("dy", "1.2em").
	      attr("text-anchor", "middle").
	      text(function(datum) { return datum.count;});
	
	    languageBars.selectAll("text.yAxis").
	      data(data).
	      enter().append("svg:text").
	      attr("x", function(datum, index) { return x(index) + barWidth; }).
	      attr("y", height).
	      attr("dx", -barWidth/2).
	      attr("text-anchor", "middle").
	      text(function(datum) { return datum.language;}).
	      attr("transform", "translate(0, 18)").
	      attr("class", "yAxis");
	  </script>
	</html>


呼！再次地，你不用太担心这堆代码在干什么。重点是位于相对上方的 `var data = <%= languages %>;` 语句，这语句将我们先前创建的 `languages` 数组传入 ERB 让其进行处理。

正如 《凡人用 D3》 指南所说的，这或许不是 D3 的最佳用例，但至少展示了如何结合 Octokit 来使用这个库, 如何来做一些真正炫目的东西。 

## 结合不同的 API 调用 ##

坦白的时候到了：存储库内的 `language` 属性只能识别“主要”的编程语言，这意味着如果你有一个存储库使用了几种不同的编程语言，那么只有占用字节最多的代码所使用的编程语言算数。

我们来结合几种 API 调用来获得一个能显示哪种编程语言拥有最多字节的代码的*真实表示*。[treemap](http://bl.ocks.org/mbostock/4063582) 是可以很好地可视化编程语言占用比例的方法，而不是像上面那样简单的计数。为此我们需要构造类似这样子的一个对象数组：

	[ { "name": "language1", "size": 100},
	  { "name": "language2", "size": 23}
	  ...
	]

因为我们之前已经有了一个存储库列表，所以直接检视每一个存储库，并调用[列出编程语言的 API 方法](https://developer.github.com/v3/repos/#list-languages):

	repos.each do |repo|
	  repo_name = repo.name
	  repo_langs = octokit_client.languages("#{github_user.login}/#{repo_name}")
	end

接着, 在“主表”中累加每个找到的编程语言：

	repo_langs.each do |lang, count|
	  if !language_obj[lang]
	    language_obj[lang] = count
	  else
	    language_obj[lang] += count
	  end
	end

然后, 我们将内容的格式转换成 D3 能理解的结构：

	language_obj.each do |lang, count|
	  language_byte_count.push :name => "#{lang} (#{count})", :count => count
	end
	
	# 一些必须的格式化操作
	language_bytes = [ :name => "language_bytes", :elements => language_byte_count]


(若想获取更多关于 D3 tree map 原理的信息，参见[这个简单的教程](https://developer.github.com/v3/repos/#list-languages)。)

最后, 将这个 JSON 信息传给一样的 ERB 模板：

	erb :lang_freq, :locals => { :languages => languages.to_json, :language_byte_count => language_bytes.to_json}

和之前一样, 这是一段 JavaScript 代码，你可以将其直接复制到你的模板中：

	<div id="byte_freq"></div>
	<script>
	  var language_bytes = <%= language_byte_count %>
	
	  var childrenFunction = function(d){return d.elements};
	  var sizeFunction = function(d){return d.count;};
	  var colorFunction = function(d){return Math.floor(Math.random()*20)};
	  var nameFunction = function(d){return d.name;};
	
	  var color = d3.scale.linear()
	              .domain([0,10,15,20])
	              .range(["grey","green","yellow","red"]);
	
	  drawTreemap(5000, 2000, '#byte_freq', language_bytes, childrenFunction, nameFunction, sizeFunction, colorFunction, color);
	
	  function drawTreemap(height,width,elementSelector,language_bytes,childrenFunction,nameFunction,sizeFunction,colorFunction,colorScale){
	
	      var treemap = d3.layout.treemap()
	          .children(childrenFunction)
	          .size([width,height])
	          .value(sizeFunction);
	
	      var div = d3.select(elementSelector)
	          .append("div")
	          .style("position","relative")
	          .style("width",width + "px")
	          .style("height",height + "px");
	
	      div.data(language_bytes).selectAll("div")
	          .data(function(d){return treemap.nodes(d);})
	          .enter()
	          .append("div")
	          .attr("class","cell")
	          .style("background",function(d){ return colorScale(colorFunction(d));})
	          .call(cell)
	          .text(nameFunction);
	  }
	
	  function cell(){
	      this
	          .style("left",function(d){return d.x + "px";})
	          .style("top",function(d){return d.y + "px";})
	          .style("width",function(d){return d.dx - 1 + "px";})
	          .style("height",function(d){return d.dy - 1 + "px";});
	  }
	</script>

当当当当！一个美丽的矩形包含着你的储存库内各种编程语言，面积均以相对比例显示，一看就懂。为了能适当地显示所有的信息，你可能还需要对你的 treemap 的高度和宽度进行一些调整，这两个参数就是上面的 `drawTreemap` 的参数中的前两个。