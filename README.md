# IBC Demo
这个 Demo 是基于 Cosmos 官方在11月4日会议中展示的一段演示代码，该示例展示了如何使用包含了Cosmos IBC 部分实现的 Cosmos SDK 来进行跨链交易，包括在两条应用链上建立彼此的客户端、建立连接、建立通道、以及发送交易和确认交易的完整跨链流程。从本示例中我们可以窥见 IBC 跨链协议的部分实现和相关细节，完善我们对于跨链的理解。

> 运行完如下流程预计20分钟（不包含 Golang 环境搭建）

[项目地址](https://github.com/cosmos/gaia/blob/cwgoes/ibc-demo-fixes/ibc-demo.md)<br/>
[Cosmos IBC 跨链标准剖析](https://www.notion.so/arthaszeng/Cosmos-IBC-ICS-fad72555b8bc476e820d92cfec88b167)

## Process
**项目依赖**

安装`jq`，`jq` 是一款轻量级的命令行 json 处理工具。
```
brew install jq
```

这个演示分支是存在与 Gaia 项目中的，如果是第一次运行 Gaia，请确保本地 Go 开发环境是可工作的。

拉取代码和编译二进制文件：

```
mkdir -p ${GO_PATH}/github.com/tw-bc-group && cd ${GO_PATH}/github.com/tw-bc-group
git clone git@github.com:tw-bc-group/ibc-demo.git
cd ibc-demo
git checkout master
make install
gaiad version
gaiacli version
```
> `GO_PATH` 是本地的 Go 根目录的环境变量，示例值如：__/Users/cwzeng/.go/src__

**测试网环境设置**

创建临时测试目录，创建测试链 `ibc0` 和 `ibc1`

```
cd ~ && mkdir ibc-testnets && cd ibc-testnets
gaiad testnet -o ibc0 --v 1 --chain-id ibc0 --node-dir-prefix n
gaiad testnet -o ibc1 --v 1 --chain-id ibc1 --node-dir-prefix n
```

**修改默认的 `gaiad` 和 `gaiacli` 配置**

`gaiad`和 `gaiacli` 是我们通过编译之后自动安装到 Go 可执行文件目录下的 Gaia 客户端，其中`gaiad` 是全节点客户端，`gaiacli` 是轻节点客户端。

通过运行如下命令，修改链的默认配置，使得两条链可以在本地同时跑起来：
```
sed -i "_back" 's/"leveldb"/"goleveldb"/g' ibc0/n0/gaiad/config/config.toml
sed -i "_back" 's/"leveldb"/"goleveldb"/g' ibc1/n0/gaiad/config/config.toml
sed -i "_back" 's#"tcp://0.0.0.0:26656"#"tcp://0.0.0.0:26556"#g' ibc1/n0/gaiad/config/config.toml
sed -i "_back" 's#"tcp://0.0.0.0:26657"#"tcp://0.0.0.0:26557"#g' ibc1/n0/gaiad/config/config.toml
sed -i "_back" 's#"localhost:6060"#"localhost:6061"#g' ibc1/n0/gaiad/config/config.toml
sed -i "_back" 's#"tcp://127.0.0.1:26658"#"tcp://127.0.0.1:26558"#g' ibc1/n0/gaiad/config/config.toml
gaiacli config --home ibc0/n0/gaiacli/ chain-id ibc0
gaiacli config --home ibc1/n0/gaiacli/ chain-id ibc1
gaiacli config --home ibc0/n0/gaiacli/ output json
gaiacli config --home ibc1/n0/gaiacli/ output json
gaiacli config --home ibc0/n0/gaiacli/ node http://localhost:26657
gaiacli config --home ibc1/n0/gaiacli/ node http://localhost:26557
```

配置两条链的私钥，使得`n0` 和`n1` 同时出现在两条链上。

```
gaiacli --home ibc1/n0/gaiacli keys delete n0
gaiacli keys test --home ibc0/n0/gaiacli n1 "$(jq -r '.secret' ibc1/n0/gaiacli/key_seed.json)" 12345678
gaiacli keys test --home ibc1/n0/gaiacli n0 "$(jq -r '.secret' ibc0/n0/gaiacli/key_seed.json)" 12345678
gaiacli keys test --home ibc1/n0/gaiacli n1 "$(jq -r '.secret' ibc1/n0/gaiacli/key_seed.json)" 12345678
```

进行检查，确保两条命令输出结果是一样的

```
gaiacli --home ibc0/n0/gaiacli keys list | jq -r '.[].address'
gaiacli --home ibc1/n0/gaiacli keys list | jq -r '.[].address'
```

当私钥配置完毕后，可以启动两条链的全节点，个人建议不要在后台运行，因为会搞忘记关掉，也不便于观察日志🙈

```
gaiad --home ibc0/n0/gaiad start
```

打开一个新的命令行，在与上一个命令行相同的路径下：

```
gaiad --home ibc1/n0/gaiad start
```

到此，两条链`ibc0` 和`ibc1` 就启动好了。接下来，我们需要进行跨链相关的命令操作。

**IBC 操作流程**

**建立客户端**

通过如下命令在两条链上建立彼此的客户端，值得注意的是，创建客户端的过程本身也是一笔交易，既然是交易，那么创建客户端的流程也同样会经过对应链的共识。在共识阶段，可信任的节点会去验证来自与其他链上的交易，在建立链接阶段，他们至少会通过握手协议更新最新的区块信息一次，之后他们会周期性地更新信息。

```
# client for chain ibc1 on chain ibc0
echo -e "12345678\n" | gaiacli --home ibc0/n0/gaiacli \
  tx ibc client create ibconeclient \
  $(gaiacli --home ibc1/n0/gaiacli q ibc client node-state) \
  --from n0 -y -o text

# client for chain ibc0 on chain ibc1
echo -e "12345678\n" | gaiacli --home ibc1/n0/gaiacli \
  tx ibc client create ibczeroclient \
  $(gaiacli --home ibc0/n0/gaiacli q ibc client node-state) \
  --from n1 -y -o text
```

查询详细的客户端状态：

```
gaiacli --home ibc0/n0/gaiacli q ibc client consensus-state ibconeclient --indent
gaiacli --home ibc1/n0/gaiacli q ibc client consensus-state ibczeroclient --indent
```

**建立连接**

为了在两条链之间进行跨链交易，在这之前，必须通过`握手`进行`连接`。一旦连接建立，需要通过另一次握手协议建立针对某一特定应用的`通道` ，使得这一应用特定的数据能在这个通道之内进行传输。典型的例子比如通证转移、跨链验证、跨链账户等等，以及演示项目`ibc-mock`。

通过如下命令建立连接，需要注意的是，在建立连接的过程中，一共会在两条链上广播**7**笔来自与两个钱包的交易。

运行命令的过程中，会要求输入账户的密码，在这里是 12345678。这个过程会耗费一些时间，Cosmos的交易确认时间大概是6-7秒，所以等待时间在代码里被设置为8秒，理论上整个过程的等待时间大约为 7*8 = 56s，约为一分钟。

```
gaiacli \
  --home ibc0/n0/gaiacli \
  tx ibc connection handshake \
  connectionzero ibconeclient $(gaiacli --home ibc1/n0/gaiacli q ibc client path) \
  connectionone ibczeroclient $(gaiacli --home ibc0/n0/gaiacli q ibc client path) \
  --chain-id2 ibc1 \
  --from1 n0 --from2 n1 \
  --node1 tcp://localhost:26657 \
  --node2 tcp://localhost:26557
```

当键入密码后，连接启动，你会看到如下的日志：

```
ibc0 <- connection_open_init    [OK] txid(B41C15A8F31524CB34EE061BA4418F48A3A37A7348BF8F818E67F5EE90AED45F) client(ibconeclient) connection(connectionzero)
ibc1 <- update_client           [OK] txid(CD9F9AFD311DDF6B3604BCDD3DF371FAE93F76A01E66815AD6E2DCDDA942976D) client(ibconeclient)
ibc1 <- connection_open_try     [OK] txid(F364081569703146978605D9079B751FA83E8766C749B1DA96054D2728DBD715) client(ibczeroclient) connection(connectionone)
ibc0 <- update_client           [OK] txid(58A8E012E623303AA2B4A73099D00A6A3F34220E10FA5445251B1F7E6435EFF5) client(ibczeroclient)
ibc0 <- connection_open_ack     [OK] txid(9535B4E25E204C129B91CF2FBDBD3E9AC17881AB4D4BFD4F57731E96D348B053) connection(connectionzero)
ibc1 <- update_client           [OK] txid(50D737D7798E1D0A2E0452B7EDDC2D06A06524424F10B54F6400F148D9255DAF) client(ibconeclient)
ibc0 <- connection_open_confirm [OK] txid(CF0F7E54481D90A438A625E16EAB48F5480334AF090CE05736BDFACB69B8F798) connection(connectionone)
```

连接建立之后，可以运行如下命令查询连接状态：

```
gaiacli --home ibc0/n0/gaiacli q ibc connection end connectionzero --indent --trust-node
gaiacli --home ibc1/n0/gaiacli q ibc connection end connectionone --indent --trust-node
```

**建立通道**

现在两条链上的客户端已经建立连接，现在需要对我们的`ibc-mock` 应用建立`通道` 。通过通道我们可以在链`ibc0` 和 `ibc1` 之间发送数据。

与建立连接类似的，建立通道同样需要在两条链上广播7笔交易，需要大约一分钟的交易确认时间。

```
gaiacli \
  --home ibc0/n0/gaiacli \
  tx ibc channel handshake \
  ibconeclient bank channelzero connectionzero \
  ibczeroclient bank channelone connectionone \
  --node1 tcp://localhost:26657 \
  --node2 tcp://localhost:26557 \
  --chain-id2 ibc1 \
  --from1 n0 --from2 n1
```

会看到如下日志：

```
ibc0 <- channel_open_init       [OK] txid(792E51E0455A8E0C85705C61A638A4D7C5399B3BA5AF6F29C85BB4E090FCA1B7) portid(bank) chanid(channelzero)
ibc1 <- update_client           [OK] txid(CEA961B9BE931E7B06E6D5643486D267677E66A253F104BC00E2BCE1F9343C03) client(ibczeroclient)
ibc1 <- channel_open_try        [OK] txid(D6BC3B03646EF61D1DA153C6678FE047DF76CB12981AC1D524C69C22124967D7) portid(bank) chanid(channelone)
ibc0 <- update_client           [OK] txid(FA22E93601218CEA839FDEB7BD0D8F47D81E1172E18A8B21717675FF5C4BCF40) client(ibconeclient)
ibc0 <- channel_open_ack        [OK] txid(1840343AFB2D5666F52440C199A1356C63A884A959F0A9A53175773CDB83006B) portid(bank) chanid(channelzero)
ibc1 <- update_client           [OK] txid(BBE212C5041AC366C018BB97F8DF8A495562EAFFFF51BFDA7BBAE9952BE589D0) client(ibczeroclient)
ibc1 <- channel_open_confirm    [OK] txid(69F50CA44AE6AD84BD24866E7DB7FE8ADFD9C484171662CB9E6F0C71BFC222A9) portid(bank) chanid(channelone)
```

通道建立之后可以进行状态查询：

```
gaiacli --home ibc0/n0/gaiacli q ibc channel end bank channelzero --indent --trust-node
gaiacli --home ibc1/n0/gaiacli q ibc channel end bank channelone --indent --trust-node
```

**发送数据包**

通道建立之后，我们需要使用`bank` 应用协议发送数据包，需要提供通道的名字和端口号，以及该笔交易的地址和通证信息：

```
gaiacli \
  --home ibc0/n0/gaiacli \
  tx ibc transfer transfer \
  bank channelzero \
  $(gaiacli --home ibc0/n0/gaiacli keys show n1 -a) 1stake \
  --from n0 \
  --source 
```

该笔交易提交之后，会返回在发送的这条链上的块高。在链的日志上，也可以看到该笔交易被包裹进了对应区块。

**接受数据包**

现在，试着查询要发往的`ibc1` 链上的账户信息，当前应该是不存在`1stake` 这个资产的：

```
gaiacli --home ibc1/n0/gaiacli q account $(gaiacli --home ibc0/n0/gaiacli keys show n1 -a) --indent
```

为了完成该笔跨链交易，在目标链上，这笔转账交易需要被确认，通过一下指令来接受转账：

```
gaiacli \
  tx ibc transfer recv-packet \
  bank channelzero ibczeroclient \
  --home ibc1/n0/gaiacli \
  --packet-sequence 1 \
  --timeout 1007 \
  --from n1 \
  --node2 tcp://localhost:26657 \
  --chain-id2 ibc0 \
  --source \
  --indent
```
在目标链上进行“确认收货”的过程本质上也是在该链上广播一笔交易，告诉网络中的节点，“我收到了这笔转账，现在我拥有了这个转过来的资产了”。

现在，我们可以查询我们确实收到了这笔资产了：

```
gaiacli --home ibc1/n0/gaiacli q account $(gaiacli --home ibc0/n0/gaiacli keys show n1 -a) --indent
```
输出的信息会说明有一笔从新建通道里面转移过来的coin。
