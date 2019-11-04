# 性能调优指南

#### 调整Jaeger实例以获得更好的性能

Jaeger是从第一天开始构建的,能够以弹性方式摄取大量数据。
为了更好地利用可能导致延迟的资源(例如存储或网络通信),Jaeger会对数据进行缓冲和批处理。
如果生成的跨度超过Jaeger能够安全处理的范围,则跨度可能会丢失。
但是,默认值可能并不适合所有情况:例如,作为Sidecar运行的Agent可能比在裸机中作为守护程序运行的Agent具有更多的内存约束。

##部署注意事项

尽管单个组件的性能调整很重要,但是Jaeger的部署方式对于获得最佳性能可能是决定性的。

###上下缩放收集器

使用平台的自动缩放功能:收集器几乎可以水平扩展,因此可以按需添加和删除更多实例。
放大和缩小的一种好方法是检查`jaeger_collector_queue_length`度量标准:在长度超过最大大小的50％的时间较长的情况下添加实例。
可以考虑的另一个指标是`jaeger_collector_in_queue_latency_bucket`,这是一个直方图,
指示在跨接队列之前,工人等待了多长时间。当队列延迟随着时间的推移而变高时,这是增加工作人员数量或提高存储性能的一个很好的指示。

当您的平台提供自动扩展功能,或者启动/停止Collector实例比更改现有的正在运行的实例更容易时,建议添加Collector实例。
当CPU使用量应分布在各个节点上时,还将指示水平缩放。

###确保存储可以跟上

收集器使用一个工作程序将每个范围写入存储,直到阻塞为止。
当存储太慢时,存储阻止的工作人员数量可能会过多,从而导致跨度下降。
为了帮助诊断这种情况,可以分析直方图`jaeger_collector_save_latency_bucket`。
理想情况下,延迟应随时间保持不变。
当直方图显示大多数跨度随着时间的推移越来越长时,这很好地表明您的存储可能需要注意。

###将代理靠近您的应用程序

代理应与检测的应用程序放置在同一主机上,以避免通过网络丢失UDP数据包。
通常,这是通过在传统应用中或每个裸机主机在Kubernetes等容器环境中作为sidecar来实现的,
因为这有助于分散由Ag​​ent处理的负载,并具有允许单独调整每个Agent的附加优势,这通常可以实现。应用程序的需求和重要性。

###考虑使用Apache Kafka作为中间缓冲区

Jaeger[可以使用Apache Kafka](./architecture.md)作为收集器和实际后备存储(Elasticsearch,Apache Cassandra)之间的缓冲区。
对于流量高峰相对频繁(黄金时间流量)但一旦流量正常化,存储最终可以追上的情况,这是理想的选择。
为此,应在收集器中将环境变量SPAN_STORAGE_TYPE设置为kafka,并且可以使用Jaeger Ingester组件,从Kafka读取数据并将其写入存储。

除性能方面外,将跨度写入Kafka对于构建实时数据管道以进行聚合和从迹线提取特征很有用。

##客户端(跟踪器)设置

Jaeger客户端的构建对已检测的应用程序影响最小。因此,它具有保守的默认值,可能并不适合所有情况。
请注意,可以通过编程方式或通过[环境变量](./client-features.md)配置Jaeger客户端。

###调整采样配置

`JAEGER_SAMPLER_TYPE``和`JAEGER_SAMPLER_PARAM`一起指定应该对跟踪进行"采样"的频率,
即记录并发送到Jaeger后端。对于生成大量跨度的应用程序,将采样类型设置为"概率",并将值设置为`0.001`(默认值)
将导致以1/1000的机会报告跟踪。请注意,采样决策是在根跨度上做出的,并向下传播到所有子跨度。

对于低到中等流量的应用程序,将采样类型设置为`const`,将值设置为`1`将导致报告所有跨度。类似地,可以通过将值设置为`0`来禁用跟踪,
而上下文传播将继续起作用。

一些客户端支持设置`JAEGER_DISABLED`以完全禁用Jaeger Tracer。
仅当Tracer的行为方式对所检测的应用程序造成问题时才建议这样做,因为它不会将上下文传播到下游服务。

> 我们推荐建议您设置客户使用[`remote`采样策略](./sampling.md#收集器采样配置),以便管理员可以集中设置每种服务的具体采样策略。

###增加内存队列大小

大多数Jaeger客户端(例如Java,Go和C＃客户端)都会在将内存跨度发送到Jaeger代理/收集器之前对其进行缓冲
。该缓冲区的最大大小由环境变量`JAEGER_REPORTER_MAX_QUEUE_SIZE`(默认值:大约`100` spans)定义:大小越大,潜在的内存消耗就越大。
当检测到的应用程序生成大量跨度时,队列可能已满,
导致客户端丢弃新的跨度(度量`jaeger_tracer_reporter_spans_total{result ="dropped",}`)。

在最常见的情况下,队列将接近空(度量标准:`jaeger_tracer_reporter_queue_length`),
因为会定期或在达到一定大小的批处理时将跨度刷新到代理或收集器。
此队列的详细行为在此[GitHub问题](https://github.com/jaegertracing/jaeger-client-java/issues/607)中进行了描述。

###修改批处理范围刷新间隔

Java,Go,NodeJS,Python和C＃客户端允许自定义由报告者(例如`RemoteReporter`)用来触发`flush`操作的刷新间隔(默认值:" 1000"毫秒或1秒).
将所有内存跨区发送到代理或收集器。刷新间隔设置得越小,刷新操作发生的频率就越高。
由于大多数报告程序将等待直到队列中有足够的数据,因此此设置将强制以定期间隔进行刷新操作,以便将跨度及时发送到后端。

当检测到的应用程序正在生成大量跨度并且代理程序/收集器靠近该应用程序时,网络开销可能会很低,因此需要进行更多的刷新操作。
当使用HttpSender且Collector与应用程序的距离不够时,网络开销可能会太高,因此使此属性的值更高是有意义的。

##代理设置

Jaeger代理从客户端接收数据,然后将其批量发送到收集器。如果配置不正确,即使主机具有大量资源,它也可能最终会丢弃数据。

###调整服务器队列大小

"服务器队列大小"属性集(`processor.jaeger-binary.server-queue-size`,`processor.jaeger-compact.server-queue-size`,
`processor.zipkin-compact.server-queue-size`)表示代理可以接受并存储在内存中的最大跨度批处理数。
可以肯定地说,`jaeger-compact`是代理设置中最重要的处理器,因为它是大多数客户端(例如Java和Go客户端)中唯一可用的处理器。

每个队列的默认值为`1000`个span批次。假定每个跨度批处理最多具有64KiB的跨度,则每个队列最多可以容纳64MiB的跨度。

在典型情况下,队列将接近空(度量标准为`jaeger_agent_thrift_udp_server_queue_size`),
因为span批处理应由工作人员快速拾取和处理。
但是,客户提交的跨度批次数量可能会突然增加,从而使批次排队。
当队列已满时,较旧的批次将被覆盖,从而导致跨度被丢弃(度量`jaeger_agent_thrift_udp_server_packets_dropped_total`)。

###调整处理器工作人员

`处理程序工作者`属性集(` processor.jaeger-binary.workers`,`processor.jaeger-compact.workers`,
`processor.zipkin-compact.workers`)指示要启动的并行跨批处理程序的数量。
每个工作程序类型的默认大小均为10。通常,跨度批处理一旦将其放入服务器队列就将进行处理,
并将阻塞一个工作器,直到将整个数据包发送到收集器。
对于处理来自多个客户端的数据的代理,应增加工作人员的数量。鉴于每个工作人员的成本都很低,
一个好的经验法则是每个客户端有10个工作人员且流量适中:鉴于每个跨度批次可能包含多达64KiB的跨度,
这意味着10个工作人员能够同时发送约640KiB给Collector。

##收集器设置

收集器从客户端和代理接收数据。如果配置不正确,它处理的数据可能少于同一主机上可能处理的数据,或者可能会通过消耗比允许更多的内存来使主机过载。

###调整队列大小

与代理类似,收集器能够接收跨度并将其放置在内部队列中进行处理。这使收集器可以立即返回到客户端/代理,而不必等待跨度进入存储。

设置`collector.queue-size"(默认值:`2000")决定了队列应支持多少个跨度。在典型情况下,队列将接近空,因为有足够的工作人员



应该存在以从队列中获取跨度并将其发送到存储。如果队列中的项目数(度量标准为`jaeger_collector_queue_length`)始终很高,
则表明应该增加工作者数,或者表明存储无法跟上接收数据的数量。当队列已满时,队列中较旧的项目将被覆盖,
从而导致跨度被丢弃(度量`jaeger_collector_spans_dropped_total`)。

> 代理的队列大小约为_span batchs_,而收集器的队列大小约为_span batchs_。

鉴于队列大小在大多数情况下应接近于空,因此此设置应与收集器的可用内存一样大,以最大程度地防止突发流量高峰。
但是,如果您的存储层配置不足并且无法跟上,那么即使队列很大,也会迅速填满并开始删除数据。

###调整处理器工作人员

收集器中跨度队列中的项目由工作人员拾取。每个工作人员从队列中选择一个跨度并将其持久化到存储中。
可以通过设置`collector.num-workers`(默认值:`50`)来指定工作者的数量,并且该数量应尽可能高以保持队列接近零。
一般规则是:后备存储越快,工作人员的数量就越少。鉴于工人相对便宜,这个数字可以任意增加。通常,在快速存储的情况下,
队列中每50个项目个worker就足够了。在`collector.queue-size'为`2000`的情况下,大约有40名工人应该足够。
对于较慢的存储机制,应该相应地调整此比率,每个队列项目有更多工人。