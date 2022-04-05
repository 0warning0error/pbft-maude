# 用maude验证pbft协议
建模参考论文 ：https://pmg.csail.mit.edu/papers/osdi99.pdf 。

## 简化处理
- 只做正常情况下的
- 忽略对消息摘要和签名的验证，直接变成判断两个消息是否相等。（以后也会是这样，因为验证消息签名的目的是验证消息是否被篡改，建模的时候可以直接加以限制即可）
- 忽略序号n必须在水线（watermark）上下限h和H之间。
- 忽略时间对节点的影响（对节点而言相当于瞬时完成收发消息）
- CLIENT的request的操作是写入一个数，replica将操作附加到数组最后面

## 正常情况(Normal Case)下的步骤
- 初试阶段，CLIENT处于request-start（请求开始）状态（因为需要发送消息），PRIMARY处于request-start状态，BACKUP处于pre-prepare状态
（因为要等待PRIMARY会直接发送pre-prepare消息）
- 首先，CLIENT给PRIMARY发送请求消息，其中有请求的操作（在这个模型里被简化成对current-num的写入）、时间戳（这个暂时被忽略掉了）
此时client进入等待回复(wait-for-reply(等待时间))状态 ：对应CLIENT-EX中的snd-request规则。
- PRIMARY收到CLIENT的request消息后，给这个请求添加序号，然后向其他节点发送pre-prepare消息。PRIMARY此时对此序号进入prepare状态
- 一个backup-i收到pre-prepare(v, n, d)消息后的操作
  - 检查条件判断是否接收：
    - 请求和pre-prepare的签名正确，d是m的摘要
    - v是当前view
    - n在水线范围内
	- 没有接收过pre-prepare(v,n,d')，其中d' ≠ d
  - 如果成功接收pre-prepare(v, n, d)，那么该backup-i 进入prepare(d)阶段，广播prepare(v, n, d, i)消息给其他所有replica。
- 处于prepare(seq)阶段的replica此时等待接收prepare(v, n, d, i)消息(接收条件类似上面的i，ii，iii）并放入log中…

如果一满足prepared(v, n, d, i)预测器后，replica-i进入commit(d)阶段，广播**commit(v, n, d, i)消息** （并且将其放入log中）

replica-i 满足prepared(m, v, n, i)谓词的条件：
  - log中有**m**
  - log中有**m**对应的**pre-prepare(v, n, d)消息**
  - log中至少有2f条与**pre-prepare(v, n, d)消息**相匹配的来自不同backup的**prepare消息.** (相匹配指：它们的v，n，d相同）

- 处于commit(d)阶段的replica此时等待接收**commit(v, n, d, i)消息**(接收条件类似上面的i，ii，iii）并放入log中…

一满足commit-local(m, v, n, i)谓词，replica-i开始执行m请求的命令，对序号为n进入complete(n)状态，返回携带replica-i执行命令后的状态的**reply消息**给client

replica-i 满足commit-local(m, v, n, i)谓词的条件：
  - 满足prepare(m, v, n, i)谓词
  - log中至少有2f+1条与**pre-prepare(v, n, d)消息**相匹配的来自不同replica的**commit消息.** (相匹配指：它们的v，n，d相同）
- client至少接收f+1条来自不同replica的、signature正确的、匹配同一个request的、结果相同的reply消息后，接受该结果。

## 目前未完善的点
- 目前没对时间进行建模，primary对请求来者不拒。
- 忽略了生成和验证digest的密码学技术。对消息摘要，简单判断消息和原来保存的消息是否相等。
- 判断是否接收prepare消息的4个条件中，仅考虑了是否为同一view这一条件。
- 节点的Byzantine behavior未建模。