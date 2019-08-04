整体架构
kafka是我们目前主要使用的消息队列服务，之前一直使用redis队列（基于laravel自带的消息队列服务），但考虑到redis的不稳定性，我们慢慢将队列服务迁移到了kafka. 我们的业务开发语言是php，基于此我们介绍一下我们的kafka架构设计，以及使用过程中需要注意的点，我们还把相关的源代码放在了github上 kafka-client kafka-consumer-proxy

kafka作为一种消息队列服务，我们将消息发给kafka集群，然后通过消费者将消息读取出来，所以说kafka对消息的消费是拉模式(pull)，关于消费拉(pull)或推(push)的模式优劣，kafka官方文档有说明，但是对于我们php业务模块来说，如果由每个业务去开发维护自己的消息消费者，成本较高且容易出错，所以我们开发了消息转发代理(proxy)，将kafka的拉模式转为推模式，proxy将消息从kafka集群读出来，然后将消息转发给相应模块，相应模块只要提供http的接口即可接收消息。整体架构如下图所示：
 我们的proxy通过php实现，即不同队列(topic)使用不同的php进程组进行消费，这样各业务队列的消费是相互隔离的，一个队列消费出问题不影响其它的队列. 可以将我们的proxy理解为集中管理的消费者进程组，同时不直接处理消息，而是将消息向http接口转发出去。
当然，也可以选择即时处理消息而不转发, 应该场景较少。（如处理耗时太长的消息）

消息发送到kafka集群
通过composer的方式引入我们的客户端库后，发送消息很简单，设置队列名，然后发送即可，代码示例如下：
```
KafkaClient::init('127.0.0.1', '/home/work/logs/kafka'); //设置集群地址，及日志地址
KafkaClient::setTopic('topic_name');//设置队列名
KafkaClient::sendMsgAsyn("test msg");//异步发送消息
//or
KafkaClient::sendMsg("test msg");//同步发送消息
```
 
我们发送消息的库使用rdkaka扩展(https://github.com/arnaud-lb/php-rdkafka)，而rdkafka是librdkafka库的php扩展封装（https://github.com/edenhill/librdkafka）。
librdkafka消息发送的实现是将消息放入本地的一个内存队列，然后由另一个线程从内存队列中取出消息并发送到kafka集群。
消息发送如果采用同步模式，即等到消息发送到kafka集群，并得到成功或失败的结果，则我们的php进程会一直查询本地内存队列是不是已经空了，并且当前的消息是不是返回了结果，会导致大量占用CPU， 甚至将php进程卡死。 以我们线上的使用经验，当将socket.blocking.max.ms（可以理解这个值的作用为扩展线程间隔多久轮询一次查看返回结果）的值配置小于50ms时，运行一段时间后，线上的php进程可能因CPU占用过高而卡死而不能响应服务。 而当将socket.blocking.max.ms 的值配置为50ms时则没有出现过这个问题。为什么不将socket.blocking.max.ms 的值调得更大呢，因为这个值也基本等于一个发送请求的耗时值，所以在同步模式下，一个消息发送请求要耗时50ms，这个已经很慢了，50ms是我们找到的在不引起php进程卡死的情况下的最小值。 关于这个问题，之前咨询过librdkafka库的作者，见（https://github.com/edenhill/librdkafka/issues/1553，貌似作者最新回复已经对此问题进行了优化）

所以，如果不是超级超级重要的数据，我们建议使用异步发送模式，把消息放入本地内存之后就返回，响应超级快。

消息代理转发（Proxy)
当一个业务需要处理消息时，首先要准备一个消费消息的http接口，然后提供proxy 的配置文件，最后启动相应消费者进程组即可， proxy会自动转发消息。 proxy配置是一个php文件，如下所示：
```
return [
    'host' => '127.0.0.1', 
    'port' => '80', 
    'url' => '/inner/mqdemo/test',
    'concurrency' => 3,
    'can_skip' => false,
    'retry_nums' => 3, 
    'retry_interval_ms' => 2000, 
    'conn_timeout_ms' => 10000, 
    'exec_timeout_ms' => 60000, 
    'domain' => 'api.dqd.com',
    'use_saved_offset_time' => 0,
    'slow_time_ms' => 1000, 
];
```
<table>
<thead>
<tr>
<th>配置项</th>
<th>示例值</th>
<th>是否必选</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr>
<td>host</td>
<td>127.0.0.1</td>
<td>必选</td>
<td>ip或域名，多个以‘，’ （逗号）分隔</td>
</tr>
<tr>
<td>port</td>
<td>80</td>
<td>必选</td>
<td>端口</td>
</tr>
<tr>
<td>url</td>
<td>/inner/mqdemo/test’</td>
<td>必选</td>
<td>http请求url</td>
</tr>
<tr>
<td>concurrency</td>
<td>3</td>
<td>必选</td>
<td>消息转发并发数，实际等于启动的消费者进程数</td>
</tr>
<tr>
<td>can_skip</td>
<td>false</td>
<td>必选</td>
<td>消息送达出错时能否跳过，不能跳过会一直重试</td>
</tr>
<tr>
<td>retry_nums</td>
<td>3</td>
<td>必选</td>
<td>消息出错能跳过时重试次数</td>
</tr>
<tr>
<td>retry_interval_ms</td>
<td>2000</td>
<td>必选</td>
<td>出错重试间隔</td>
</tr>
<tr>
<td>conn_timeout_ms</td>
<td>10000</td>
<td>必选</td>
<td>http连接超时</td>
</tr>
<tr>
<td>exec_timeout_ms</td>
<td>60000</td>
<td>必选</td>
<td>http执行超时</td>
</tr>
<tr>
<td>use_saved_offset_time</td>
<td>0</td>
<td>必选</td>
<td>kafka服务在出现offset问题时需要重置offset的时间间隔，一般设为0，出问题立即重置offset</td>
</tr>
<tr>
<td>slow_time_ms</td>
<td>0</td>
<td>可选</td>
<td>慢请求阈值，当转发耗时超过该值则会打印相应消息日志</td>
</tr>
<tr>
<td>domain</td>
<td>‘<a href="http://api.dqd.com" rel="nofollow" data-token="058a4965f0eceedb5a135297743faefc">api.dqd.com</a>’</td>
<td>可选</td>
<td>http请求 header host字段</td>
</tr>
<tr>
<td>consumer_conf</td>
<td>[]</td>
<td>可选</td>
<td>kafka配置，参见：<a href="https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md" rel="nofollow" data-token="7fbc296014df9df6ba3f7b1eb9caf2de">https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md</a></td>
</tr>
<tr>
<td>consumer_topic_conf</td>
<td>[‘auto.offset.reset’ =&gt; ‘smallest’]</td>
<td>可选</td>
<td>kafka配置，参见：<a href="https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md" rel="nofollow" data-token="7fbc296014df9df6ba3f7b1eb9caf2de">https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md</a></td>
</tr>
<tr>
<td>async_concurrency</td>
<td>3</td>
<td>可选</td>
<td>同一个消费进程的处理并发数，不保证消息顺序与消息一定投递成功</td>
</tr>
<tr>
<td>cluster</td>
<td>‘kafka_dqd’</td>
<td>可选</td>
<td>集群配置，当有多个kafka集群时，可以进行切换</td>
</tr>
<tr>
<td>message_delivery_guarantees</td>
<td>2</td>
<td>可选</td>
<td>消息投递保证, 可能值为1、2、3 ，见下文解释</td>
</tr>
</tbody>
</table>
 
 
 一些特殊配置的解释
can_skip：
消息转发出错时能否跳过，不能跳过会一直重试，即一般情况下我们是要保证消息一定成功转发的。 但有时我们可能想手动跳过堵住的命令，有相应脚本工具可以实现。

async_concurrency ：
标识转发消息时是同步还是异步，默认是同步，即一个消息必须处理成功后才会处理下一条消息，如果将async_concurrency 值配置为大于0，则会异步并发的转发消息，消息处理速度更快，但不保证每一条消息成功转发，同时也不保证同一个partition中消息的处理顺序

message_delivery_guarantees：
kafka每条消息都会有一个编号（即offset)，每成功消费一条消息，proxy会记录相应的offset到kafka集群（broker)，这样当进程重启时，会知道下一条需要消费的消息起点。
为了防止消息丢失，我们只有在一条消息投递成功时才会将其offset记录到服务器， 但进程可能意外退出，已经成功消费的消息offset没有提交成功，
导致进程重启后重复处理该消息，为了防止这种情况，我们将成功处理的消息offset立即写入本地文件，这样重复到来的消息会被我们过滤掉。
当消费者进程多机部署时，一个队列的消息可能先后由不同的机器的进程消费，写入本地的offset信息不能由另一台机器获得，导致消息可能重复，但只会重复处理一条消息。
对于不同业务队列可以采用不同的消息投递保证：

为了proxy的高可用，我们会多机部署运行proxy, 但在进程意外退出，如人为误杀进程，可能会导致已经成功消费的消息重复消费，但概率极小， 这是我们默认采用的模式， 将 message_delivery_guarantees 的值配置为 2，即消息可能极小概率重复，但不会丢失
在不考虑写本地文件失败的情况下，如果要保证消息的不重复也不丢失，如支付相关的重要数据，我们要单机部署消费者进程，同时将 message_delivery_guarantees 设置为3，即消息不会重复也不会丢失。
每条消息处理完提交offset可能给kafka服务器带来压力，可以间隔一段时间批量提交offset, 将message_delivery_guarantees 的值设为1, 消息可能会丢失，也可能极小概率消息会重复。
--------------------- 
作者：micweaver 
来源：CSDN 
原文：https://blog.csdn.net/MICweaver/article/details/85041252 
版权声明：本文为博主原创文章，转载请附上博文链接！
