# 学习Filecoin开发一个自己的公链（一）共识 - LearnDapp系列

> 区块链存储一直被认为是最有希望落地的方向，存储的重要性不言而喻，公链普遍存储贵。分片和layer2方案可以提升性能和降低存储成本，但如果以存储作为挖矿基础岂不是一举两得。
> 
> 为什么LearnDapp要讲公链？探索落地应用是W3c.Group的使命，每个Dapp开发者一定会涉及链的调用或者合约编写，有一个自己的公链之后，我们能在需要时自行改参数，也免去了在“水龙头”申请测试网络代币的环节，Dapp开发效率自然提升。


### 一些存储类公链

**Burstcoin**

计算一次（一个称为绘图的过程）并将您的工作结果缓存在硬盘空间上，然后挖掘只需要读取缓存，大部分时间你的硬盘空闲，并且每个块只读几秒钟的绘图文件。当硬盘容量越大，能够获得奖励的命中率越大。除此以外，提交答案的过程也是网速越快越有利。由私钥推导出的Account Id然后结合Shabal256最终完成绘图，以此保证缓存不会被复用从而避免作弊。白皮书附录部分提到了Poc和Poc2，也承认了矿工吐槽说硬盘被浪费而没有起到执行绘图以外的作用，因此官方规划了Poc3或者说Dymaxion，集合了纠缠等各种技术。

**Chia**

基于PoSpace的SpaceMint，使用像PoSpace这样的证明系统，每个矿工都可以立即使用计算一个PoSpace，所以需要指定谁“赢”以及何时继续下一个区块。为每个PoSpace分配一个质量，但要最终确定块，必须增加PoSpace具有VDF（Verifiable Delay Functions 可验证延迟函数）的输出。用VDF指定需要多少个连续计算步骤计算输出，在PoSpace的质量上是线性的，具有最佳质量的PoSpace可以最终确定胜出。


**Storj和Sia**
定位是分布式云存储的产品，实现更经济的存储。文件在上传前会分割加密，并且p2p节点保证数据取回速度。这两个项目的白皮书一长一短也是有趣，就如同他们的去中心化程度一样有差别，相对而言sia做的彻底些，没有注册，没有服务器，不需要中介或信任第三方因为引入了智能合约执行。


**Lambda**

Lambda采用PoST时空证明算法保证数据的存储安全，同时实现了VRF+BFT的共识算法保证共识网络的运转效率与可靠性。Lambda通过交易市场连接存储供应方（矿工）与存储需求方（用户），在链上完成去中心化交易。


**Filecoin**

基于IPFS的设计，也是其从底层去中心化的一个表现。使用Proof-of-Storage共识，包含复制证明（PoRep）和时空证明（PoSt），作用主要有两点：证明矿工做了有效存储；竞争区块打包出块，获取区块奖励。除去硬盘空间大小还有时间（网速快慢）的影响。矿工的算力是他硬盘上存储的数据的大小，存储多少数据在于获取多少订单，而订单数量取决于网络速度快慢，以及报价合不合适。


其他还有genaro等等存储类项目都可以继续了解，这里不展开。在以上项目中，Filecoin的设计目标远大并且也备受关注，出于学习和今后迭代的考虑，接下来我们通过学习Filecoin，完成一个最简单的属于自己的公链。


### PoRep和PoSt

时空证明（PoSt）可以理解为矿工持续性地生成复制证明（PoRep）或者说是数字证明，filecoin的PoRep以及PoSt的数据存储证明是通过FPS模块实现，也就是rust-fil-proofs。

代码结构如下：
![](https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/ldc/01/fps-dependencies.png)

其中filecoin-proofs（实现filecoin存储证明的接口）和storage-proofs（存储证明）都有对应的目录。代码已更新，但文档中的图还是之前的，sector-base和storage-backend这两个部分的文件已经转移，比如原先在sector-base中的sector_class.rs移动到了storage-proofs目录下。


你可能会好奇go-filecoin是如何实现go到rust语言的调用的？

在rust-fil-proofs的READMD中搜索“Go implementation of filecoin-proofs”可以看到有rustverifier.go和rustsectorbuilder.go这2个文件相关，查看go-filecoin中的对应文件发现都引入了libsectorbuilder这个用cgo实现的package。找到cgo_bindings.go中的函数VerifySeal，能发现在rust-fil-proofs的api.rs中有verify_seal对应，不使用cgo的实现可以自行了解。最终，在go-filecoin的miner.go的CommitSector中调用 ctx.Verifier().VerifySeal(req)，VerifySeal就是Verifier接口中的函数实现。


如果想要详细了解各个共识算法，可以参考这篇文章[https://www.chainnews.com/articles/729117267789.htm](https://www.chainnews.com/articles/729117267789.htm)。



### 运行官方实例

根据文档指引，成功运行并参与了replication-game的游戏（Zigzag 10M）

![](https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/ldc/01/game.png)

游戏流程是先在本地build完replication-game，再获取seed，然后在本地选择不同的算法计算出proof结果，最后上传结果到官方服务器参与。
过程中，有两个地方值得关注，一个是生成proof.json，另一个是验证，来看看replication-game的代码：

**1.从game.rs中的这段开始**

```rust
let res = match typ {
    proof::ProofType::DrgPoRep => porep_work(prover, params, seed),
    proof::ProofType::Zigzag => zigzag_work(prover, params, seed),
};
```
找到proofs.rs中能找到对应的porep_work和zigzag_work，依赖了```storage_proofs::drgporep```等在rust-fil-proofs的storage_proofs中的函数，通过```extern crate storage_proofs```标记为外部函数接口。

**2.再看server.rs**

既然是基于rocket这个web框架实现，我们就可以从routes目录开始，找到proof.rs文件，其中有validate方法，调用了```storage_proofs::layered_drgporep```。

以上提到的zigzag_drgporep和layered_drgporep在rust-fil-proofs的storage_proofs目录下被定义为模块，layered_drgporep中的LayerChallenges和ChallengeRequirements分别是数据复制逻辑和复制证明逻辑的实现和结构，详细算法解析的文章可以看看[https://mp.weixin.qq.com/s/Sd6Y0gSX6HB4BFRKPV_0dQ](https://mp.weixin.qq.com/s/Sd6Y0gSX6HB4BFRKPV_0dQ)，代码已更新文章分享的版本较早但也非常值得学习。



## 简单的共识示例

近两年使用Rust语言的区块链项目越来越多，相关的包也逐渐完善。虽说Filecoin使用了golang，但出于方便今后从polkadot/substrate或nervos/ckb等等更多新区块链项目中学习的考虑，我们自己的公链也用Rust实现。若你之前的技术栈中未加入Rust，不妨先看看[https://kaisery.github.io/trpl-zh-cn/ch01-01-installation.html](https://kaisery.github.io/trpl-zh-cn/ch01-01-installation.html)，能够帮助你快速上手。

代码结构如下：

![](https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/ldc/01/ldc.png)

rust-fil-proofs中的storage_proofs直接通过storage-proofs包来调用，共识的生成和验证逻辑就在```proofs/all.rs```下。

执行以下命令可以生成ldc
```bash
cargo +nightly build
```

然后运行 
```bash
./target/debug/ldc --size 1024 --prover lduser zigzag > proof.json
```
可以生成proof.json

![](https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/ldc/01/result-1.png)

最后运行 
```bash
./target/debug/ldc --proof-path ./proof.json proof
```
可以验证结果

![](https://raw.githubusercontent.com/learndapp/LearnDapp/master/topics/ldc/01/result-2.png)


以上过程执行了一次加密到解密验证的过程，体验了PoRep共识的执行，而PoSt可以理解为矿工一定时间内持续地生成复制证明和接受挑战和验证的过程。所以“存力挖矿”的说法相比“硬盘挖矿”更贴切，验证过程涉及到加密计算，并非只是硬盘越大得到的奖励越多。


代码已经上传 [https://github.com/learndapp/LDC](https://github.com/learndapp/LDC)，下一部分将实现p2p网络，明确证明者（prover）和检验者（verifier）以及系统之间的关系。此后也会逐步完善我们的公链，以学习的目的略去一些部分做到最小可执行的程度。

本文作者是W3c.Group社区（[https://w3c.group](https://w3c.group)）的核心共建者。W3c.Group社区目前正在进行只对开发者开放的有奖投票活动，以及有奖发文计划。也欢迎前往W3c.Group的“LearnDapp小组”私信组长和本文作者建立联系。


**参考连接：**

白皮书：

[https://www.burst-coin.org/wp-content/uploads/2017/07/The-Burst-Dymaxion-1.00.pdf](https://www.burst-coin.org/wp-content/uploads/2017/07/The-Burst-Dymaxion-1.00.pdf)    
[https://www.chia.net/assets/ChiaGreenPaper.pdf](https://www.chia.net/assets/ChiaGreenPaper.pdf)  
[https://storj.io/storjv3.pdf](https://storj.io/storjv3.pdf)  
[https://sia.tech/sia.pdf](https://sia.tech/sia.pdf)  
[https://www.lambdastorage.com/doc/Lambda%E9%BB%84%E7%9A%AE%E4%B9%A6-Beta.pdf](https://www.lambdastorage.com/doc/Lambda%E9%BB%84%E7%9A%AE%E4%B9%A6-Beta.pdf)  
[https://filecoin.io/filecoin.pdf](https://filecoin.io/filecoin.pdf)  

代码：

[https://github.com/filecoin-project/go-filecoin](https://github.com/filecoin-project/go-filecoin)  
[https://github.com/filecoin-project/rust-fil-proofs](https://github.com/filecoin-project/rust-fil-proofs)  
[https://github.com/filecoin-project/replication-game](https://github.com/filecoin-project/replication-game)  

文档：

[https://github.com/waynewyang/analysis-of-filecoin-in-Chinese](https://github.com/waynewyang/analysis-of-filecoin-in-Chinese)  
[https://github.com/filecoin-project/go-filecoin/blob/master/CODEWALK.md#sector-builder--proofs](https://github.com/filecoin-project/go-filecoin/blob/master/CODEWALK.md#sector-builder--proofs)  
[https://github.com/filecoin-project/specs/blob/master/proofs.md](https://github.com/filecoin-project/specs/blob/master/proofs.md)  
