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
  schema: /schemes/vm-sdf-1.0.yaml
  name: LNMP Server
  version: 1.0
  description: LAMP Server (Linux, NGINX, MySQL, PHP FPM)
  icon: /assets/lnmp-10.png
  license: Apache 2.0
  engine: ansible
  source: https://reposerver.com/applications/L/lnmp-10/lnmp-10.tar.gz
  templates:
    - compatible-template1
    - compatible-template2
  minimal-root-disk-size: 10
  service-offering:
    filters:
      resources:
        minimal-ram: 1024
        minimal-cores: 2
        minimal-frequency: 1000
  features-requested:
    vm-key-value-storage: true
    log-storage: true
    use-ota: true
  deployment-progress:
    log: /var/log/deployment_log
    vm-key: 'deployment-progress'
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
  engine: ansible
  source: https://reposerver.com/applications/L/lnmp/lnmp-10.tar.gz
  mode: raw
  application-manifest: base64-encoded-manifest-for-application
  features:
	vm-key-value-storage:
	  endpoint: https://kvs.com/
	  name: UUID1
	  secret: XXXXXXX
	log-store:
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

Следующие типы переменных должны поддерживаться в формах:
 * Number &ndash; число;
 * String &ndash; строка;
 * DomainName &ndash; доменное имя;
 * IpAddress &ndash; интернет-адрес v4, v6;
 * URI &ndash; корректный URI;
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

В настоящее время планируется поддержка двух способов объединения:
* `and`, который активирует переменную, если активирующие переменные все одновременно имеют указанные значения;
* `or`, который активирует переменную, если хотя бы одна активирующая переменная имеет указанное значение.

В качестве значения активирующей переменной можно установить одно значение или список. В случае списка, активация происходит при совпадении значения с любым из элементов списка.

В случае, если переменная неактивна в рамках текущих активирующих зависимостей, в результирующем YaML ей сопоставляется значение YaML `NULL`.

Должна поддерживаться многоуровневая активация: если активированная переменная имеет значение по-умолчанию, которое достаточно для активации новых переменных, они должны быть активированы тоже.

### Атрибут Volatile

Атрибут определяет может ли параметр изменяться после первоначальной установки. Это необходимо для разделения параметров, которые являются модифицируемыми после развертывания или не могут изменяться.

По-умолчанию, 

## Переменная типа Number

Дополнительно к атрибутам общего вида добавляются следующие атрибуты:
* **minimum** &ndash; минимально допустимое значение, по-умолчанию не ограничен;
* **maximum** &ndash; максимально допустимое значание, по-умолчанию не ограничен;
* **is-integer** &ndash; целое или float, по умолчанию `true`, то есть целое число.

### Переменная типа String
Дополнительно добавляются следующие атрибуты:
opt: minimum - минимальное количество символов
opt: maximum - максимальное количество символов
opt(false): is-multiline - input или text-area
opt(false): multiline-base64 - трансформируется ли многострочный текст в однострочный base64
opt: regex - регулярное выражение, которому должна соответствовать строка

В случае (is-multiline: true, multiline-base64: false), в целевом YaML для данной переменной генерируется текст вида:

varname:
line1
line2
line3
line4

Не забываем про специальные символы в YaML, генерировать лучше с помощью DOM-библиотеки, а не вручную.
### DomainName
Однострочная STRING с валидатором корректного FQDN.
### IpAddress
Поле ввода IP-адреса в форме строки. Дополнительные параметры:
opt(4): version - версия протокола, может иметь значение 4, 6.
opt(false): is_cidr: true|false - должен ли пользователь вводить префикс CIDR-сети или нет.

field:
	cidr: true
version:
4
6

field4:
	cidr: false
	version:
4

field6:
	cidr: true
	version:
6

### URI

Поле ввода URL. Дополнительный параметр:
opt(all): scheme - схема протокола (https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml)
opt(1024): maximum - максимальная длина URI 

E.g.

scheme:
http
https

Пользователь может вводить scheme://name:password@host.com:port/… 

### Email

Однострочная STRING с валидатором корректного E-mail.
Password
Пароль с подтверждением и возможностью автогенерации. Дополнительные параметры:
opt(5): minimum - минимальное количество символов.
opt(16): maximum - максимальное количество симоволов.
opt(none): validator - regex

Компонент представляет собой классическое задание пароля с подтверждением, дополнительные функции - автозаполнение (генерация, которая работает только при отсутствии установленного валидатора) и кнопка “показать пароль”.

### OneOf

Выбор одного значения из нескольких. Дополнительные параметры:
options - опции, между которыми производится выбор.
opt(dropdown) type - тип компонента - может быть набор radio-кнопок или dropdown. 

```yaml
field-name:
  options:
    option1:
      labels:
        default: label1
        ru_RU: буу
    option2:
      labels:
        default: label2
        ru_RU: буу2
```

### Some

Выбор нескольких значений. Дополнительные параметры:
options - опции, аналогично радио.
defaults - список, выбранных по-умолчанию компонентов. Унаследованное значение default так же может использоваться для выбора элементов, выбранных по умолчанию. 

defaults:
abc
def

### ServiceOffering

Выбор сервисного предложения из доступных. Выбор SO производится аналогичным образом как и при обычном создании VM. Дополнительные параметры:
filters - фильтрующий компонент, определяющий допустимые SO.

Все SO, которые окажутся попадающими под критерии любого из фильтров участвуют в выборе. Custom SO в случае наличия фильтра по ресурсам отбирается еще после задания конкретного количества ресурсов. В каждом фильтре могут встречаться от 1 до всех критериев фильтрации:

OR-like:

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

AND-like


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

### Section 

Вспомогательное поле, использующееся для разделения блоков переменных и для активации блоков по зависимости.

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
