# Чейнкод как внешняя служба

Fabric v2.0 поддерживает развертывание и запуск чейнкода вне Fabric, что позволяет пользователям управлять запущенным чейнкодом без взаимодействия с пиром.
Это упрощает развертывание чейнкода в облаке. Вместо того, чтобы собирать и запускать чейнкод на каждом пире, чейнкод может быть запущен как служба вне Fabric.
Это использует механизм Fabric v2.0 [External Builder and Launcher](./cc_launcher.html).

До выпуска этого механизма, содержимое чейнкод-пакета должно было быть набором исходников на конкретном языке программирования, которые собирались и запускались как бинарный файл с чейнкодом.
Новый механизм позволяет пользователям менять процесс сборки.
Процесс сборки позволяет указать адрес сервера, на котором исполняется чейнкод.
Таким образом, пакет может состоять всего лишь из адреса и TLS-артефактов для безопасного соединения.
Использование TLS опционально, но крайне рекомендуется.

Эта статья описывает, как настроить чейнкод как внешнюю службу:

* [Пакетирование чейнкода](#пакетирование-чейнкода)
* [Настройка пира](#настройка-пира)
* [Пример программ External builder and launcher](#пример-программ-external-builder-and-launcher)
* [Реализация чейнкода как внешней службы](#реализация-чейнкода-как-внешней-службы)
* [Развертывание чейнкода](#развертывание-чейнкода)
* [Исполнение чейнкода как внешней службы](#исполнение-чейнкода-как-внешней-службы)

**Важно:** Это продвинутая опция Fabric, которая, скорее всего, потребует особой сборки peer image.
Например, последующие примеры используют `go` и `bash`, не включенные в официальный `fabric-peer` image.

## Пакетирование чейнкода

С приходом жизненного цикла чейнкода Fabric v2.0, чейнкод теперь [пакетируется](./cc_launcher.html#chaincode-packages) 
и устанавливается в формате `.tar.gz`. Следующий архив `myccpackage.tgz` демонстрирует требуемую структуру пакета:

```sh
$ tar xvfz myccpackage.tgz
metadata.json
code.tar.gz
```

Пакет чейнкода используется, чтобы донести процессу external builder and launcher две вещи:
* идентифицировать чейнкод как внешнюю службу. Секция `bin/detect` описывает, как это сделать через файл `metadata.json`
* передать информацию об адресе чейнкода через файл `connection.json`. Секция `bin/run` описывает файл `connection.json`

Есть много способов донести эту информацию. Примеры программ из секции [External builder and launcher sample scripts](#external-builder-and-launcher-sample-scripts) 
показывают простой способ это сделать.
Как пример такой гибкости, рассмотрим пакетирование файлов с индексами couchdb (см. [Add the index to your chaincode folder](couchdb_tutorial.html#add-the-index-to-your-chaincode-folder)). 
Примеры программ ниже описывают способ пакетирования файлов в `code.tar.gz`.

```
tar cfz code.tar.gz connection.json metadata
tar cfz $1-pkg.tgz metadata.json code.tar.gz
```

## Настройка пира

В этой секции мы рассмотрим конфигурацию, нужную, чтобы
* выявить, определяет ли чейнкод-пакет внешнюю чейнкод-службу
* создать `connection.json`

### Настройка core.yaml

Пусть программы buildpack находятся в директории `bin` так:
```
    <полный путь в окружении пира>
    └── bin
        ├── build
        ├── detect
        └── release
```

Включите в секцию `chaincode` файла `core.yaml` элемент конфигурации `externalBuilders`:

```yaml
externalBuilders:
     - name: myexternal
       path: <полный путь в окружении пира>   
```

### Пример программ External builder and launcher

Для лучшего понимания, что должен содержать buildpack для работы с внешней чейнкод-службой, эта секция содержит примеры `bin/detect` `bin/build`, `bin/release`, и `bin/run`.

**Важно:** Примеры используют команду `jq` для парсинга json. Чтобы проверить, установлена ли она, выполните `jq --version`. Иначе установите `jq` или измените код.

#### `bin/detect`

Программа  `bin/detect script` определяет, должен ли этот buildpack использоваться для сборки и исполнения чейнкод-пакета.
В нашем случае скрипт определяет, установлено ли поле `type` на `external` в файле `metadata.json`:

```json
{"path":"","type":"external","label":"mycc"}
```

Пир вызывает `detect` с двумя аргументами:

```
bin/detect CHAINCODE_SOURCE_DIR CHAINCODE_METADATA_DIR
```

Пример `bin/detect`:

```sh

#!/bin/bash

set -euo pipefail

METADIR=$2
# проверить, что поле "type" установлено на "external"
if [ "$(jq -r .type "$METADIR/metadata.json")" == "external" ]; then
    exit 0
fi

exit 1

```

#### bin/build

В нашем случае `build` предполагает, что `code.tar.gz` содержит `connection.json`, которую он копирует в `BUILD_OUTPUT_DIR`.
Пирв вызывает `build` с тремя аргументами:

```
bin/build CHAINCODE_SOURCE_DIR CHAINCODE_METADATA_DIR BUILD_OUTPUT_DIR
```

Пример `bin/build`:

```sh

#!/bin/bash

set -euo pipefail

SOURCE=$1
OUTPUT=$3

# внешним чейнкодам необходимо иметь connection.json в директории с исходниками
if [ ! -f "$SOURCE/connection.json" ]; then
    >&2 echo "$SOURCE/connection.json not found"
    exit 1
fi

cp $SOURCE/connection.json $OUTPUT/connection.json

if [ -d "$SOURCE/metadata" ]; then
    cp -a $SOURCE/metadata $OUTPUT/metadata
fi

exit 0

```

#### bin/release

В случае чейнкода как внешней службы, `bin/release` передает `connection.json` пиру, помещая его в `RELEASE_OUTPUT_DIR`. Файл `connection.json` имеет следующую структуру:

* **address** - адрес chaincode server endpoint, доступного пиру. Должен быть указан в формате “<host>:<port>”.
* **dial_timeout** - интервал ожидания соединения. Указывается в виде строки как число с единицей измерения (например "10s", "500ms", "1m"). По умолчанию “3s”.
* **tls_required** - true или false. Если false, то поля "client_auth_required", "client_key", "client_cert" и "root_cert" не требуются. По умолчанию “true”.
* **client_auth_required** - если true, то поля "client_key" и "client_cert" не требуются. По умолчанию "false". Поле игнорируется, если tls_required - false.
* **client_cert** - TLS-сертификат клиента в формате PEM.
* **client_key** - TLS-ключ клиента в формате PEM.
* **root_cert** - Корневой TLS-сертификат в формате PEM для сервера (пира).

Пример:

```json
{
  "address": "your.chaincode.host.com:9999",
  "dial_timeout": "10s",
  "tls_required": "true",
  "client_auth_required": "true",
  "client_key": "-----BEGIN EC PRIVATE KEY----- ... -----END EC PRIVATE KEY-----",
  "client_cert": "-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----",
  "root_cert": "-----BEGIN CERTIFICATE---- ... -----END CERTIFICATE-----"
}
```

Как отмечено в секции `bin/build`, этот пример предполагает, что чейнкод-пакет непосредственно содержит файл `connection.json`,
который `build` копирует в `BUILD_OUTPUT_DIR`. 

Пир вызывает `release` с двумя аргументами:

```
bin/release BUILD_OUTPUT_DIR RELEASE_OUTPUT_DIR
```

Пример `bin/release`:


```sh

#!/bin/bash

set -euo pipefail

BLD="$1"
RELEASE="$2"

if [ -d "$BLD/metadata" ]; then
   cp -a "$BLD/metadata/"* "$RELEASE/"
fi

# Для внешних чейнкод-служб ожидается, что артефакты будут помещены в "$RELEASE"/chaincode/server
if [ -f $BLD/connection.json ]; then
   mkdir -p "$RELEASE"/chaincode/server
   cp $BLD/connection.json "$RELEASE"/chaincode/server

   # если tls_required - true, скопировать TLS файлы (полный путь к этим файлам - "$RELEASE"/chaincode/server/tls)

   exit 0
fi

exit 1
```    

## Реализация чейнкода как внешней службы

Модель чейнкода как внешней службы на текущий момент поддерживается только GO chaincode shim. 

Начиная с Fabric v2.0, GO shim API включает тип `ChaincodeServer`, который необходимо использовать для создания чейнкод-сервера.
`Invoke` и `Query` APIs не затронуты. Разработчики должны реализовать `shim.ChaincodeServer` API, а затем собрать чейнкод и исполнить его в произвольном внешнем окружении.
Пример чейнкода:

```go

package main

import (
        "fmt"

        "github.com/hyperledger/fabric-chaincode-go/shim"
        pb "github.com/hyperledger/fabric-protos-go/peer"
)

type SimpleChaincode struct {
}

func (s *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
        // код init
}

func (s *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
        // код invoke
}

//ВАЖНО - параметры, такие как ccid и информация о endpoint встроены в целях демонстрации. Они могут быть переданы в чейнкод некоторыми стандартными способами.
func main() {
       // ccid присвается чейнкоду при установке (с помощью команды “peer lifecycle chaincode install <package>”)
        ccid := "mycc:fcbf8724572d42e859a7dd9a7cd8e2efb84058292017df6e3d89178b64e6c831"

        server := &shim.ChaincodeServer{
                        CCID: ccid,
                        Address: "myhost:9999"
                        CC: new(SimpleChaincode),
                        TLSProps: shim.TLSProperties{
                                Disabled: true,
                        },
                }
        err := server.Start()
        if err != nil {
                fmt.Printf("Error starting Simple chaincode: %s", err)
        }
}
```
Этот код использует новый shim API `shim.ChaincodeServer` со следующими параметрами чейнкод-службы:

* **CCID** (string) - CCID должен совпадать с именем чейнкод-пакета на пире. Это `CCID`, возвращенный командой CLI `peer lifecycle chaincode install <package>`.
  Он может быть получен после установки чейнкода командой `peer lifecycle chaincode queryinstalled`.
* **Address** (string) - Адрес прослушки чейнкод-сервера.
* **CC** (Chaincode) -  CC это чейнкод, обрабатывающий Init и Invoke
* **TLSProps** (TLSProperties) - TLSProps это параметры TLS, передающиеся чейнкод-серверу.
* **KaOpts** (keepalive.ServerParameters) -  опции KaOpts keepalive, если nil, то используются разумные стандартные значения.

Затем соберите чейнкод подходящим для вашего окружения способом.

## Развертывание чейнкода

Когда Go-чейнкод готов к развертыванию, вы можете пакетировать чейнкод (см. [Пакетирование чейнкода](#пакетирование-чейнкода))
и развернуть чейнкод (см. статью [Жизненный цикл чейнкода Fabric](./chaincode_lifecycle.html)).

## Исполнение чейнкода как внешней службы

Создайте чейнкод как указано в секции [Реализация чейнкода как внешней службы](#реализация-чейнкода-как-внешней-службы). 
Исполните собранную программу в окружении на ваш выбор, таком как Kubernetes или напрямую как процесс на сервере пира.

При использовании чейнкода как внешней службы, больше не требуется устанавливать его на каждый пир.
После того, как чейнкод развертан, вы можете продолжить стандартный процесс сохранения определения чейнкода в канал и вызова чейнкода.

<!---
Licensed under Creative Commons Attribution 4.0 International License https://creativecommons.org/licenses/by/4.0/
-->
