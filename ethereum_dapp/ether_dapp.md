# **以太坊 Dapp 调研**
随着加密货币市场的逐渐火热，区块链也从原来的小众玩耍发展为一个新的科技热点。那么截止2017年底，究竟有多少人参与了区块链相关的行业呢？为了一探究竟，我以Dapp为切入点，从 Github 中抓取了所有和以太坊Dapp相关的项目的数据，并做了简单的整理和分析。


# **Dapp 现状**
数据截止 2017年12月20日。

先看看所有Dapp的创建数量分布，横坐标表示新项目的创建时间，纵坐标表示新项目的数量：

![Dapp创建数量分布](https://github.com/heeeeeng/my_docs/blob/master/ethereum_dapp/img/line_createnum.png?raw=true)

从上图中我们可以很明显的发现，在2017年之前每月新增的Dapp数量一直是小于10个的。但是从2017年开始至2017年底，新项目的数量一直在飞速的增长，直到年底才有所缓和。由此可见从2017年开始，越来越多的人参与到区块链行业当中。

那么 Github 上创建的这些项目又有多少是当前还一直活跃的呢？下表展示了Dapp项目的活跃度分布，横坐标为项目的创建时间，纵坐标为项目的最后一次更新时间，方框中的数字表示项目的数量，数字越大则颜色越深：

![Dapp项目活跃度分布](https://github.com/heeeeeng/my_docs/blob/master/ethereum_dapp/img/heat_active_projects.png?raw=true)

图中最上面一行


# **方案实现**

### **方案设计**
- 使用 “Ethereum” 和 “Dapp” 为关键词，在 Github 上搜索相关的项目。
- 用 python 来抓取搜索到的数据。
- 使用 MySQL 来存储获取的数据。
- 用 Echarts 呈现统计后的图表。

## **数据抓取**

本来打算所有数据都用爬虫来获取，但后来发现 Github 官方提供了调取数据的 API，Github 对程序员真是很友好哈。在这里，使用了python库 [github3](https://github.com/sigmavirus24/github3.py) 来调取 API:

```python
from github3 import login, GitHub
import getpass

def main():
    username = "your_username"
    password = getpass.getpass("enter your password: ")
    git = login(username, password)

    search_result = git.search_repositories("ethereum dapp in:name,description ")
```

`search_repositories` 方法返回的是一个包含了所有搜索结果信息的数组。数组的每个元素都可以通过 `as_dict()` 方法转化为 json 格式，json 转化后的元素大概长这样：

```json
...
downloads_url:"https://api.github.com/repos/dapphub/dapp/downloads"
events_url:"https://api.github.com/repos/dapphub/dapp/events"
fork:false
forks:29
forks_count:29
forks_url:"https://api.github.com/repos/dapphub/dapp/forks"
full_name:"dapphub/dapp"
git_commits_url:"https://api.github.com/repos/dapphub/dapp/git/commits{/sha}"
git_refs_url:"https://api.github.com/repos/dapphub/dapp/git/refs{/sha}"
git_tags_url:"https://api.github.com/repos/dapphub/dapp/git/tags{/sha}"
git_url:"git://github.com/dapphub/dapp.git"
...
```
可以看到在这里能直接知道此项目的 name, url, stars 数量等基本信息。（真贴心哈哈哈~）  

但是除了基本信息外，还需要知道项目的 commits 以及 issue 等情况用来分析项目的活跃度。同时，还需要获取项目拥有人的信息，比如拥有者是个人还是组织，拥有者的活跃状况等。在这里我们采用网络爬虫的方式获取这些数据。

具体方法是从上面的 json 数据中获取相关的 url。以获得 commits 数量为例子，我们通过上面的 json 中的 `html_url` 字段获得项目的 Github 主页地址：`https://github.com/dapphub/dapp` 。然后用 python 请求这个 url 从而获得返回的 HTML 文件：
```python
# 简化后的代码
def do_request(url):
    req = requests.get(url)
    req_str = req.text
    req.close()

    return req_str
```
有了 html 文件后，我们就可以通过正则表达式来获取我们需要的数据。为了获得正则表达式的规则，我们先用浏览器打开上面的 url。然后在我们需要的 commits 数据上面点 `右键 -> 检查`。然后浏览器就会帮我们定位到需要的 html 标签上。在这里我们得到的标签内容：

```html
<a data-pjax="" href="/dapphub/dapp/commits/master">
    <svg aria-hidden="true" class="octicon octicon-history" height="16" version="1.1" viewBox="0 0 14 16" width="14"><path fill-rule="evenodd" d="M8 13H6V6h5v2H8v5zM7 1C4.81 1 2.87 2.02 1.59 3.59L0 2v4h4L2.5 4.5C3.55 3.17 5.17 2.3 7 2.3c3.14 0 5.7 2.56 5.7 5.7s-2.56 5.7-5.7 5.7A5.71 5.71 0 0 1 1.3 8c0-.34.03-.67.09-1H.08C.03 7.33 0 7.66 0 8c0 3.86 3.14 7 7 7s7-3.14 7-7-3.14-7-7-7z"></path></svg>
    <span class="num text-emphasized">
        227
    </span>
    commits
</a>
```
这里的 `227` 就是我们想要的，为了匹配上这个数据，我们可以这样写正则表达式：
```python
import re

def reg_helper():
    pattern = '''<span class="num text-emphasized">\s+(?P<flag>[\d|\,]+)\s+</span>\s+commits.*\s+'''

    reg_expr = re.search(pattern, re_str)
    if not reg_expr:
        logger.error("url: https://github.com/dapphub/dapp not match: commits")
        return -1

    # case: "1,567", convert 1,567 to 1567 .
    result_str = reg_expr.group("flag").replace(",", "")
    result_num = int(result_str)

    return result_num
```
这里的 `result_num` 就是我们最后想要的数据了。用同样的方法可以获得其他你想要的数据。

> **注:** 事实上这里完全可以继续使用 Github 的 API 来获得这些数据，那样应该会更方便。具体方法可以查询一下 [github3](https://github.com/sigmavirus24/github3.py) 这个 python 库的可调用方法，或者查询 [Github 官方 API 列表](https://developer.github.com/v3/)。


## **结果统计**

为了更方便做后续统计聚类工作，我们将数据存至 MySQL 中

