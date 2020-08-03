# Политики

**Для кого это**: Архитекторы, разработчики приложений и смарт-контрактов,
администраторы

В этом разделе мы обсудим:

* [Что такое политика](#что-такое-политика)
* [Зачем нужны политики](#зачем-нужны-политики)
* [Как политики реализованы в Fabric](#как-политики-реализованы-в-fabric)
* [Области политик Fabric](#области-политик-Fabric)
* [Как написать политику в Fabric](#как-написать-политику-в-fabric)
* [Fabric chaincode lifecycle](#fabric-chaincode-lifecycle)
* [Overriding policy definitions](#overriding-policy-definitions)

## Что такое политика?

Простыми словами, политика это набор правил, определяющих структуру принятия 
решений. Политика, как правило, описывает права, которыми субъект обладает в
отношении какого-либо актива. Мы увидим, что политики используются в нашей 
повседневной жизни для защиты важных активов, таких как машины или дома.

Например, страховой полис описывает условия, при которых будет выплачена 
страховка. Политика --- соглашение между владельцем страховки и страховой 
компанией и определяет права и обязанности обеих сторон.

Страховые полиса используются для управления рисками, а в Hyperledger
Fabric политики --- это механизм для управлением инфраструктурой. Политики 
Fabric показывают, как члены сети приходят к соглашению, принимая или 
отклоняя изменения в сети, канале или смарт-контракте. Политики согласуются
консорциумом при первой настройке сети, однако их можно поменять по мере ее 
роста и развития. Например, они описывают критерии добавления в канал и исключения 
из него членов сети, могут поменять порядок формирования блоков или указать 
число организаций, необходимое для утверждения смарт-контрактов. Все 
эти действия описываются политикой, определяющей лица, которые могут совершить 
определеннное действие. Проще говоря, все, что вы хотите делать в сети Fabric 
регулируется политиками.

## Зачем нужны политики

Политики --- одна из черт Hyperledger Fabric, отличающих его от других блокчейнов,
таких как Ethereum или Bitcoin. В таких системах транзакции могут создаваться и 
утверждаться любым узлом в сети. Политики, регулирующие сеть, могут быть изменены в
любой момент, но лишь с использованием того же процесса, который управляет кодом. 
Поскольку Fabric --- permissioned-блокчейн, ее пользователи принимают все решения,
касающиеся управления сети до ее запуска, а после запуска могут изменять ее.

Политики позволяют членам выбирать организации, которые имеют доступ в сеть Fabric 
или право ее обновлять, и предоставляют для этого механизмы. Политики содержат списки
организаций, имеющих доступ к определенному ресурсу, например пользователям или чейнкодам.
Они также знают количество организаций должны дать свое согласие на предложение
обновить ресурс, например, смарт-контракт или канал. Политики собирают подписи, 
относящиеся к транзакциям и их предложениям и подтверждают их, если они соответствуют
правилам, установленным в сети.

## Как политики реализованы в Fabric

Политики реализованы на разных уровнях сети Fabric. Каждая отдельная область
политики управляет разными аспектами работы сети.

![policies.policies](./FabricPolicyHierarchy-2.png) *Наглядное представление
иерархии политик Fabric.*

### Настройка системного канала в части ordering.

Каждая сеть начинается с **системного канала**. В каждой сети должен быть
ровно один системный канал для ordering-службы, который создается первым.
Системный канал содержит организации-члены ordering-службы (ordering-организации), 
а также организации, которые являются сторонами транзакций (консорциум-организации).

Политики в конфигурационных блоках системного канала управляют консенсусом,
используемым ordering-службой, и определяют то, как создаются новые блоки. Системный
канал также отвечает за то, кто из членов консорциума могут создавать новые каналы.

### Настройка канала в части политик прикладных транзакций.

Обычные _channels_ предоставляют механизмы для приватной коммуникации
организаций консорциума.

Политики могут предоставить возможность удалять или добавлять членов в канал.
Каналы также управляют тем, какие организации должны одобрить чейнкод до того, как 
чейнкод определен и сохранен в канал, следуя жизненному циклу чейнкода в Fabric. Канал
сразу после создания по умолчанию наследует все параметры ordering-службы и от 
системного канала ordering-службы. Однако эти параметры (как и политики) можно настроить 
в каждом канале.

### Access control lists (ACL) (списки контроля доступа)

Администраторам сети будет особенно интересно использование Fabric ACL (списки 
контроля доступа), которые предоставляют возможность настраивать доступ к ресурсам,
связывая эти ресурсы с существующей политикой. Эти "ресурсы" могут быть правами 
на взаимодействие с системным чейнкодом (например, "GetBlockByNumber" для системного
чейнкода"qscc") и другими ресурсами (например, возможность получать события, связанные 
с блоками). ACL обращается к политикам, опрделенным в конфигурации канала и расширяет их
для контроля над дополнительными ресурсами. По умолчанию набор ACL Fabric находится в 
файле `configtx.yaml` под разделом `Application: &ApplicationDefaults`, но они могут и 
должны быть переопределены в промышленном окружении. Список ресурсов, находящийся в 
`configtx.yaml` --- полный набор всех внутренних ресурсов, на данный момент определенных 
Fabric.

В этом файле ACL представлены в следующем формате:

```
# ACL policy for chaincode to chaincode invocation
peer/ChaincodeToChaincode: /Channel/Application/Readers
```

Здесь `peer/ChaincodeToChaincode` отвечает за предоставляемый ресурс, а
`/Channel/Application/Readers` отсылает к политике, которая должна быть удовлетворена для того, 
чтобы соответствующая сделка считалась действительной.

Чтобы узнать больше про ACL, ознакомьтесь с разделом про [ACLs](../access_control.html) из 
Руководстве по эксплуатации.

### Политика подтверждения смарт-контрактов.

Каждый смарт-контракт внутри пакета чейнкода имеет политику подтверждения, которая указывает, 
сколько пиров разных членов канала должны выполнить и проверить транзакцию смарт-контракта, 
чтобы транзакция была признана валидной. Таким образом, политика подтверждения определяет организации (через их пиры), которые должны "подтвердить" (одобрить) реализацию предложения.


### Политики изменения

`Modification policy` (политики изменения) --- последний тип политик, играющий важную роль в 
работе Fabric. Политики изменения указывают группу identities, необходимых для подписи 
(одобрения) любой конфигурации _update_. Это политика, определяющая, как другие политики могут 
изменяться. Таким образом, каждый элемент конфигурации канала включает в себя ссылку на 
политику, которая управляет изменениями этого канала.

## Области политик Fabric

Хотя политики Fabric могут подстраиваться под нужды сети, области политик разделяются 
управляемых организациями ordering-службы и управляемых членамт консорциума. В 
нижеприведенной диаграмме можно увидеть, как стандартные политики реализуют контроль над 
областями политик Fabric.

![policies.policies](./FabricPolicyHierarchy-4.png) *Более детальный взгяд на разделение политик 
на управляемых ordering-организациями и управляемых организациями консорциума.*

В функционирующей сети Fabric может существовать множество организаций, обладающих разными 
обязанностями. Области предоставляют возможность раздачи различных ролей и привилегий разным 
организациям, позволяя создателям ordering-службы устанавливать первоначальные правила и 
членство в консорциуме. Также они позволяют присоединившимся к консорциуму организациям 
создавать приватные каналы, управлять своей бизнес логикой, и регулировать доступ к данным, 
вводящимся в сеть.

Конфигурация системного канала и частичная конфигурация обычного канала предоставляет ordering-о
рганизациям механизм консенсуса, исползуемый узлами ordering-службы, а также контроль над тем, 
какие организации являются членами консорциума, как блоки 
достовляются в каналы.

Конфигурация системного канала позволяет членам консорциума создавать каналы. Каналы и ACL --- 
это механизмы, используемые организациями консорциума для добавления в канал и удаления из него, 
а также регулирования доступа к данным и смарт-контрактам в канале.

## Как написать политику в Fabric

Если вы хотите что-нибудь изменить в Fabric, политика, связанная с ресурсом, определяет **кто** 
должен подтвердить изменение, или с явной подписью каждого участника или же с неявной подписью 
группы. В области страхования, аналогом явной подписи может быть требование одобрения хотя бы одного члена группы страхования. А аналогом неявной подписи может быть требование одобрения 
большинством управляющих членов группы страхования. Это особенно полезно, поскольку изменения в 
составе группы не требуют обновления политики. В Hyperledger Fabric явные подписи в политике используют синтаксис `Signature`, а неявные `ImplicitMeta`.

### Политики типа Signature 

Политики `Signature` указывают определенных типов пользователей, которые должны подписать для 
подтверждения политики, например, `OR('Org1.peer', 'Org2.peer')`. Эти политики считаются самыми 
универсальными, поскольку позволяют конструировать очень специфичные правила, например: 
"Администратор организации А (org A) и 2 других администратора, или 5 из 6 администраторов 
организации". Синтаксис поддерживает произвольные сочетания `AND`, `OR` и `NOutOf`. Например, 
политику можно описать, используя `AND('Org1.member', 'Org2.member')`, что означает, что для 
удовлетворения политики нужна подпись хотя бы одного члена Org1 И (AND) одного члена Org2. 

### Политики типа ImplicitMeta

Политики `ImplicitMeta` допустимы только в контексте конфигурации канала, основанной на 
многоуровневой иерархии политики в дереве конфигураций. Политики ImplicitMeta объединяют в себе 
результат политик, расположенных глубже в конфигурационном дереве, определенных политикой 
Signature. Политики называются `Implicit`, поскольку они неявно строятся на основе существующих оганизаций в конфигурации канала, а `Meta`, поскольку они зависят не от конкретных MSP principals, а другими политиками, находящимися ниже их в конфигруационном дереве.

Нижеприведенная диаграмма иллюстрирует многоуровневую структуру канала и показывет, как политика
`ImplicitMeta` администраторов конфигурации канала, `/Channel/Admins`
The following diagram illustrates the tiered policy structure for an application
channel and shows how the `ImplicitMeta` channel configuration admins policy,
named `/Channel/Admins`, is resolved when the sub-policies named `Admins` below it
in the configuration hierarchy are satisfied where each check mark represents that
the conditions of the sub-policy were satisfied.

![policies.policies](./FabricPolicyHierarchy-6.png)

Как можно видеть на диаграмме выше, политики `ImplicitMeta`, Type = 3, используют другой синтаксис, `"<ANY|ALL|MAJORITY> <SubPolicyName>"`, например:
```
`MAJORITY sub policy: Admins`
```
The diagram shows a sub-policy `Admins`, which refers to all the `Admins` policy
below it in the configuration tree. You can create your own sub-policies
and name them whatever you want and then define them in each of your
organizations.

As mentioned above, a key benefit of an `ImplicitMeta` policy such as `MAJORITY
Admins` is that when you add a new admin organization to the channel, you do not
have to update the channel policy. Therefore `ImplicitMeta` policies are
considered to be more flexible as the consortium members change. The consortium
on the orderer can change as new members are added or an existing member leaves
with the consortium members agreeing to the changes, but no policy updates are
required. Recall that `ImplicitMeta` policies ultimately resolve the
`Signature` sub-policies underneath them in the configuration tree as the
diagram shows.

You can also define an application level implicit policy to operate across
organizations, in a channel for example, and either require that ANY of them
are satisfied, that ALL are satisfied, or that a MAJORITY are satisfied. This
format lends itself to much better, more natural defaults, so that each
organization can decide what it means for a valid endorsement.

Further granularity and control can be achieved if you include [`NodeOUs`](msp.html#organizational-units) in your
organization definition. Organization Units (OUs) are defined in the Fabric CA
client configuration file and can be associated with an identity when it is
created. In Fabric, `NodeOUs` provide a way to classify identities in a digital
certificate hierarchy. For instance, an organization having specific `NodeOUs`
enabled could require that a 'peer' sign for it to be a valid endorsement,
whereas an organization without any might simply require that any member can
sign.

## An example: channel configuration policy

Understanding policies begins with examining the `configtx.yaml` where the
channel policies are defined. We can use the `configtx.yaml` file in the Fabric
test network to see examples of both policy syntax types. We are going to examine
the configtx.yaml file used by the [fabric-samples/test-network](https://github.com/hyperledger/fabric-samples/blob/{BRANCH}/test-network/configtx/configtx.yaml) sample.

The first section of the file defines the organizations of the network. Inside each
organization definition are the default policies for that organization, `Readers`, `Writers`,
`Admins`, and `Endorsement`, although you can name your policies anything you want.
Each policy has a `Type` which describes how the policy is expressed (`Signature`
or `ImplicitMeta`) and a `Rule`.

The test network example below shows the Org1 organization definition in the system
channel, where the policy `Type` is `Signature` and the endorsement policy rule
is defined as `"OR('Org1MSP.peer')"`. This policy specifies that a peer that is
a member of `Org1MSP` is required to sign. It is these signature policies that
become the sub-policies that the ImplicitMeta policies point to.  

<details>
  <summary>
    **Click here to see an example of an organization defined with signature policies**
  </summary>

```
 - &Org1
        # DefaultOrg defines the organization which is used in the sampleconfig
        # of the fabric.git development environment
        Name: Org1MSP

        # ID to load the MSP definition as
        ID: Org1MSP

        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp

        # Policies defines the set of policies at this level of the config tree
        # For organization policies, their canonical path is usually
        #   /Channel/<Application|Orderer>/<OrgName>/<PolicyName>
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.peer', 'Org1MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org1MSP.admin')"
            Endorsement:
                Type: Signature
                Rule: "OR('Org1MSP.peer')"
```
</details>

The next example shows the `ImplicitMeta` policy type used in the `Application`
section of the `configtx.yaml`. These set of policies lie on the
`/Channel/Application/` path. If you use the default set of Fabric ACLs, these
policies define the behavior of many important features of application channels,
such as who can query the channel ledger, invoke a chaincode, or update a channel
config. These policies point to the sub-policies defined for each organization.
The Org1 defined in the section above contains `Reader`, `Writer`, and `Admin`
sub-policies that are evaluated by the `Reader`, `Writer`, and `Admin` `ImplicitMeta`
policies in the `Application` section. Because the test network is built with the
default policies, you can use the example Org1 to query the channel ledger, invoke a
chaincode, and approve channel updates for any test network channel that you
create.

<details>
  <summary>
    **Click here to see an example of ImplicitMeta policies**
  </summary>
```
################################################################################
#
#   SECTION: Application
#
#   - This section defines the values to encode into a config transaction or
#   genesis block for application related parameters
#
################################################################################
Application: &ApplicationDefaults

    # Organizations is the list of orgs which are defined as participants on
    # the application side of the network
    Organizations:

    # Policies defines the set of policies at this level of the config tree
    # For Application policies, their canonical path is
    #   /Channel/Application/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
        Endorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
```
</details>

## Fabric chaincode lifecycle

In the Fabric 2.0 release, a new chaincode lifecycle process was introduced,
whereby a more democratic process is used to govern chaincode on the network.
The new process allows multiple organizations to vote on how a chaincode will
be operated before it can be used on a channel. This is significant because it is
the combination of this new lifecycle process and the policies that are
specified during that process that dictate the security across the network. More details on
the flow are available in the [Fabric chaincode lifecycle](../chaincode_lifecycle.html)
concept topic, but for purposes of this topic you should understand how policies are
used in this flow. The new flow includes two steps where policies are specified:
when chaincode is **approved** by organization members, and when it is **committed**
to the channel.

The `Application` section of  the `configtx.yaml` file includes the default
chaincode lifecycle endorsement policy. In a production environment you would
customize this definition for your own use case.

```
################################################################################
#
#   SECTION: Application
#
#   - This section defines the values to encode into a config transaction or
#   genesis block for application related parameters
#
################################################################################
Application: &ApplicationDefaults

    # Organizations is the list of orgs which are defined as participants on
    # the application side of the network
    Organizations:

    # Policies defines the set of policies at this level of the config tree
    # For Application policies, their canonical path is
    #   /Channel/Application/<PolicyName>
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
        Endorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
```

- The `LifecycleEndorsement` policy governs who needs to _approve a chaincode
definition_.
- The `Endorsement` policy is the _default endorsement policy for
a chaincode_. More on this below.

## Chaincode endorsement policies

The endorsement policy is specified for a **chaincode** when it is approved
and committed to the channel using the Fabric chaincode lifecycle (that is, one
endorsement policy covers all of the state associated with a chaincode). The
endorsement policy can be specified either by reference to an endorsement policy
defined in the channel configuration or by explicitly specifying a Signature policy.

If an endorsement policy is not explicitly specified during the approval step,
the default `Endorsement` policy `"MAJORITY Endorsement"` is used which means
that a majority of the peers belonging to the different channel members
(organizations) need to execute and validate a transaction against the chaincode
in order for the transaction to be considered valid.  This default policy allows
organizations that join the channel to become automatically added to the chaincode
endorsement policy. If you don't want to use the default endorsement
policy, use the Signature policy format to specify a more complex endorsement
policy (such as requiring that a chaincode be endorsed by one organization, and
then one of the other organizations on the channel).

Signature policies also allow you to include `principals` which are simply a way
of matching an identity to a role. Principals are just like user IDs or group
IDs, but they are more versatile because they can include a wide range of
properties of an actor’s identity, such as the actor’s organization,
organizational unit, role or even the actor’s specific identity. When we talk
about principals, they are the properties which determine their permissions.
Principals are described as 'MSP.ROLE', where `MSP` represents the required MSP
ID (the organization),  and `ROLE` represents one of the four accepted roles:
Member, Admin, Client, and Peer. A role is associated to an identity when a user
enrolls with a CA. You can customize the list of roles available on your Fabric
CA.

Some examples of valid principals are:
* 'Org0.Admin': an administrator of the Org0 MSP
* 'Org1.Member': a member of the Org1 MSP
* 'Org1.Client': a client of the Org1 MSP
* 'Org1.Peer': a peer of the Org1 MSP
* 'OrdererOrg.Orderer': an orderer in the OrdererOrg MSP

There are cases where it may be necessary for a particular state
(a particular key-value pair, in other words) to have a different endorsement
policy. This **state-based endorsement** allows the default chaincode-level
endorsement policies to be overridden by a different policy for the specified
keys.

For a deeper dive on how to write an endorsement policy refer to the topic on
[Endorsement policies](../endorsement-policies.html) in the Operations Guide.

**Note:**  Policies work differently depending on which version of Fabric you are
  using:
- In Fabric releases prior to 2.0, chaincode endorsement policies can be
  updated during chaincode instantiation or by using the chaincode lifecycle
  commands. If not specified at instantiation time, the endorsement policy
  defaults to “any member of the organizations in the channel”. For example,
  a channel with “Org1” and “Org2” would have a default endorsement policy of
  “OR(‘Org1.member’, ‘Org2.member’)”.
- Starting with Fabric 2.0, Fabric introduced a new chaincode
  lifecycle process that allows multiple organizations to agree on how a
  chaincode will be operated before it can be used on a channel.  The new process
  requires that organizations agree to the parameters that define a chaincode,
  such as name, version, and the chaincode endorsement policy.

## Overriding policy definitions

Hyperledger Fabric includes default policies which are useful for getting started,
developing, and testing your blockchain, but they are meant to be customized
in a production environment. You should be aware of the default policies
in the `configtx.yaml` file. Channel configuration policies can be extended
with arbitrary verbs, beyond the default `Readers, Writers, Admins` in
`configtx.yaml`. The orderer system and application channels are overridden by
issuing a config update when you override the default policies by editing the
`configtx.yaml` for the orderer system channel or the `configtx.yaml` for a
specific channel.

See the topic on
[Updating a channel configuration](../config_update.html#updating-a-channel-configuration)
for more information.

<!--- Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/) -->
