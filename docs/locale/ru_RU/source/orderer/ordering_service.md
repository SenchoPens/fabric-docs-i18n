# Ordering-служба

**Для кого это:** Архитекторы, администраторы ordering-службы, создатели каналов

Этот раздел служит идейным вступлением в ordering, как ordering-службы взаимодействуют с пирами, 
их роль в транзакционном потоке и обзор доступных в настоящее время реализаций ordering-службы, с 
фокусом на рекомендованной реализации ordering-службы **Raft**.

## Что такое ordering (упорядочивание)?

Многие распределенные блокчейны, такие как Ethereum и Bitcoin, являются permissionless, что 
означает, что любой узел может участвовать в процессе консенсуса, когда транзакции 
упорядочиваются и группируются в блоки. Поэтому эти системы полагаются на **вероятностные** 
алгоритмы консенсуса, которые гарантируют высокую вероятность синхронизированности реестра, но 
все еще уязвимы к проблеме расходящихся реестров (проблеме "форков" реестра), когда разные 
участники сети имеют разные представления принятого порядка транзакций.

Hyperledger Fabric работает по-другому. Он содержит узел, называемый ordering-узлом, который 
упорядочивает транзакции и в совокупности с другими ordering-узлами формирует **ordering-
службу**. Поскольку архитектура Fabric полагается на **детерминированные** алгоритмы консенсуса, 
каждый блок проверенный пиром гарантированно корректен. В Hyperledger Fabric форк реестров не 
может произойти, как в других распределенных permissionless блокчейн-сетях.

Кроме того разделение процессов подтверждения запуска чейнкодов и упорядочивания делает Fabric 
более производительной и масштабируемой. 

## Ordering-узлы и конфигурация канала

В дополнение к **упорядочивающей** роли, ordering-узлы также поддерживают список организаций, 
имеющих право создавать каналы. Этот список организаций называется **консорциум** и хранится в 
конфигурации системного канала. По умолчанию этот список и канал, где он хранится, могут 
редактироваться только администратором ordering-службы. Заметьте, что ordering-служба может 
хранить несколько таких списков. Конфигурационные транзакции обрабатываются ordering-службой, 
поскольку ей нужно знать текущий набор политик для контролирования доступа.

Ordering-узлы также предоставляют базовый контроль доступа для каналов, ограничивая, кто может 
читать и записывать данные, а также кто может их настраивать. Помните, что на тех, кому разрешено 
изменять конфигурацию в канале, распространяется политика, установленная соответствующими 
администраторами при создании консорциума или канала. Конфигурационные транзакции обрабатываются 
ordering-службой, поскольку она должна знать текущий набор политик для контролирования доступа.
В этих случаях ordering-узел обрабатывает обновление конфигурации, чтобы убедиться, что задавший 
запрос обладает надлежащими администативными правами. Если это так, то ordering-служба проверяет 
запрос обновления по существующей конфигурации, создает новую конфигурационную транзакцию и 
упаковывает ее в блок, который передается всем пирам в канале. Пиры обрабатывают конфигурационную 
транзакцию, чтобы проверить, что изменения, подтвержденные ordering-узлом действительно 
соответствуют политикам, определенным в канале.

## Orderering-узлы и Identity

Все, взаимодействующее с блокчейн-сетью, в том числе пиры, приложения, администраторы, ordering-службы, получает identity их организации из определения Membership Service Provider (MSP)
Everything that interacts with a blockchain network, including peers,
applications, admins, and orderers, acquires their organizational identity from
their digital certificate and their Membership Service Provider (MSP) definition.

Чтобы узнать больше про identities и MSP, ознакомьтесь с нашей документацией про [Identity](../identity/identity.html) и [Membership](../membership/membership.html).

Также как и пиры ordering-узлы принадлежат организациям. По аналогии с пирами, для каждой организации используются отдельные Certificate Authority (CA), будет ли он корневым или промежуточный, связанный с корнем, зависит от вас.


## Orderering-служба и транзакционный поток

### Первая фаза: Proposal

Из темы про [пиры](../peers/peers.html) мы знаем, что они формируют основу блокчейн-сети, храня 
реестры, которые приложения через смартконтракты запрашивают или обновляют.

В частности, приложения, которые хотят обновить реестр участвуют в процессе, состоящем из трех 
этапов и обеспечивающим всем участникам блокчейн-сети синхронизированность их реестров.

На первом шаге клиентское приложение посылает транзакционное proposal набору пиров, которые 
запустят смартконтракты для создания предлагаемого обновления реестра и затем подтвердят 
результаты. Подтверждающие пиры не применяют предложенное обновление к их копии реестра, а 
возвращают ответ на proposal клиентскому приложению. Подтвержденные ответы на транзакционное 
proposal будут в конечном счете упорядочены в блоки на втором этапе, а затем распространены среди 
всех пиров сети для окончательной проверки и сохранения на третьем шаге.

Для более подробного описания первой фазы, ознакомьтесь с темой [пиры](../peers/peers.html#phase-1-proposal).

### Вторая фаза: Упорядочивание и упаковка транзакций в блоки

После завершения первой фазы транзакции клиентское приложение получило подтвержденный ответ на 
транзакционное proposal от набора пиров. Теперь наступает вторая фаза транзакции.

На этой фазе клиентские приложения передают транзакции, содержащие одобренные ответы на 
транзакционное proposal ordering-узлу. Ordering-служба создает блоки из транзакций, которые в 
конечном счете будут распространены среди всех пиров для окончательной проверки и сохранения на 
третьей фазе.

Узлы ordering-службы одновременно получают транзакции от многих разных клиентских приложений. 
Узлы ordering-службы работают вместе, в совокупности состовляя ordering-службу. Ее задача 
заключается в том, чтобы сформировать из представленных им транзакций последовательность и 
упаковать их в *блоки*, составляющие блокчейн.

Количество транзакций в блоке зависит от параметров конфигурации канала, связанных с желаемым 
размером и максимальной задержкой между блоками (а именно параметров `BatchSize` и 
`BatchTimeout`). Блоки затем сохраняются в реестр ordering-службы и распространяются всем пирам, 
присоединившихся к каналу. Если так получилось, что пир был не в сети или присоединился к каналу 
позже, он получить блоки после повторного подключения к узлу ordering-службы или после общения по 
gossip протоколу с другим пиром. Посмотрим, как пиры обработают этот блок на третьем этапе.

![Orderer1](./orderer.diagram.1.png)

*Первой задачей ordering-узла является упаковка предложенных обновлений реестра. В нашем примере 
приложение A1 посылает транзакцию T1, подтвержденную E1 и E2 ordering-узлу O1. Параллельно 
приложение A2 посылает транзакцию T2, подтвержденную E1 ordering-узлу O1. O1 упаковывает T1 и T2 
вместе с другими транзакциями от других приложений сети в блок B2. Можно видеть, что порядок 
транзакций в B2 такой: T1,T2,T3,T4,T6,T5 -- что не обязательно является порядком, в котором они 
прибыли в ordering-узел. (Этот пример показыает очень упрощенную конфигурацию ordering-службы с 
единственным ordering-узлом.)*

Важно заметить, что последовательность транзакций в блоке не обязательно упорядочена по времени 
поступления в ordering-узел, поскольку может быть несколько узлов ordering-службы, которые 
получают транзакции приблизительно в одно время. Ordering-служба строго упорядочивает транзакции, 
и именно этот порядок будут использовать пиры при проверке и сохранении транзакций.

Строгий порядок транзакций в блоках отличает Hyperledger Fabric от других блокчейнов, где одни и 
те же транзакции могут быть упакованы в несколько разных блоков, соревшующиеся за формирование 
цепи. В Hyperledger Fabric блоки, сгенерированные ordering-службой, являются **окончательными**. 
После того, как транзакция записывается в блок, ее позиция в реестре становится неизменяемой. 
Окончательность Hyperledger Fabric означает, что не появляются **форки реестра** --- проверенные 
транзакции никогда не аннулируются и не возвращаются.

Мы также можем видеть, что, в отличие от пиров, ordering-узлы не обрабатывают транзакции и не 
запускают смартконтракты. Каждая транзакция, приходящая к ordering-службе механически 
упаковывается в блок --- ordering-узел не выносит сужденя о содержании транзакции (кроме 
транзакций конфигурации канала).

Теперь мы знаем, что ordering-узлы ответственны за простые, но жизненно важные процессы сбора 
предлагаемых обновлений реестра, их упорядочивание и упаковку в блоки, готовые к распространению. 

### Третья фаза: Проверка и сохранение

Третья фаза транзакционного потока состоит из распространения и последующей проверки блоков, 
после чего их можно сохранять в реестр.

Третья фаза начинается с распространения блоков ordering-узлом всем пирам, подключенным к нему. 
Важно заметить, что не каждый пир должен быть подключен к ordering-узлу --- пиры могут передавать 
друг другу блоки с использованием [**gossip**](../gossip.html) протокола.

Каждый пир независимо проверяет распространенные ему блоки, но детерменировано, поскольку реестр 
должен оставаться повсюду одинаковым. Говоря конкретнее, каждый пир в канале проверит все 
транзакции блока, чтобы удостовериться в подтверждении пиров нужных организаций, что их 
подтверждения совпадают, и что они не утратили валидность после недавно сохраненных транзакций, 
выполнявшихся во время подтверждения. Аннулированные транзакции сохранены в неизменяемом блоке, 
созданном ordering-службой, но помечены, как невалидные пирами и не обновляют состояние реестра.

![Orderer2](./orderer.diagram.2.png)

*Второй задачей ordering-узла является распространение блоков пирамю В нашем примере ordering-
узел O1 распространяет блок B2 пиру P1 и пиру P2. Пир P1
обрабатывает блок B2, добавляет новый блок в свою копию реестра L1. Параллельно пир P2 
обрабатывает блок B2, добавляет новый блок в свою копию реестра L1. По завершению процесса реестр 
L1 обновлен одинаково на обоих пирах P1 и P2, и каждый из них может передать подключенному к нему 
приложению, что транзакции были обработаны.*

По итогу в третьей фазе блоки, созданные ordering-службой, равномерно добавляются в реестр. 
Строгий порядок транзакций в блоках позволяет каждому пиру проверять, что транзакции обновления 
равномерно добавляются повсюду в блокчейн-сети.

Для более подробной информации вернитесь к теме про [пиров](../peers/peers.html#phase-3-validation-and-commit).

## Реализации ordering-служб

While every ordering service currently available handles transactions and
configuration updates the same way, there are nevertheless several different
implementations for achieving consensus on the strict ordering of transactions
between ordering service nodes.

For information about how to stand up an ordering node (regardless of the
implementation the node will be used in), check out [our documentation on standing up an ordering node](../orderer_deploy.html).

* **Raft** (recommended)

  New as of v1.4.1, Raft is a crash fault tolerant (CFT) ordering service
  based on an implementation of [Raft protocol](https://raft.github.io/raft.pdf)
  in [`etcd`](https://coreos.com/etcd/). Raft follows a "leader and
  follower" model, where a leader node is elected (per channel) and its decisions
  are replicated by the followers. Raft ordering services should be easier to set
  up and manage than Kafka-based ordering services, and their design allows
  different organizations to contribute nodes to a distributed ordering service.

* **Kafka** (deprecated in v2.x)

  Similar to Raft-based ordering, Apache Kafka is a CFT implementation that uses
  a "leader and follower" node configuration. Kafka utilizes a ZooKeeper
  ensemble for management purposes. The Kafka based ordering service has been
  available since Fabric v1.0, but many users may find the additional
  administrative overhead of managing a Kafka cluster intimidating or undesirable.

* **Solo** (deprecated in v2.x)

  The Solo implementation of the ordering service is intended for test only and
  consists only of a single ordering node.  It has been deprecated and may be
  removed entirely in a future release.  Existing users of Solo should move to
  a single node Raft network for equivalent function.

## Raft

For information on how to configure a Raft ordering service, check out our
[documentation on configuring a Raft ordering service](../raft_configuration.html).

The go-to ordering service choice for production networks, the Fabric
implementation of the established Raft protocol uses a "leader and follower"
model, in which a leader is dynamically elected among the ordering
nodes in a channel (this collection of nodes is known as the "consenter set"),
and that leader replicates messages to the follower nodes. Because the system
can sustain the loss of nodes, including leader nodes, as long as there is a
majority of ordering nodes (what's known as a "quorum") remaining, Raft is said
to be "crash fault tolerant" (CFT). In other words, if there are three nodes in a
channel, it can withstand the loss of one node (leaving two remaining). If you
have five nodes in a channel, you can lose two nodes (leaving three
remaining nodes).

From the perspective of the service they provide to a network or a channel, Raft
and the existing Kafka-based ordering service (which we'll talk about later) are
similar. They're both CFT ordering services using the leader and follower
design. If you are an application developer, smart contract developer, or peer
administrator, you will not notice a functional difference between an ordering
service based on Raft versus Kafka. However, there are a few major differences worth
considering, especially if you intend to manage an ordering service:

* Raft is easier to set up. Although Kafka has many admirers, even those
admirers will (usually) admit that deploying a Kafka cluster and its ZooKeeper
ensemble can be tricky, requiring a high level of expertise in Kafka
infrastructure and settings. Additionally, there are many more components to
manage with Kafka than with Raft, which means that there are more places where
things can go wrong. And Kafka has its own versions, which must be coordinated
with your orderers. **With Raft, everything is embedded into your ordering node**.

* Kafka and Zookeeper are not designed to be run across large networks. While
Kafka is CFT, it should be run in a tight group of hosts. This means that
practically speaking you need to have one organization run the Kafka cluster.
Given that, having ordering nodes run by different organizations when using Kafka
(which Fabric supports) doesn't give you much in terms of decentralization because
the nodes will all go to the same Kafka cluster which is under the control of a
single organization. With Raft, each organization can have its own ordering
nodes, participating in the ordering service, which leads to a more decentralized
system.

* Raft is supported natively, which means that users are required to get the requisite images and
learn how to use Kafka and ZooKeeper on their own. Likewise, support for
Kafka-related issues is handled through [Apache](https://kafka.apache.org/), the
open-source developer of Kafka, not Hyperledger Fabric. The Fabric Raft implementation,
on the other hand, has been developed and will be supported within the Fabric
developer community and its support apparatus.

* Where Kafka uses a pool of servers (called "Kafka brokers") and the admin of
the orderer organization specifies how many nodes they want to use on a
particular channel, Raft allows the users to specify which ordering nodes will
be deployed to which channel. In this way, peer organizations can make sure
that, if they also own an orderer, this node will be made a part of a ordering
service of that channel, rather than trusting and depending on a central admin
to manage the Kafka nodes.

* Raft is the first step toward Fabric's development of a byzantine fault tolerant
(BFT) ordering service. As we'll see, some decisions in the development of
Raft were driven by this. If you are interested in BFT, learning how to use
Raft should ease the transition.

For all of these reasons, support for Kafka-based ordering service is being
deprecated in Fabric v2.x.

Note: Similar to Solo and Kafka, a Raft ordering service can lose transactions
after acknowledgement of receipt has been sent to a client. For example, if the
leader crashes at approximately the same time as a follower provides
acknowledgement of receipt. Therefore, application clients should listen on peers
for transaction commit events regardless (to check for transaction validity), but
extra care should be taken to ensure that the client also gracefully tolerates a
timeout in which the transaction does not get committed in a configured timeframe.
Depending on the application, it may be desirable to resubmit the transaction or
collect a new set of endorsements upon such a timeout.

### Raft concepts

While Raft offers many of the same features as Kafka --- albeit in a simpler and
easier-to-use package --- it functions substantially different under the covers
from Kafka and introduces a number of new concepts, or twists on existing
concepts, to Fabric.

**Log entry**. The primary unit of work in a Raft ordering service is a "log
entry", with the full sequence of such entries known as the "log". We consider
the log consistent if a majority (a quorum, in other words) of members agree on
the entries and their order, making the logs on the various orderers replicated.

**Consenter set**. The ordering nodes actively participating in the consensus
mechanism for a given channel and receiving replicated logs for the channel.
This can be all of the nodes available (either in a single cluster or in
multiple clusters contributing to the system channel), or a subset of those
nodes.

**Finite-State Machine (FSM)**. Every ordering node in Raft has an FSM and
collectively they're used to ensure that the sequence of logs in the various
ordering nodes is deterministic (written in the same sequence).

**Quorum**. Describes the minimum number of consenters that need to affirm a
proposal so that transactions can be ordered. For every consenter set, this is a
**majority** of nodes. In a cluster with five nodes, three must be available for
there to be a quorum. If a quorum of nodes is unavailable for any reason, the
ordering service cluster becomes unavailable for both read and write operations
on the channel, and no new logs can be committed.

**Leader**. This is not a new concept --- Kafka also uses leaders, as we've said ---
but it's critical to understand that at any given time, a channel's consenter set
elects a single node to be the leader (we'll describe how this happens in Raft
later). The leader is responsible for ingesting new log entries, replicating
them to follower ordering nodes, and managing when an entry is considered
committed. This is not a special **type** of orderer. It is only a role that
an orderer may have at certain times, and then not others, as circumstances
determine.

**Follower**. Again, not a new concept, but what's critical to understand about
followers is that the followers receive the logs from the leader and
replicate them deterministically, ensuring that logs remain consistent. As
we'll see in our section on leader election, the followers also receive
"heartbeat" messages from the leader. In the event that the leader stops
sending those message for a configurable amount of time, the followers will
initiate a leader election and one of them will be elected the new leader.

### Raft in a transaction flow

Every channel runs on a **separate** instance of the Raft protocol, which allows
each instance to elect a different leader. This configuration also allows
further decentralization of the service in use cases where clusters are made up
of ordering nodes controlled by different organizations. While all Raft nodes
must be part of the system channel, they do not necessarily have to be part of
all application channels. Channel creators (and channel admins) have the ability
to pick a subset of the available orderers and to add or remove ordering nodes
as needed (as long as only a single node is added or removed at a time).

While this configuration creates more overhead in the form of redundant heartbeat
messages and goroutines, it lays necessary groundwork for BFT.

In Raft, transactions (in the form of proposals or configuration updates) are
automatically routed by the ordering node that receives the transaction to the
current leader of that channel. This means that peers and applications do not
need to know who the leader node is at any particular time. Only the ordering
nodes need to know.

When the orderer validation checks have been completed, the transactions are
ordered, packaged into blocks, consented on, and distributed, as described in
phase two of our transaction flow.

### Architectural notes

#### How leader election works in Raft

Although the process of electing a leader happens within the orderer's internal
processes, it's worth noting how the process works.

Raft nodes are always in one of three states: follower, candidate, or leader.
All nodes initially start out as a **follower**. In this state, they can accept
log entries from a leader (if one has been elected), or cast votes for leader.
If no log entries or heartbeats are received for a set amount of time (for
example, five seconds), nodes self-promote to the **candidate** state. In the
candidate state, nodes request votes from other nodes. If a candidate receives a
quorum of votes, then it is promoted to a **leader**. The leader must accept new
log entries and replicate them to the followers.

For a visual representation of how the leader election process works, check out
[The Secret Lives of Data](http://thesecretlivesofdata.com/raft/).

#### Snapshots

If an ordering node goes down, how does it get the logs it missed when it is
restarted?

While it's possible to keep all logs indefinitely, in order to save disk space,
Raft uses a process called "snapshotting", in which users can define how many
bytes of data will be kept in the log. This amount of data will conform to a
certain number of blocks (which depends on the amount of data in the blocks.
Note that only full blocks are stored in a snapshot).

For example, let's say lagging replica `R1` was just reconnected to the network.
Its latest block is `100`. Leader `L` is at block `196`, and is configured to
snapshot at amount of data that in this case represents 20 blocks. `R1` would
therefore receive block `180` from `L` and then make a `Deliver` request for
blocks `101` to `180`. Blocks `180` to `196` would then be replicated to `R1`
through the normal Raft protocol.

### Kafka (deprecated in v2.x)

The other crash fault tolerant ordering service supported by Fabric is an
adaptation of a Kafka distributed streaming platform for use as a cluster of
ordering nodes. You can read more about Kafka at the [Apache Kafka Web site](https://kafka.apache.org/intro),
but at a high level, Kafka uses the same conceptual "leader and follower"
configuration used by Raft, in which transactions (which Kafka calls "messages")
are replicated from the leader node to the follower nodes. In the event the
leader node goes down, one of the followers becomes the leader and ordering can
continue, ensuring fault tolerance, just as with Raft.

The management of the Kafka cluster, including the coordination of tasks,
cluster membership, access control, and controller election, among others, is
handled by a ZooKeeper ensemble and its related APIs.

Kafka clusters and ZooKeeper ensembles are notoriously tricky to set up, so our
documentation assumes a working knowledge of Kafka and ZooKeeper. If you decide
to use Kafka without having this expertise, you should complete, *at a minimum*,
the first six steps of the [Kafka Quickstart guide](https://kafka.apache.org/quickstart) before experimenting with the
Kafka-based ordering service. You can also consult
[this sample configuration file](https://github.com/hyperledger/fabric/blob/release-1.1/bddtests/dc-orderer-kafka.yml)
for a brief explanation of the sensible defaults for Kafka and ZooKeeper.

To learn how to bring up a Kafka-based ordering service, check out [our documentation on Kafka](../kafka.html).

<!--- Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/) -->
