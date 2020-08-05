# Membership Service Provider (MSP)

## Зачем нужен MSP?

Поскольку Fabric -- permissioned сеть, участникам блокчейн-сети нужен способ подтвердить их 
identity остальной части сети, чтобы осуществлять транзакции в сети. Если вы читали документацию 
про [Identity](../identity/identity.html), то вы видели, как Public Key Infrastructure (PKI) 
(инфраструктура публичного ключа) с помощью цепочек доверия предоставляет проверяемые 
identities. Как эта цепочка доверия используется блокчейн-сетью?

Certificate Authorities раздают identities, генерируя публичный и приватный ключи, которые 
формируют пару, с помощью которой можно подтвердить identity. Поскольку приватный ключ должен 
оставаться приватным, необходим механизм, позволяющий получить подтверждение, и здесь появляется 
MSP. Например, пир использует приватный ключ, чтобы поставить цифровую подпись на транзакции. 
MSP ordering-службы владеет публичным ключом пира и использует его, чтобы проверить, что 
приложенная к транзакции подпись валидна. Приватный ключ используется для создания подписи на 
транзакции, которая может соответствовать только парному публичному ключу (и именно за эту часть 
отвечает MSP). Таким образом, MSP -- это механизм, позволяющий узнавать и доверять отдельно 
взятой identity, при этом не раскрывая ее публичный ключ.

Возвращаясь к сценарию из темы Identity про кредитные карты: Certificate Authority -- как 
поставщик карт -- выдает множество различных видов identities, поддающихся проверке. MSP же 
определяет, какие поставщики карт принимаются в магазине. Так, MSP превращает identity 
(кредитную карту) в "роль" (возможность покупать вещи в магазине). 

Возможность превращать проверяемые identities в роли является очень важной в работе сетей 
Fabric, поскольку это позволяет организациям, узлам и каналам устанавливать MSP, который 
определяет кто что может делать на уровне организации, узла и канала. 

![MSP1a](./membership.msp.diagram.png)

*Identities похожи на кредитные карты, подтверждающие, что вы способны заплатить. MSP похож на 
список принимаемых карт.*

Рассмотрим консорциум банков, управляющих блокчейн-сетью. Каждый банк управляет пирами и 
ordering-узлами, пиры подтверждают транзакции в сети. Однако в каждом банке есть департаменты и 
владельцы счетов. Владельцы счетов принадлежат организациям, но не управляют узлами в сети. Они 
лишь будут взаимодействовать с системой с помощью мобильных или веб приложений. Как сеть узнает 
и будет различать эти identities? CA создал identities, но, как и в примере с картой, эти 
identities нельзя просто выдать -- их должна узнавать сеть. MSP испоьзуются для определения 
организаций, которым доверяют члены сети. MSP также является механизмом, предоставляющим членам 
роли и разрешения в сети. Поскольку MSP, определяющие эти организации, известны членам сети, их 
можно использовать, чтобы проверить, что структуры в сети, пытающиеся выполнить действия, имеют 
право на эти действия. 

В итоге, если вы хотите присоединиться к _существующей_ сети, вы должны превратить вашу identity 
во что-то узнаваемое в сети. MSP -- механизм, позволяющий вам участвовать в permissioned 
блокчейн-сети. Для осуществления транзакций член сети Fabric должен иметь:

1. Иметь identity, выданное доверенным CA
2. Стать членом _организации_, утвержденной и признаваемой членами сети
С помощью MSP identity привязана к организации, что достигается путем добавления публичного 
ключа члена (также называемого сертификатом) в MSP организации.
3. Добавить MSP или в [консорциум](../glossary.html#consortium) в сети или в канал.
4. Убедиться, что MSP включен в определения [политики](../policies/policies.html) в сети.

## Что такое MSP?

Несмотря на имя, Membership Service Provider ничего не предоставляет. MSP реализован как набор 
директорий, которые добавляются в конфигурацию сети и используются для определения организации 
как внутри организации (организации решают, кто является их администратором), так и снаружи 
(позволяя другим организациям подтверждать, что эти органы имеют полномочия на действия, которые 
они хотят совершить).

MSP определяет, какие identity валидные, а какие нет, либо состовляя список валидных identity, 
либо указывая доверенные корневые и промежуточные CA, которые могут выписывать identity.

Но область ответственности MSP выходит за рамки простого перечисления участников сети и членов 
каналов. MSP превращает identity в **роль**, определяя особые привилегии участника в канале или 
узле. Заметьте, что если пользователь регистрируется с помощью Fabric CA, роль администратора, 
пира, ordering-службы или члена должна быть связана с пользователем. Например, identities, 
зарегестрированные в качестве "пиров" должны, очевидно, принадлежать пирам. Аналогично, 
identities, зарегестрированные в качестве "администраторов" должны принадлежать администраторам 
организаций. Мы более подробно рассмотрим значение этих ролей в этой теме.

В дополнение, MSP может отзывать (аннулировать) конкретные identity (что упоминалось в ...), но 
мы обсудим, как этот процесс выглядит со стороны MSP.

## Области MSP

MSP встречается в двух областях блокчейн-сети:

* Локально на узле участника (**локальный MSP**)
* В конфигурации каналов (**канальный MSP**)

Ключевое различие между локальным и канальным MSP не в том, как они функционируют -- оба 
превращают identities в роли -- а их **масштаб**. В каждом MSP перечисляются функции и 
полномочия на определенном уровне управления.

### Локальные MSP

**Локальные MSP определены для клиентов и узлов (пиров и ordering-служб)**.
Локальные MSP определяют разрешения для узла в сети. Локальный MSP клиентов (владельцев счетов в 
сценарии про банк) позволяют пользователям указывать себя на транзакциях в качестве члена канала 
(например, в транзакции чейнкода) или в качестве обладателя конкретной роли в системе (например, 
администратор организации в конфигурационных транзакциях).

**Каждый узел должен иметь определенный локальный MSP**, поскольку он определяет, кто имеет 
административные права или права участия на этом уровне (пиры-администраторы не обязательно 
будут администраторами каналов и наоборот). Это позволяет аутентифицировать сообщения членов вне 
канала и определять разрешения на управление конкретными узлами (имеющими, например, возможность 
установить чейнкод на пир). Заметьте, что у организации может быть несколько узлов. MSP 
определяет администраторов организации. И организация, администратор организации, администратор 
узла и сам узел должны входить в одну цепочку доверия.

Локальный MSP ordering-службы также определен в файловой системе узла и применяется только к 
этому узлу. Так же как и пиры, ordering-службы принадлежат единственной организации и поэтому 
имеют один MSP для ведения списка участников или узлов, которым она доверяет.

### Канальные MSP

В отличие от локального MSP, **канальный MSP определяет административные права и права участия на уровне каналов**. Пиры и ordering-узлы в канале пользуются одним и тем же канальным MSP, и поэтому могут корректно определять участников канала. Это означает, что, если организация хочет присоединиться к каналу, в конфигурацию канала нужно включить MSP. В противном случае транзакции от identities этой организации будут отклоняться. В отличие от локальных MSP, канальные MSP описываются в конфигурации канала.

![MSP1d](./ChannelMSP.png)

*Кусок кода из файла config.json канала, включающего два MSP организации.*

**Канальные MSP определяют, кто на уровне канала обладает полномочиями**.
Канальный MSP определяет _отношения_ между identities членов канала (которые сами являются MSP) и обеспечивает соблюдение политики на уровне каналов. Канальные MSP содержат MSP организаций-членов канала.

**Каждая участвующая в канале организация должна иметь MSP, определенный для нее**. На самом деле рекомендуется соотношение один-к-одному между организациями и MSP. MSP определяет, какие члены уполномочены действовать от имени организации. Это включает в себя конфигурацию самого MSP 
**Every organization participating in a channel must have an MSP defined for it**. In fact, it is recommended that there is a one-to-one mapping between organizations and MSPs. The MSP defines which members are empowered to act on behalf of the organization. This includes configuration of the MSP itself as well as approving administrative tasks that the organization has role, such as adding new members to a channel. If all network members were part of a single organization or MSP, data privacy is sacrificed. Multiple organizations facilitate privacy by segregating ledger data to only channel members. If more granularity is required within an organization, the organization can be further divided into organizational units (OUs) which we describe in more detail later in this topic.

**The system channel MSP includes the MSPs of all the organizations that participate in an ordering service.** An ordering service will likely include ordering nodes from multiple organizations and collectively these organizations run the ordering service, most importantly managing the consortium of organizations and the default policies that are inherited by the application channels.

**Local MSPs are only defined on the file system of the node or user** to which they apply. Therefore, physically and logically there is only one local MSP per
node. However, as channel MSPs are available to all nodes in the channel, they are logically defined once in the channel configuration. However, **a channel MSP is also instantiated on the file system of every node in the channel and kept synchronized via consensus**. So while there is a copy of each channel MSP on the local file system of every node, logically a channel MSP resides on and is maintained by the channel or the network.

The following diagram illustrates how local and channel MSPs coexist on the network:  

![MSP3](./membership.diagram.2.png)

*The MSPs for the peer and orderer are local, whereas the MSPs for a channel (including the network configuration channel, also known as the system channel) are global, shared across all participants of that channel. In this figure, the network system channel is administered by ORG1, but another application channel can be managed by ORG1 and ORG2. The peer is a member of and managed by ORG2, whereas ORG1 manages the orderer of the figure. ORG1 trusts identities from RCA1, whereas ORG2 trusts identities from RCA2. It is important to note that these are administration identities, reflecting who can administer these components. So while ORG1 administers the network, ORG2.MSP does exist in the network definition.*

## Какую роль в MSP играют организации?

**Организация** -- это логическая управляемая группа членов. Она может быть размером с международную корпорацию или размером с цветочный магазин. Самое важное в организациях (или  **orgs**) -- то, что они управляют своими членами в рамках единого MSP. MSP привязывает identity к организации. Заметьте, что это отличается от определения организации в сертификате X.509, который мы упоминали выше.

Благодаря уникальным отношениям между организацией и ее MSP разумно называть MSP в честь организации, обычай, который принят в большинстве конфигураций политик. Например, у организации `ORG1` MSP будет называться как-то похоже на `ORG1-MSP`. В некоторых случаях организации могут требоваться многочисленные группы, например, в тех случаях, когда организации используют каналы для выполнения различных бизнес-функций. В таких случаях логично иметь несколько MSP, названных соответственно, например, `ORG2-MSP-NATIONAL` и `ORG2-MSP-GOVERNMENT`.

### Organizational Units (OUs) и MSP

Организация может также быть разделена на несколько **organizational units** (OU, организационные подразделения)
An organization can also be divided into multiple **organizational units**, each of which has a certain set of responsibilities, also referred to as `affiliations`. Think of an OU as a department inside an organization. For example, the `ORG1` organization might have both `ORG1.MANUFACTURING` and `ORG1.DISTRIBUTION` OUs to reflect these separate lines of business. When a CA issues X.509 certificates, the `OU` field in the certificate specifies the line of business to which the identity belongs. A benefit of using OUs like this is that these values can then be used in policy definitions in order to restrict access or in smart contracts for attribute-based access control. Otherwise, separate MSPs would need to be created for each organization.

Specifying OUs is optional. If OUs are not used, all of the identities that are part of an MSP --- as identified by the Root CA and Intermediate CA folders --- will be considered members of the organization.

### Node OU Roles and MSPs

Additionally, there is a special kind of OU, sometimes referred to as a `Node OU`, that can be used to confer a role onto an identity. These Node OU roles are defined in the `$FABRIC_CFG_PATH/msp/config.yaml` file and contain a list of organizational units whose members are considered to be part of the organization represented by this MSP. This is particularly useful when you want to restrict the members of an organization to the ones holding an identity (signed by one of MSP designated CAs) with a specific Node OU role in it. For example, with node OU's you can implement a more granular endorsement policy that requires Org1 peers to endorse a transaction, rather than any member of Org1.

In order to use the Node OU roles, the "identity classification" feature must be enabled for the network. When using the folder-based MSP structure, this is accomplished by enabling "Node OUs" in the config.yaml file which resides in the root of the MSP folder:

```
NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: client
  PeerOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: peer
  AdminOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: admin
  OrdererOUIdentifier:
    Certificate: cacerts/ca.sampleorg-cert.pem
    OrganizationalUnitIdentifier: orderer
```

В примере выше есть 4 возможные `ROLES` Node OU  для MSP:

   * клиент
   * пир
   * администрация
   * ordering-служба

This convention allows you to distinguish MSP roles by the OU present in the CommonName attribute of the X509 certificate. The example above says that any certificate issued by cacerts/ca.sampleorg-cert.pem in which OU=client will identified as a client, OU=peer as a peer, etc. Starting with Fabric v1.4.3, there is also an OU for the orderer and for admins. The new admins role means that you no longer have to explicitly place certs in the admincerts folder of the MSP directory. Rather, the `admin` role present in the user's signcert qualifies the identity as an admin user.

These Role and OU attributes are assigned to an identity when the Fabric CA or SDK is used to `register` a user with the CA. It is the subsequent `enroll` user command that generates the certificates in the users' `/msp` folder.   

![MSP1c](./ca-msp-visualization.png)

The resulting ROLE and OU attributes are visible inside the X.509 signing certificate located in the `/signcerts` folder. The `ROLE` attribute is identified as `hf.Type` and  refers to an actor's role within its organization, (specifying, for example, that an actor is a `peer`). See the following snippet from a signing certificate shows how the Roles and OUs are represented in the certificate.

![MSP1d](./signcert.png)

**Note:** For Channel MSPs, just because an actor has the role of an administrator it doesn't mean that they can administer particular resources. The actual power a given identity has with respect to administering the system is determined by the _policies_ that manage system resources. For example, a channel policy might specify that `ORG1-MANUFACTURING` administrators, meaning identities with a role of `admin` and a Node OU of  `ORG1-MANUFACTURING`, have the rights to add new organizations to the channel, whereas the `ORG1-DISTRIBUTION` administrators have no such rights.

Finally, OUs could be used by different organizations in a consortium to distinguish each other. But in such cases, the different organizations have to use the same Root CAs and Intermediate CAs for their chain of trust, and assign the OU field to identify members of each organization. When every organization has the same CA or chain of trust, this makes the system more centralized than what might be desirable and therefore deserves careful consideration on a blockchain network.

## MSP Structure

Let's explore the MSP elements that render the functionality we've described so far.

A local MSP folder contains the following sub-folders:

![MSP6](./membership.diagram.6.png)

*The figure above shows the subfolders in a local MSP on the file system*

* **config.yaml:**  Used to configure the identity classification feature in Fabric by enabling "Node OUs" and defining the accepted roles.

* **cacerts:** This folder contains a list of self-signed X.509 certificates of the Root CAs trusted by the organization represented by this MSP. There must be at least one Root CA certificate in this MSP folder.

  This is the most important folder because it identifies the CAs from which all other certificates must be derived to be considered members of the
  corresponding organization to form the chain of trust.

* **intermediatecerts:** This folder contains a list of X.509 certificates of the Intermediate CAs trusted by this organization. Each certificate must be signed by one of the Root CAs in the MSP or by any Intermediate CA whose issuing CA chain ultimately leads back to a trusted Root CA.

  An intermediate CA may represent a different subdivision of the organization (like `ORG1-MANUFACTURING` and `ORG1-DISTRIBUTION` do for `ORG1`), or the
  organization itself (as may be the case if a commercial CA is leveraged for the organization's identity management). In the latter case intermediate CAs
  can be used to represent organization subdivisions. [Here](../msp.html) you may find more information on best practices for MSP configuration. Notice, that
  it is possible to have a functioning network that does not have an Intermediate CA, in which case this folder would be empty.

  Like the Root CA folder, this folder defines the CAs from which certificates must be issued to be considered members of the organization.

* **admincerts (Deprecated from Fabric v1.4.3 and higher):** This folder contains a list of identities that define the actors who have the role of administrators for this organization. In general, there should be one or more X.509 certificates in this list.

  **Note:** Prior to Fabric v1.4.3, admins were defined by explicitly putting certs in the `admincerts` folder in the local MSP directory of your peer. **With Fabric v1.4.3 or higher, certificates in this folder are no longer required.** Instead, it is recommended that when the user is registered with the CA, that the `admin` role is used to designate the node administrator. Then, the identity is recognized as an `admin` by the Node OU role value in their signcert. As a reminder, in order to leverage the admin role, the "identity classification" feature must be enabled in the config.yaml above by setting "Node OUs" to `Enable: true`. We'll explore this more later.

  And as a reminder, for Channel MSPs, just because an actor has the role of an administrator it doesn't mean that they can administer particular resources. The actual power a given identity has with respect to administering the system is determined by the _policies_ that manage system resources. For example, a channel policy might specify that `ORG1-MANUFACTURING` administrators have the rights to add new organizations to the channel, whereas the `ORG1-DISTRIBUTION` administrators have no such rights.

* **keystore: (private Key)** This folder is defined for the local MSP of a peer or orderer node (or in a client's local MSP), and contains the node's private key. This key is used to sign data --- for example to sign a transaction proposal response, as part of the endorsement phase.

  This folder is mandatory for local MSPs, and must contain exactly one private key. Obviously, access to this folder must be limited only to the identities of users who have administrative responsibility on the peer.

  The **channel MSP** configuration does not include this folder, because channel MSPs solely aim to offer identity validation functionalities and not signing abilities.

  **Note:** If you are using a [Hardware Security Module(HSM)](../hsm.html) for key management, this folder is empty because the private key is generated by and stored in the HSM.

* **signcert:** For a peer or orderer node (or in a client's local MSP) this folder contains the node's **signing key**. This key matches cryptographically the node's identity included in **Node Identity** folder and is used to sign data --- for example to sign a transaction proposal response, as part of the endorsement phase.

  This folder is mandatory for local MSPs, and must contain exactly one public key. Obviously, access to this folder must be limited only to the identities of users who have  administrative responsibility on the peer.

  Configuration of a **channel MSP** does not include this folder, as channel MSPs solely aim to offer identity validation functionalities and not signing abilities.

* **tlscacerts:** This folder contains a list of self-signed X.509 certificates of the Root CAs trusted by this organization **for secure communications between nodes using TLS**. An example of a TLS communication would be when a peer needs to connect to an orderer so that it can receive ledger updates.

  MSP TLS information relates to the nodes inside the network --- the peers and the orderers, in other words, rather than the applications and administrations that consume the network.

  There must be at least one TLS Root CA certificate in this folder. For more information about TLS, see [Securing Communication with Transport Layer Security (TLS)](../enable_tls.html).

* **tlsintermediatecacerts:** This folder contains a list intermediate CA certificates CAs trusted by the organization represented by this MSP **for secure communications between nodes using TLS**. This folder is specifically useful when commercial CAs are used for TLS certificates of an organization. Similar to membership intermediate CAs, specifying intermediate TLS CAs is optional.

* **operationscerts:** This folder contains the certificates required to communicate with the [Fabric Operations Service](../operations_service.html) API.

A channel MSP includes the following additional folder:

* **Revoked Certificates:** If the identity of an actor has been revoked, identifying information about the identity --- not the identity itself --- is held in this folder. For X.509-based identities, these identifiers are pairs of strings known as Subject Key Identifier (SKI) and Authority Access Identifier (AKI), and are checked whenever the certificate is being used to make sure the certificate has not been revoked.

  This list is conceptually the same as a CA's Certificate Revocation List (CRL), but it also relates to revocation of membership from the organization. As a result, the administrator of a channel MSP can quickly revoke an actor or node from an organization by advertising the updated CRL of the CA. This "list of lists" is optional. It will only become populated as certificates are revoked.

If you've read this doc as well as our doc on [Identity](../identity/identity.html), you
should now have a pretty good grasp of how identities and MSPs work in Hyperledger Fabric.
You've seen how a PKI and MSPs are used to identify the actors collaborating in a blockchain
network. You've learned how certificates, public/private keys, and roots of trust work,
in addition to how MSPs are physically and logically structured.

<!---
Licensed under Creative Commons Attribution 4.0 International License https://creativecommons.org/licenses/by/4.0/
-->
