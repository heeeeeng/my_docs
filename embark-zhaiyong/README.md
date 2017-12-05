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






