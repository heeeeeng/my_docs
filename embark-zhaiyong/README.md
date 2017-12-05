# Embark 环境搭建

> Truffle和Embark是目前以太坊Dapp开发社区中比较流行的两款开发框架，本文介绍Embark的环境搭建与使用过程。
> 
> 作者：ZY

## **环境要求**
1. 安装geth1.5.8以上版本
1. Node.js 6.9.1以上版本，Node.js的安装一定要注意，不能用apt-get安装。否则安装到的版本过低，会导致Embark无法启动。安装可以采用如下方式：
            
下载Node.js的压缩文件：
``` 
$ wget https://nodejs.org/dist/v8.1.0/node-v8.1.0-linux-x64.tar.xz 
```
            
解压：
```
$ tar -xvf node-v8.1.0-linux-x64.tar.xz
```

将node和npm设置为全局

```
$ sudo ln /home/ubuntu/node-v8.1.0-linux-x64/bin/node /usr/local/bin/node
$ sudo ln /home/ubuntu/node-v8.1.0-linux-x64/bin/npm /usr/local/bin/npm
```

## **IPFS安装启动**
参考 [http://www.8btc.com/ipfs-blockchain](http://www.8btc.com/ipfs-blockchain)，此文章对IPFS的安装过程做了详细描述。
PFS安装完成后启动ipfs:

```
$ ipfs daemon
```
为测试ipfs是否正常运行，我们可以尝试上传一个文件，然后再下载下来对比一下。
首先，编辑一个文件命名为`mytxt.txt`，随便输入一些文字例如：`This is a test for IPFS`.
保存并用ipfs命令添加到节点，如下如所示：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/01.png?raw=true)

可见文件已经被添加到节点，并返回文件并且返回了一个唯一hash值。如果要查看本地节点的文件，可以直接用命令:
```
$ ipfs cat Qmcu71Rbx7q2cJ5YyTApmaLMNg6ureFrymSU9Bn2WFxvJn
```
![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/02.png?raw=true)

此时文件尚未上传到IPFS网络，需要先同步到网络中,先停掉当前节点再重启：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/03.png?raw=true)

此时，文件已经同步到ipfs网络中。打开浏览器，访问地址：`https://ipfs.io/ipfs/Qmcu71Rbx7q2cJ5YyTApmaLMNg6ureFrymSU9Bn2WFxvJn`。可以看到，文件已经能够在ipfs网络访问了。

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/04.png?raw=true)

## **安装Embark**

```
$ npm -g install embark
```

如果要跑以太坊模拟环境，需要安装testrpc

```
$ npm -g install ethereumjs-testrpc
```

执行下列命令,进入demo示例中：

```
$ embark demo
```

embark_demo目录结构如下：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/05.png?raw=true)

目录内配置结构可以参考文件：embark.json

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/06.png?raw=true)

根据配置文件可以知道：
- 智能合约文件位于app/contracts目录下
- app目录下为合约的访问与功能调用页面
- Config目录下为配置文件
- Dist目录下为build后的文件
- test目录下为合约代码对的测试文件，采用Mocha测试框架

智能合约的开发可以在以太坊真实节点上进行：

```
$ embark blockchain
```

如果要在模拟器上进行智能合约开发的话，运行命令启动模拟器：

```
$ embark simulator
```

可选的节点运行方式配置在config/blockchain.json文件中
- development   
- testnet
- livenet
- privatenet

启动并运行livenet示例：

```
$ embark blockchain livenet
$ embark run livenet
```

另开一个command line窗口，启动embark ：

```
$ embark run
```

此命令会直接部署合约，Dapp会被部署到本地服务器上：http://localhost:8000
如果合约代码有变动更新，只需要在页面刷新下，不必重启embark。


进入操作主界面：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/07.png?raw=true)

在浏览器总打开链接：http://localhost:8000 可以查看demo提供的操作页面。因为文档写作时候是在xshell上远程操作，无法打开图形界面。因此采用了w3m命令行了浏览器来查看，如下图所示：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/08.png?raw=true)

网页中可见，demo提供了在合约中设置和获取value的操作，同时提供了在写入文件并上传到ipfs网络，以及ipfs上把文件下载到本地的功能。其中Set Value和Get Value操作对应于合约代码中的函数set和get操作：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/09.png?raw=true)

在前端页面中，可以在js代码中直接调用如下：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/10.png?raw=true)

合约的测试代码为位于test/simple_storage_spec.js中，采用Mocha测试框架：

![](https://github.com/heeeeeng/my_docs/blob/master/embark-zhaiyong/img/11.png?raw=true)

示例代码中分别对storeData和get、set函数进行了测试。到了这一步，我们就可以仿照demo示例进行自己的智能合约代码开发了。

