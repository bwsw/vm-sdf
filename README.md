# VM Services Deployment Framework

VM-SDF представляет собой экосистему, предназначенную для расширенного конфигурирования виртуальных машин перед развертыванием. Под расширенным конфигурированием понимается выбор приложения из библиотеки приложений и настройка данного приложения с помощью конфигурационных переменных аналогично тому, как это сделано в DC/OS.

При создании виртуальной машины в CSUI пользователь может выбрать создание VM из шаблона, ISO. VM-SDF добавляет третий вариант &ndash; каталог приложений. 

В третьем случае пользователь может задать параметры для развертываемого в виртуальной машине приложения (более обобщенно &ndash; для группы приложений в группе виртуальных машин) с помощью формы, которая генерируется для выбранного из каталога приложения.

В форме создания VM пользователь должен иметь возможность выбрать установку приложения и просматривать все приложения, которые в ней доступны, осуществлять фильтрацию приложений. Для выбранного приложения пользователь задает конфигурацию с помощью конфигурационных переменных, сгенерированная в итоге конфигурация передается в виртуальную машину посредством механизма `USERDATA`, который [поддерживает](https://cloudstack.apache.org/api/apidocs-4.11/apis/deployVirtualMachine.html) CloudStack. Поскольку размер блока конфигурации, передааемый в `USERDATA` в некоторых случаях может быть большим, рекомендуется использовать метод POST для вызова `deployVirtualMachine`.

Все определения приложений, компилируются специальным сценарием сборки в единый YaML-документ, который, в свою оцередь, загружается на сервер и проксируется приложением CSUI, при этом обращение к данному документу должно производиться без кэширования, чтобы при его обновлении, CSUI мог текущие, а не устаревшие данные.

VM-SDF поддерживает три способа передачи параметров в виртуальную машины с помощью `USERDATA`:
 * текстовый в виде фрагмента YaML;
 * с использованием CS-KVS
 * с использованием CS-KVS и CS-OTA

Далее вышеуказанные механизмы будут описаны подробно.

Экосистема VM-SDF должна быть построена на основе динамически генерируемых на основе YaML форм. Каждая такая форма определяется для каждого приложения и позволяет указать как параметры, которые могут задаваться пользователем, запрашиваемые приложением разрешения, и другие свойства, которые определяют значимые элементы конфигурации приложения.

Пользователь редактирует редактируемые параметры, соглашается с запрашиваемыми привилегиями приложения и нажимает "ОК", что генерирует целевой YaML-фрагмент конфигураци, который будет тем или иным способом передан в `USERDATA`.

В самом простом случае VM-SDF может использоваться для развертывания приложения в рамках одного узла, рассмотрим пример LAMP сервер:

```yaml
lnmp-10:
  mode: raw
  manifest-schema: /schemes/vm-sdf-manifest-1.0.yaml
  userdata-schema: /schemes/vm-sdf-userdata-1.0.yaml
  name: LNMP Server
  description:
    default:
      - Linux Web LxMP Server
      - "Components: Linux, NGINX, MySQL, PHP FPM"
      - "Deployed and managed by Provider.COM"
      - "MySQL Version: 5.3"
      - "PHP Versions: 5.4, 7.0.1"
      - "Nginx Version: 1.0.1"
  icon-32x32: /assets/lnmp-10-32x32.png
  icon-256x256: /assets/lnmp-10-256x256.png
  tags:
    - lamp
    - nginx
    - mysql
    - www
    - hosting
  version: 1.0
  icon: /assets/lnmp-10.png
  license: 
    name: Apache License, Version 2.0
    description: Свободная лицензия без ограничений на использование
    url: http://apache.org/license
  source: https://reposerver.com/applications/L/lnmp-10/lnmp-10.tar.gz
  template:
    match-tags:
      tag1: value
      tag2: value
  minimal-root-disk-size: 10
  service-offering:
    filters:
      resources:
        minimal-ram: 1024
        minimal-cores: 2
        minimal-frequency: 1000
  network-security:
    ingress:
      - rule:
          protocol: icmp
          type: -1
          code: -1
          cidr: 0.0.0.0/0
      - rule:
          protocol: tcp
          ports: 
            - 80
            - 22
            - 1025-5555
          cidr: 0.0.0.0/0
    egress:
      - rule:
          protocol: icmp
          type: -1
          code: -1
          cidr: 0.0.0.0/0
      - rule:
          protocol: tcp
          ports:
            - 1-65535
          cidr: 0.0.0.0/0
  features-requested:
    - application-db
    - logs
  deployment-progress:
    log: /var/log/deployment_log
    application-key: 'deployment-progress'
    application-deployment-complete-label: 'dafcbe0d4111d0d42378ec40545489ac'
  application-state:
    mysql-state:
      default: MySQL server state 
      ru_RU: Состояние сервера MySQL
    nginx-state:
      default: Nginx server state
      ru_RU: Состояние Nginx
  variables:
    domain-name:
      description:
        default: Domain name for the first Website
        ru_RU: Имя домена для создаваемого сайта
      type: DomainName
      default: example.com
    mysql-root-password:
      description:
        default: Password for MySQL root user
        ru_RU: Пароль root-пользователя MySQL
      type: Password
    mysql-database:
      description: 
        default: MySQL database for the website
        ru_RU: Имя базы данных для web-сайта
      type: String
      regex: '^[a-zA-Z][a-zA-Z_0-9]{1,15}$'
      default: db
    mysql-user-name:
      description: 
        default: Database User Name
        ru_RU: Имя пользователя базы данных
      type: String
      regex: '^[a-zA-Z][a-zA-Z_0-9]{1,15}$'
      default: www
    mysql-user-password:
      description: 
        default: Database User Password
        ru_RU: Имя пользователя базы данных   
      type: Password
    mysql-php-admin-install:
      description: 
        default: Install PhpMyAdmin?
        ru_RU: Устанавливать PhpMyAdmin?
      type: OneOf
      options:
        false:
          labels:
            default: No
            ru_RU: Нет
        true:
          labels:
            default: Yes
            ru_RU: Да
      default: false
    mysql-php-admin-https-port:
      active-if:
        predicate: and
        sources:
          mysql-php-admin-install: true
      description: 
        default: PhpMyAdmin HTTPS port
        ru_RU: HTTPS порт PhpMyAdmin
      type: Number
      integer_min: 1024
      integer_max: 65535
      default: 8443
    use-lets-encrypt:
      active-if:
        predicate: and
        sources:
          mysql-php-admin-install: true
      description: 
        default: Use Let's Encrypt certificate?
        ru_RU: Использовать Let's Encrypt?
      type: OneOf
      options:
        false:
          labels:
            default: No
            ru_RU: Нет
        true:
          labels:
            default: Yes
            ru_RU: Да
      default: false
```

В более сложных случаях, когда развертывается многоузловое приложение, первая машина, которая выполняет конфигурацию может выступать в качестве seed-узла, который создает дополнительные машины.

Все параметры, сконфигурированные на данном этапе пользователь может посмотреть в форме YaML-документа на дополнительной вкладке формы редактирования параметров.

Пример YaML, сгенерированного для некоторой формы:

```yaml
deployment-info:
  manifest-schema: /schemes/vm-sdf-manifest-1.0.yaml
  userdata-schema: /schemes/vm-sdf-userdata-1.0.yaml
  source: https://reposerver.com/applications/L/lnmp/lnmp-10.tar.gz
  mode: raw
  application-manifest: base64-encoded-manifest-for-application
  features:
    vm-db:
      endpoint: https://kvs.com/
      name: UUID1
      secret: XXXXXXX
    logs:
      endpoint: https://logs.com/
      name: VMUUID
      secret: XXXXXXX
  deployment-progress:
    log: /var/log/deployment_log
    vm-key: deployment-progress
  variables:
    domain-name: www.com
    mysql-root-password: XXXX
    mysql-database: db1
    mysql-user-name: username
    mysql-user-password: XXX
    mysql-php-admin-install: true
    mysql-php-admin-https-port: 8443
    use-lets-encrypt: true
```

## Способы передачи данных в USERDATA

Существует три способа передачи данных в `USERDATA`:
 * `raw` &ndash; данные передаются в форме `YaML` как есть;
 * `kvs` &ndash; в `USERDATA` передаются только данные доступа к `kvs`, а информация для развертывания приложения упаковывается уже в `kvs`;
 * `ota+kvs` &ndash; так же как и в случае (2), но `KV` не передается напрямую, а используется одноразовый токен `OTA` для `KV`.

Пример для `raw` отображен ранее, `raw` просто передает конфигурацию в формате `YaML` в виртуальную машину.

К недостатку `raw` можно отнести то, что виртуальная машина для получения доступа к обновленным данным должна быть перезапущена. Другой недостаток заключается в том, что машина получает доступ к `USERDATA` как есть, а, значит, в случае многопользовательской среды (хостинг) может быть получен доступ к секретной информации, которая должна оставаться приватной.

**kvs**. В этом режиме частное хранилище KV, автоматически создаваемое для виртуальной машины, используется для передачи параметров конфигурации, которые преобразуются из YaML в строки вида “a.b.c”  и помещаются в хранилище KV. в `USERDATA`, в этом случае передается только информация о самом хранилище:

```yaml
deployment-info:
  mode: kvs
  kvs:
    endpoint: https://kvs-uri.com/
    name: UUID1
    secret: XXXXXXX
```

**ota+kvs**. В этом режиме одноразовый токен `OTA` используется для безопасной передачи данных подключения к хранилищу KV, которое используется для передачи параметров конфигурации, как и в случае использования метода `KV`.

```yaml
deployment-info:
  mode: ota+kvs
  ota+kvs:
    endpoint: https://ota.com/
    token: XXXXXXX
```
  
## Типы переменных

> manifest-schema: /schemes/vm-sdf-1.0.yaml

Следующие типы переменных должны поддерживаться в формах:
 * Number &ndash; число;
 * String &ndash; строка;
 * DomainName &ndash; доменное имя;
 * IpAddress &ndash; интернет-адрес v4, v6;
 * URL &ndash; корректный URI;
 * Email &ndash; корректный E-mail;
 * Password &ndash; поле ввода пароля;
 * OneOf &ndash; радиокнопки или Dropdown; 
 * Some &ndash; набор чекбоксов;
 * ServiceOffering &ndash; выбор сервисного предложения;
 * Section &ndash; разделитель разделов.

### Общие поля для любой переменной

```yaml
field-name:
  label:
    default: Name
    ru_RU: Имя
    en_US: Name
    ….
  description:
    default: Tooltip text
    ru_RU: …
    en_US
  type: TYPE
  active-if:
  predicate: and | or
    sources:
      field-name-2: X
      field-name-3:
        - Y
        - Z
    default: value
    is-volatile: true | false
```

* **label** &ndash; то, что отображается в качестве имени поля;
* **description** &ndash; то, что отображается в качестве tooltip при наведении на поле;
* **type** &ndash; тип поля, определяющий расширенные настройки;
* **active-if** &ndash; означает, что данная переменная активна для изменения тогда, когда выполняется предикат;
* **default** &ndash; значение по-умолчанию;
* **is-volatile** &ndash; атрибут, который определяет может ли данное поле измениться после развертывания приложения.

### Атрибут Label

Атрибут может задаваться в двух формах:

```yaml
label: value
```

или

```yaml
label:
  default: value
  locale1: value
  locale2: value
```

В случае первого способа определения `value` используется как `default`, а в случае второго способа определения `label.default` &ndash; обязательный атрибут, Атрибуты других локалей опциональны.

### Атрибут Description

Атрибут может задаваться в двух формах:

```yaml
description: value
```

или

```yaml
description:
  default: value
  locale1: value
  locale2: value
```

В случае первого способа определения `value` используется как `default`, а в случае второго способа определения `description.default` &ndash; обязательный атрибут, Атрибуты других локалей опциональны.

### Атрибут Active-If

**active-if** используется для условной активации части переменных в зависимости от значений других переменных. 

Для комбинирования используется вложенный атрибут `active-if.predicate`, который определяет каким образом происходит построение логческого выражения на основе вложенных атрибутов `active-if.sources.*` (активирующие переменные).

Если `active-if.predicate` явно не задан, то считается 'and'.

В настоящее время планируется поддержка двух способов объединения:
* `and`, который активирует переменную, если активирующие переменные все одновременно имеют указанные значения;
* `or`, который активирует переменную, если хотя бы одна активирующая переменная имеет указанное значение.

В качестве значения активирующей переменной можно установить одно значение или список. В случае списка, активация происходит при совпадении значения с любым из элементов списка.

В случае, если переменная неактивна в рамках текущих активирующих зависимостей, в результирующем YaML ей сопоставляется значение YaML `NULL`.

Должна поддерживаться многоуровневая активация: если активированная переменная имеет значение по-умолчанию, которое достаточно для активации новых переменных, они должны быть активированы тоже.

### Атрибут Volatile

Атрибут определяет может ли параметр изменяться после первоначальной установки. Это необходимо для разделения параметров, которые являются модифицируемыми после развертывания или не могут изменяться.

По-умолчанию, переменные не являются `volatile`, то есть устанавливаются только один раз.

## Переменная типа Number

Дополнительно к общим атрибутам добавляются следующие атрибуты:

* **minimum** &ndash; минимально допустимое значение, по-умолчанию не ограничен;
* **maximum** &ndash; максимально допустимое значание, по-умолчанию не ограничен;
* **integer** &ndash; целое или float, по умолчанию `true`, то есть целое число.

## Переменная типа String

Дополнительно к общим атрибутам добавляются следующие атрибуты:

* **minimum-length** &ndash; минимально допустимое количество символов, по-умолчанию 0;
* **maximum-length** &ndash; максимально допустимое количество символов, по-умолчанию 255;
* **multiline** &ndash; используется ли однострочный компонент или многострочный компонент типа `textarea`, по-умолчанию `false`;
* **base64** &ndash; осуществлять ли кодирование введенной информации в `base64`, по-умолчанию, если не задано иное, принимает то же значени, что и атрибут `multiline`;
* **regex** &ndash; регулярное выражение, используемое для оценки корректности.

В случае `multiline: true, base64: false`, в целевом YaML для данной переменной генерируется текст вида:

```yaml
varname:
  - line1
  - line2
  - line3
  - line4
  - ...
```

Не забываем про специальные символы в YaML, генерировать лучше с помощью DOM-библиотеки, а не вручную.

## Переменная типа DomainName

Однострочное поле типа String с валидатором корректного FQDN.

## Переменная типа IpAddress

Поле ввода IP-адреса в форме строки. Дополнительные параметры:

* **version** &ndash; версия протокола, может иметь значения `4` и/или `6`, по-умолчанию `4`;
* **network-address** &ndash; должен пользователь вводить адрес в формате сети CIDR (`ip/prefix`) или для одиночного хоста, по-умолчанию `false`;

```yaml
field:
  type: IpAddress
  network-address: true
  version:
    - 4
    - 6

field4:
  type: IpAddress
  network-address: false
  version:
    - 4

field6:
  type: IpAddress
  cidr: true
  version:
    - 6
```

## Переменная типа URL

Поле ввода URL. Дополнительные параметры:

* **scheme** &ndash; схема протокола (https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml), по-умолчанию `[http, https, ftp]`;
* **maximum-length** &ndash; максимальная длина URL, по-умолчанию `1024`;

```yaml
scheme:
  - http
  - https
```

Пользователь может вводить корректные URL в форме: `scheme://name:password@host.com:port/path/to/resource`

## Переменная типа Email

Однострочная строка с валидатором корректного E-mail.

## Переменная типа Password

Пароль с подтверждением и возможностью автогенерации. Дополнительные параметры:

* **mimimum-length** &ndash; минимальное количество символов, по-умолчанию `5`;
* **maximum-length** &ndash; максимальное количество символов, по-умолчанию, `10`;
* **regex** &ndash; регулярное выражение, по-умолчанию не задано.

Компонент представляет собой классическое задание пароля с подтверждением, дополнительные функции &ndash; автозаполнение (генерация, которая работает только при отсутствии установленного валидатора) и кнопка “показать пароль”.

## Переменная типа OneOf

Выбор одного значения из нескольких. Дополнительные параметры:

* **options** &ndash; опции, между которыми производится выбор.
* **visual-mode** &ndash; визуальный вид компонента, поддерживается `dropdown`, `verical-radio-group`, `horizontal-radio-group`, по-умолчанию `vertical-radio-group`.

```yaml
field-name:
  options:
    option1:
      label:
        default: label1
        ru_RU: буу
    option2:
      label:
        default: label2
        ru_RU: буу2
```

Вложенный атрибут `label` может задаваться как в полной форме (пример выше), так и в упрощенной форме без локализации:

```yaml
label: value
```

## Переменная типа Some

Выбор нескольких значений. Дополнительные параметры:
* **options** &ndash; опции, аналогично радио.
* **defaults** &ndash; список, выбранных по-умолчанию компонентов. Унаследованное значение `default` так же может использоваться для выбора элементов, выбранных по умолчанию. 

```yaml
defaults:
  - abc
  - def
```

У опции может быть указан специальный атрибут `options.*.mandatory`, который означает, что пользователь не может отменить выбор данного атрибута и он всегда выбран.

```yaml
options:
  option1:
    mandatory: true
    label: This is always selected
  option2:
    label: This is optionally selected
defaults:
  - option1
  - option2
```

## Переменная типа ServiceOffering

Выбор сервисного предложения из доступных. Выбор SO производится аналогичным образом как и при обычном создании VM. Дополнительные параметры:

* **filters** &ndash фильтрующий атрибут, определяющий допустимые SO.

Все SO, которые окажутся попадающими под критерии любого из фильтров участвуют в выборе. Custom SO в случае наличия фильтра по ресурсам отбирается еще после задания конкретного количества ресурсов. 

**В каждом фильтре** могут встречаться от одного до всех критериев фильтрации:

* `filters.*.uuids` &ndash; фильтрация по совпадающим `UUID`;
* `filters.*.resources` &ndash; фильтрация по превышению минимально требуемых ресурсов, включает от одного до трех вложенных атрибутов `minimal-ram`, `minimal-cores`, `minimal-frequency`.
* `filters.*.regex` &ndash; фильтрация по имени SO с помощью regex.

OR-like фильтры:

```yaml
filters:
  filter1:
    uuids:
      - uuid1
      - uuid2
  filter2:
    resources:
      minimal-ram: 1024
      minimal-cores: 2
      minimal-frequency: 1000
  filter3:
    name-regex: xxx
```

AND-like фильтр:

```yaml
filters:
  filter1:
    uuids:
      - uuid1
      - uuid2
    resources:
      minimal-ram: 1024
      minimal-cores: 2
      minimal-frequency: 1000
    name-regex: xxx
```

## Переменная типа Section 

Вспомогательное поле, использующееся для разделения блоков переменных и для активации блоков по зависимостям. Данная переменная отображается как заголовок раздела (HTML H*).

```yaml
mysql-mode:
    label: MySQL mode
    description: Specify how MySQL will be used
    type: OneOf
    options:
        install:
            labels: Install MySQL
        use:
            labels: Use Installed MySQL
    default: install

mysql-install-section:
    label: MySQL Installation Options
    description: Fill in important parameters for MySQL installation
    type: Section
    active-if:
        sources:
            mysql-mode: install
    default: true

mysql-install-root-password:
  label: MySQL Root Password
  description: Fill in the password for MySQL installation
  type: Password
  active-if:
    sources:
      mysql-install-section: true

mysql-use-section:
  label: Installed MySQL Options
  description: Fill in important parameters for MySQL installation
  type: Section
  active-if:
    sources:
      mysql-mode: use
  default: true

mysql-use-root-password:
  label: MySQL Root Password
  description: Fill in the password for MySQL installation
  type: Password
  active-if:
    sources:
      mysql-use-section: true
```

В приведенном выше примере активация секций и полей в этих секций выполняется в каскадном режиме после указания mysql-mode.

## Прочие управляемые атрибуты манифеста приложения

> manifest-schema: /schemes/vm-sdf-1.0.yaml

```yaml
template:
  match-tags:
    tag1: value
    tag2: value
minimal-root-disk-size: 10
service-offering:
  filters:
    resources:
      minimal-ram: 1024
      minimal-cores: 2
      minimal-frequency: 1000
```

Атрибут `template` определяет совместимые шаблоны. Шаблоны, среди которых может делать выбор пользователь отбираются с помощью соответствия тегов. Пользователю предлагается выбрать шаблон с поомощью диалога, если доступен более чем один шаблон, либо просто показывается выбранный шаблон, если он единственный. После выбора для шаблона отображается имя и описание.

Атрибут `minimal-root-disk-size` используется для того, чтобы пользователь мог задать размер root-диска для развертываемого приложения. Атрибут ограничивает размер снизу.

Атрибут `servuce-offering` используется для выбора предложения, которое будет использоваться при развертывании. Атрибут реализуется точно так же, как и переменная типа `ServiceOffering`, описанная ранее.

## Неуправляемые атрибуты манифеста приложения

> manifest-schema: /schemes/vm-sdf-1.0.yaml

```yaml
  mode: raw
  version: 1.0
  source: https://reposerver.com/applications/L/lnmp-10/lnmp-10.tar.gz
  manifest-schema: /schemes/vm-sdf-manifest-1.0.yaml
  userdata-schema: /schemes/vm-sdf-userdata-1.0.yaml
  name: LNMP Server
  description:
    default:
      - Linux Web LxMP Server
      - "Components: Linux, NGINX, MySQL, PHP FPM"
      - "Deployed and managed by Provider.COM"
      - "MySQL Version: 5.3"
      - "PHP Versions: 5.4, 7.0.1"
      - "Nginx Version: 1.0.1"
  icon-32x32: /assets/lnmp-10-32x32.png
  icon-256x256: /assets/lnmp-10-256x256.png
  tags:
    - lamp
    - nginx
    - mysql
    - www
    - hosting
  license: 
    name: Apache License, Version 2.0
    description: Свободная лицензия без ограничений на использование
    url: http://apache.org/license
  features-requested:
    vm-key-value-storage: true
    log-storage: true
    use-ota: true
  deployment-progress:
    log: /var/log/deployment_log
  application-state:
    mysql-state:
      default: MySQL server state 
      ru_RU: Состояние сервера MySQL
    nginx-state:
      default: Nginx server state
      ru_RU: Состояние Nginx
    vm-key: 'deployment-progress'
  network-security:
    ingress:
      - rule:
          protocol: icmp
          type: -1
          code: -1
          cidr: 0.0.0.0/0
      - rule:
          protocol: tcp
          ports: 
            - 80
            - 22
            - 1025-5555
          cidr: 0.0.0.0/0
    egress:
      - rule:
          protocol: icmp
          type: -1
          code: -1
          cidr: 0.0.0.0/0
      - rule:
          protocol: tcp
          ports:
            - 1-65535
          cidr: 0.0.0.0/0
```

### Атрибут `mode`

Атрибут используется для определения способа передачи данных в виртуальную машину. Ранее были описаны поддерживаемые значения данного атрибута:
* `raw`
* `kvs`
* `ota+kvs`.

### Атрибут `version`

Определяет версию самого приложения. Используется для отслеживания проблем и ограничений развертывания.

### Атрибут `source`

Определяет URI архива, в котором содержатся все необходимые данные, чтобы развернуть приложение на основании сгенерированного конфигурационного файла.

### Атрибут `manifest-schema`

Атрибут задает URI схемы, которой соответствует манифест приложения. Файл схемы &ndash; YaML-схема. При генерации каталога все приложения проверяются на сооветствие схеме. В случае добавления новых атрибутов могут генерироваться новые схемы и меняться их версия.

### Атрибут `userdata-schema`

Атрибут задает URI схемы, которая используется для валидации сгенерированных из манифеста данных. Данную схему испольузет UI для самотестирования и приложение развертывания внутри VM для анализа корректности.

### Атрибут `name`

Имя приложения, которое видит пользователь в каталоге. Имя может задаваться в упрощенном формате `name: AppName` или в развернутом с поддержкой локализации.

### Атрибут `description`

Расширенное описание приложения, которое пользователь видит в каталоге. Может задаваться как в упрощенном формате `description: App Description`, так и в расширенном формате с локалями. Должно поддерживаться распознование многострочных описаний. Атрибут опционален.

### Атрибут `icon-32x32`

Иконка продукта в каталоге продуктов, используется при невыбранном продукте. Атрибут опционален, если не задан, используется системная иконка.

### Атрибут `icon-256x256`

Иконка продукта в каталоге продуктов, используется при выбранном продукте. Атрибут опционален, если не задан, используется системная иконка.

### Атрибут `tags`

Атрибут, задает тэги продукта, которые могут использоваться для поиска или перехода к похожему продукту по тегу.

### Атрибут `license`

Определяет лицензию приложения. Содержит атрибуты `name` (имя лицензии), `description` (описание лицензии), `url` (ссылку на текст лицензии).

При задании `description` пользователь использует однострочное описание в упрощенном или локализованном виде.

### Атрибут `features-requested`

Атрибут используется для определения того, какие системные параметры будут переданы в сгенерированный конфигурационный файл. Для каждого вложенного атрибута пользователь должен явно подтвердить согласие на его передачу в конфигурационный файл.

```yaml
  features-requested:
    - cloudstack-api
    - application-db
    - vm-db
    - temporary-db
    - logs
```

Атрибут `cloudstack-api` определяет будет ли в целевом конфигурационном YaML-фрагменте присутствовать данные для доступа к CloudStack с помощью API. Данный атрибут используется для развертывания многомашинных приложений, когда seed-машина создает необходимые машины с помощью API.

Атрибут `application-db` определяет будут ли в виртуальную машину переданы данные для доступа к разделяемому хранилищу KVS, которое используется для управления приложением (будет описано далее).

Атрибут `vm-db` определяет будут ли в виртуальную машину переданы данные для доступа к приватному хранилищу KVS, ассоциированному с создаваемой VM.

Атрибут `temporary-db` определяет, что необходимо создать временное KVS и передать его в виртуальную машину.

Атрибут `logs` определяет будут ли в виртуальную машину переданы данные для доступа к API LogStash для сохранения записей журналов в централизованный репозиторий записей.

### Атрибут `deployment-progress`

Атрибут используется для указания системному сервису внутри VM, осуществляющему развертывание приложения, куда выводить информацию о статусе и состоянии развертывания.

```yaml
  deployment-progress:
    log: /var/log/deployment_log
    application-key: 'deployment-progress'
    application-deployment-complete-label: 'dafcbe0d4111d0d42378ec40545489ac'
```

Поддерживаются следующие вложенные атрибуты:

* **log** &ndash; определяет файл журнала, в который будут поступать _сырые_ записи о ходе развертывания;
* **application-key** &ndash; определяет ключ в разделяемом KVS приложения, в который могут складываться структурированные обновления, связанные с развертыванием.
* **application-deployment-complete-label** &ndash; определяется метку, которая означает, что установка завершена; UI ожидает появление данной метки в ключе указанном  `application-key`.

### Атрибут `application-state`

Атрибут используется системой CSUI для поиска атрибутов, в которых будет содержаться информация о состоянии приложения в процессе работы. В некотором смысле &ndash; это упрощенный Zabbix для приложения. Данная секция используется для отображения формы состояния развернутого приложения (описана далее).

### Атрибут `network-security`

Атрибут используется для создания группы безопасности для seed-сервера приложения.

При создании приложения пользователь должен подтвердить разрешение на указанные порты. 

В том случае, если зона не поддерживает группы безопасности, данная секция игнорируется.

Для приложения создается новая отдельная группа безопасности.

#### `network-security.{ingress, egress}`

Атрибуты задают правила для `ingress` и `egress` трафика соответственно.

#### `network-security.{ingress, egress}.-rule`

Атрибут содержит конкретные правила для группы безопасности.

##### `network-security.{ingress, egress}.-rule.protocol={tcp,udp}`

Правила для TCP, UDP:

```yaml
- rule:
    protocol: tcp
    ports:
      - 1-65535
    cidr: 0.0.0.0/0
```

Атрибут `protocol` содержит строку `tcp` или `udp`.

Атрибут `ports` содержит список портов, которые относятся к правилу. Порты могут задаваться двумя способами:
* `port` &ndash; транслируется в `port-port` и определяет одиночный порт.
* `port-port` &ndash; соответствует диапазону портов.

В одном правиле для атрибута `ports` может быть несколько перечислений:

```yaml
ports:
  - 80
  - 8080
  - 443
  - 10000-20000
```

Атрибут `cidr` может содержать одно или более CIDR для разрешенных сетей.

```yaml
cidr: 0.0.0.0/0
```

```yaml
cidr:
  - 10.0.0.0/8
  - 8.8.0.0/16
  - ::/0
```

Поддерживаются v4 и v6 адреса.

##### `network-security.{ingress, egress}.-rule.protocol=icmp`

Правила для ICMP:

```yaml
- rule:
    protocol: icmp
    type: -1
    code: -1
    cidr: 0.0.0.0/0
```

Атрибут `protocol` содержит строку `icmp`.

Атрибут `type` содержит ICMP тип (положительное целое число или `-1`).

Атрибут `code` содержит ICMP код (положительное целое число или `-1`).

В одном правиле для атрибута `ports` может быть несколько перечислений:

```yaml
ports:
  - 80
  - 8080
  - 443
  - 10000-20000
```

Атрибут `cidr` может содержать одно или более CIDR для разрешенных сетей.

```yaml
cidr: 0.0.0.0/0
```

```yaml
cidr:
  - 10.0.0.0/8
  - 8.8.0.0/16
  - ::/0
```

Поддерживаются v4 и v6 адреса.

## Автоматически-генерируемые переменные манифеста

Система автоматически генерирует следующие переменные, которые не отображаются в форме, но содержат важную информацию о субъекте и времени выполнения операции конфигурирования (реконфигурирования) приложения:

* **last-update: Number** &ndash; переменная, в которую складывается информация о времени последнего обновления;

* **last-update-author: String** &ndash; автор последнего изменения.

## Целевая конфигурация приложения в формате YaML

Для простоты сгенерированный код конфигурации развертываемого приложения в формате YaML будем называть _файл конфигурации_.

Файл конфигурации генерируется на основании пользовательских значений и предопределенных параметров манифеста приложения. Далее в разделе мы рассмотрим как именно происходит генерация файла конфигурации.

Формат сгенерированного файла должен соответствовать YaML схеме определенной в атрибуте `userdata-schema`.

При генерации создается корневой атрибут `deployment-info`, все остальные атрибуты являются его потомками.

### Атрибут `manifest-schema`

Генерируется переносом из манифеста.

```yaml
manifest-schema: /schemes/vm-sdf-manifest-1.0.yaml
```

### Атрибут `userdata-schema`

Генерируется переносом из манифеста.

```yaml
userdata-schema: /schemes/vm-sdf-userdata-1.0.yaml
```

### Атрибут `source`

Генерируется переносом из манифеста.

```yaml
source: https://reposerver.com/applications/L/lnmp/lnmp-10.tar.gz
```

### Атрибут `mode`

Генерируется переносом из манифеста.

```yaml
mode: raw
```

### Атрибут `application-manifest`

Генерируется из манифеста кодированием в base64.

```yaml
  application-manifest: base64-encoded-manifest-for-application
```

### Атрибут `features`

Атрибут содержит определения для всех запрошенных системных свойств.

#### Атрибут `features.vm-db`

Генерируется, если запрашивается `vm-db`. Позволяет передать в VM KVS, созданную для нее. При генерации передаются все данные для использования:

```yaml
vm-db:
  endpoint: https://kvs.com/
  name: UUID1
  secret: XXXXXXX
```

> Данный атрибут невозможно сгенерировать до создания виртуальной машины, поэтому сначала необходимо создать машину в остановленном состоянии, затем получить данные KVS, созданного для VM, а потом обновить файл конфигурации.

#### Атрибут `features.application-db`

Генерируется, если запрашивается `application-db`. Позволяет передать в VM KVS, созданную для для приложения. При генерации передаются все данные для использования:

```yaml
application-db:
  endpoint: https://kvs.com/
  name: KVSUUID
  secret: XXXXXXX
```

#### Атрибут `features.temporary-db`

Генерируется, если запрашивается `temporary-db`. Позволяет передать в VM временное KVS. При генерации передаются все данные для использования:

```yaml
temporary-db:
  endpoint: https://kvs.com/
  name: KVSUUID
  secret: XXXXXXX
```

#### Атрибут `features.cloudstack-api`

Генерируется, если запрашивается `cloudstack-api`. Позволяет передать в VM данные для доступа к CloudStack от имени аккаунта (`endpoint`, `api-key`, `secret-key`):

```yaml
cloudstack-api:
  endpoint: https://cloudstack.api/client/api
  api-key: apikey
  secret-key: secret
```

#### Атрибут `features.logs`

Генерируется, если запрашивается `logs`. Позволяет передать в VM данные для удаленного логгирования:

```yaml
logs:
  endpoint: https://logs.com/
  name: UUID1
  secret: XXXXXXX
```

> Данный атрибут невозможно сгенерировать до создания виртуальной машины, поэтому сначала необходимо создать машину в остановленном состоянии, затем получить данные для логирования, а потом обновить файл конфигурации.

### Атрибут `deployment-progress`

Данный атрибут и его содержимое переносится из манифеста как есть. Поддерживаются атрибуты `vm-key`, `log`, `application-key`, `application-deployment-complete-label`.

### Атрибут `variables`

В данном атрибуте формируются все переменные параметры, указанные пользователем, включая `last-update` и `last-update-author`, которые генерируются автоматически и заполняются текущим временем в формате `unixtime` и `UUID` пользователя, который произвел изменение.


## Диаграмма развертывания приложения

![](assets/application-deployment-scheme.png)

Для работы VM-SDF обязательно наличие плагина KVS. В том случае, если плагин KVS не поддерживается сервером, то подсистема `VM-SDF` не работает.

При выборе приложения система сначала проверяет, что пользователю доступны все запрашиваемые возможности (KVS, Logs). В том случае, если какая-то из возможностей отсутствует, система показывает пользователю сообщение об этом.

Далее, выполнение развертывания происходит по шагам, отображенным на схеме.

## Реализация в CSUI

Для поддержки приложений создается отдельный плагин Applications, который представлен отдельным пунктом меню "Applications". 