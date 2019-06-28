# Libra区块链钱包开发实录附源码 - LearnDapp系列

> Facebook Libra最近很是吸引眼球，作为看好Libra的区块链应用开发者，自然是要尝试着做些什么了。本文记录了开发一个Libra钱包的具体过程，采用RPC调用方案和链做交互。过程描述较为仔细，请视情况跳过已了解的内容。最后附上了Libra钱包源码的Github仓库地址，欢迎clone。

### 1.安装Libra、编译客户端、连接测试网
安装
``` bash
git clone https://github.com/libra/libra.git && cd libra
./scripts/dev_setup.sh
```

编译客户端
``` bash
cargo build
``` 

如果遇到 google/protobuf/wrappers.proto: File not found （macos环境），则在cargo build之前执行
```
export PATH="/usr/local/opt/protobuf/bin:$PATH"
```

看到下图则表示完成，预计5分钟时间
<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/finish.png' style="max-width:88%;"></img></p>



连接测试网
```
./scripts/cli/start_cli_testnet.sh
```

进入交互终端
<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/Interactive.png' style="max-width:88%;"></img></p>


### 2.体验：创建账户、充值、发起交易、查询交易

创建账户
```
account create
```

执行
```
account list
```
列举刚刚创建的两个账号

<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/account-list.png' style="max-width:88%;"></img></p>

给#0账号充值
```
account mint 0 10000
```

然后查询余额
```
query balance 0
```
结果为 `Balance is: 10000`

发起交易

```
transfer 0 1 2
```
之后查询交易
```
query txn_acc_seq 0 0 true
```

返回包括Committed transaction和Events和两部分。通过amount可以看出数额最多保留到小数点后6位，这对于稳定币而言足矣。

此时尝试退出后重新进入，执行account list返回为空，看似数据被清除了。但当你执行account create以后创建的账户还和之前的一样，再查询余额，之前充值的影响还在，其实数据已经上了测试网络。


### 3.本地运行节点

体验过Libra的基本操作后，接下来我们需要自己在本地跑一个认证节点。

```
cargo run -p libra_swarm -- -s 
```
（注意：请提前关闭本地的代理，否则会报错）

如果执行顺利，会和上文执行```./scripts/cli/start_cli_testnet.sh```一样进入交互终端。


可以看到运行在本地的节点，数据是和测试环境独立的，并且退出后数据会重置。


### 4.调用链的API实现


由于Libra提供了rpc调用方式，我们能够很方便的选择语言进行开发。这里我基于nodejs开发了一个npm包`libra-weight`，用于封装rpc方法提供前端调用的基本api。

libra-weight在实现接口前做了这些事：

复制rust源码中的proto文件到项目中，然后执行以下代码：

```
protoc --proto_path=./ --proto_path=/usr/local/Cellar/protobuf/3.7.1/include/ --js_out=import
_style=commonjs,binary:. *.proto
```

每个.proto文件都会得到编译后的*.pb.js：

<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/proto.png' style="max-width:88%;"></img></p>

搜索proto中的request，只实现了这几个接口：

<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/find-request.png' style="max-width:88%;"></img></p>

做了接口接下来就是在钱包应用中调用了，此时就把libra-weight发布完放一边，进入Libra-wallet，代码结构以及调用的实现如下：
<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/libra-wallet.png' style="max-width:88%;"></img></p>



然后前往浏览器中调用接口，就能看到账户的交易信息了
<p align="center"><img src='https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/libra/01/request-res.png' style="max-width:88%;"></img></p>



示例使用了官方测试网络地址，当然你完全可以如前文中所写，在本地自行搭建验证节点，并且运行示例代码直观感受一番。至于钱包前端已经有不少人做了，可以先去 https://github.com/learndapp/awesome-libra#open-source-wallets 看看钱包的部分。


创建账户可以在Libra终端内进行，因为Libra没有挖矿，可以认为充值属于特殊的一种转账交易，也可以在终端中完成。如果你现在就要做到在钱包应用中创建账户，不妨看看这个案例 https://medium.com/kulapofficial/the-first-libra-wallet-poc-building-your-own-wallet-and-apis-3cb578c0bd52 ，当然这种实现方式只是用于演示，创建账户的操作交由他人或经过网络传输都是不安全的。合理的方案是本地环境创建账户+api调用进行转账交易的广播。


**后话**

Libra项目有很多可以探索的地方。比方说用Move编写的mvir后缀文件，如同以太坊Solidity的sol后缀一样，可以称之为Libra中的智能合约。目前在应用端做尝试的也不少，比如区块浏览器，可以去 https://github.com/learndapp/awesome-libra#blockchain-explorers 的区块链浏览器部分查看。接下来我也会做更多实践，有新发现会持续分享。


文中提到的钱包源码：https://github.com/learndapp/Libra-wallet

本文已整理至仓库：https://github.com/learndapp/LearnDapp

记得顺手点个Star，这是对我最好的支持。有任何问题也欢迎随时联系我的微信公众号「区块链瓦工」。


**参考连接**

官方文档：

https://developers.libra.org/docs/my-first-transaction
https://developers.libra.org/docs/move-overview
https://developers.libra.org/docs/crates/ir-to-bytecode
https://developers.libra.org/docs/reference/libra-cli




