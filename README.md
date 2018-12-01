# VM Services Deployment Framework

VM-SDF представляет собой экосистему, предназначенную для расширенного конфигурирования виртуальных машин перед развертыванием. Под расширенным конфигурированием понимается выбор приложения из библиотеки приложений и настройка данного приложения с помощью конфигурационных переменных аналогично тому, как это сделано в DC/OS.

Пользователь должен иметь возможность выбрать библиотеку приложений и видеть все приложения, которые в ней доступны. Для выбранного приложения пользователь задает конфигурацию с помощью конфигурационных переменных и данные настроечные параметры передаются в виртуальную машину с помощью USERDATA.

VM-SDF поддерживает два способа передачи параметров в виртуальную машины с помощью USERDATA:
нативный JSON | base64
 * с использованием CS-KVS
 * с использованием CS-KVS и CS-OTA
 * Далее механизмы будут описаны подробно.

VM-SDF построен на основе динамически генерируемых на основе YaML форм. Каждая такая форма определяется для каждого приложения и позволяет указать как параметры, которые могут задаваться пользователем, так и предопределенные параметры, которые определяют дополнительные передаваемые в USERDATA аргументы.

Примеры:
 * Пользовательский параметр mysql.max.memory = NUMBER, DEFAULT: 128
 * Предопределенный параметр: system.pass.account-api-credentials = True
 * Предопределенный параметр: system.executor = Ansible

Пользовательские параметры являются редактируемыми, а предопределенные нет. 

Должен поддерживаться тип параметра, который позволяет указать сервисное предложение (fixed, customized) и размер ROOT-диска.

В самом простом случае VM-SDF может использоваться для развертывания приложения в рамках одного узла, рассмотрим пример LAMP сервер:

```yaml
lnmp-10:
  schema: /schemes/vm-sdf-1.0.yaml
  name: LNMP Server
  version: 1.0
  description: LAMP Server (Linux, NGINX, MySQL, PHP FPM)
  icon: /assets/lnmp-10.png
  license: Apache 2.0
  engine: ansible | docker-stack
  source: https://reposerver.com/applications/L/lnmp-10/lnmp-10.tar.gz
  templates:
    - compatible-template1
    - compatible-template2
  minimal-disk-size: 10
  features-requested:
    pass-cloudstack-api-info: false
    shared-key-value-store: false
    private-key-value-store: true
    log-store: true
    use-ota: true
  deployment-log: /var/log/deployment_log
  deployment-progress:
    private-key-value-store-key: 'deployment-progress'
    [shared-key-value-store-key: ...]
  variables:
    domain-name:
      description:
        default: Domain name for the first Website
        ru_RU: Имя домена для создаваемого сайта
      type: FQDN_DNS
      required: true
      default: example.com
    mysql-root-password:
      description:
        default: Password for MySQL root user
        ru_RU: ...
      type: PASSWORD
      length: 8
      required: true
    mysql-database:
      description: MySQL database for the first Website
      type: REGEX
      regex: '...'
      length: 10
      default: db
      required: true
    mysql-user-name:
      description: The name for the unprivileged user who can access created database
      type: REGEX
      regex: '...'
      length: 8
      default: www
      required: true
    mysql-user-password:
      description: The password for the unprivileged user who can access created database
      type: PASSWORD
      length: 8
      required: true
    mysql-php-admin-install:
      description: Do you want PhpMyAdmin to be installed?
      type: BOOLEAN
      default: false
      required: true
    mysql-php-admin-https-port:
      description: PhpMyAdmin HTTPS port to listen
      type: INTEGER
      integer_min: 1024
      integer_max: 65535
      default: 8443
      required: true
    use-lets-encrypt:
      description: Use Let's Encrypt to autmatically protect HTTPS?
      type: BOOLEAN
      default: true
      required: true
```

В более сложных случаях первая машина, которая выполняет конфигурацию может выступать в качестве seed-узла, который создает дополнительные машины. Пример данной конфигурации:

```yaml
mysql-ms-10:
	schema: /schemes/v10.yaml
	name: MySQL Master-Slaves Bundle
	version: 1.0
	description: 
		default: MySQL Master-Slaves cluster with a single master and number of slaves
		ru_RU: ...
	icon: /assets/mysql-master-slave-10.png
	license: Apache 2.0
	engine: ansible
	source: /applications/L/mysql-ms-10/mysql-ms-10.tar.gz
group: mysql-cluster-1
	templates:
		- ubuntu-template1
		- centos-template2
	minimal-disk-size: 10
	features-requested:
		pass-cloudstack-api-info: true
		shared-key-value-store: true
		private-key-value-store: true
		log-store: true
		use-ota: true
	deployment-log: /var/log/deployment_log
	deployment-progress:
		private-key-value-store-key: 'deployment-progress'
		[shared-key-value-store-key: ...]
	variables:
		mysql-root-password:
			description: 
				default: Password for MySQL root user
				ru_RU: ...
			type: PASSWORD
			length: 8
			required: true
		mysql-database:
			description: MySQL database for the first Website
			type: REGEX
			regex: '...'
			length: 10
			default: db
			required: true
		mysql-user-name:
			description: The name for the user who can access created database 
			type: REGEX
			regex: '...'
			length: 8
			default: www
			required: true
		mysql-user-password:
			description: The password for the user who can access created database 
			type: PASSWORD
			length: 8
			required: true
		mysql-php-admin-install:
			description: Do you want PhpMyAdmin to be installed? 
			type: BOOLEAN
			default: false
			required: true
		mysql-php-admin-https-port:
			description: PhpMyAdmin HTTPS port to listen
			type: INTEGER
			integer_min: 1024
			integer_max: 65535
			default: 8443
			required: true
		use-lets-encrypt:			    
			description: Use Let's Encrypt to autmatically protect HTTPS? 
			type: BOOLEAN
			default: true
			required: true
		mysql-slave-so:
			description:
				default: Service Offering for MySQL Slave Servers
			type: SERVICE-OFFERING
			service-offering:
				minimal-cores: 2
				minimal-frequency: 1000
				minimal-ram: 2
				minimal-disk-size: 10
			required: true
		mysql-slaves-count:
			description:
				default: Number of MySQL slave servers to be launched
			type: INTEGER
			integer_min: 1
			integer_max: 32
			default: 1
			required: true
```

В данном случае, приложение создается на MySQL master-узле, который, в свою очередь, создает slave-узлы с помощью CloudStack API (pass-cloudstack-api-info: true), далее настраивает связку master-slave.

В этом сценарии используется специальный тип переменной SERVICE-OFFERING, который предназначен для отображения диалога выбора SO, который будет использоваться SLAVE серверами.

Все определения приложений, подобно приведенному выше компилируются специальным сценарием сборки  на этапе публикации в единый JSON-документ, который, в свою оцередь, загружается на сервер и проксируется приложением CSUI, при этом обращение к данному документу должно производиться без кэширования, чтобы при его обновлении, CSUI мог видеть новые, а не устаревшие данные.

При создании виртуальной машины пользователь может выбрать создать VM из шаблона, ISO или приложения. В третьем случае в действие вступает VM-SDF, который позволяет пользователю задать параметры для развертываемого приложения с помощью формы, которая генерируется для выбранного из каталога приложения.

Все параметры, сконфигурированные на данном этапе пользователь может посмотреть в форме YaML-документа на дополнительной вкладке формы редактирования параметров.

Пример YaML, сгенерированного для некоторой формы:

```yaml
deployment-info:
	engine: ansible
	source: https://reposerver.com/applications/L/mysql-ms-10/mysql-ms-10.tar.gz
	mode: RAW
	features:
		cloudstack-api-info:
			endpoint: https://api.com/client/api
			access-key: aaaAAAA
			secret-key: XXXXXXX
		private-key-value-store:
			endpoint: https://kvs.com/
			name: UUID1
			secret: XXXXXXX
		shared-key-value-store:
			endpoint: https://kvs.com/
			name: UUID2
			secret: XXXXXXX
		log-store:
			endpoint: https://logs.com/
			name: VMUUID
			secret: XXXXXXX
	deployment-progress:
		log: /var/log/deployment_log
		private-key-value-store-key: deployment-progress
		shared-key-value-store-key: deployment-progress
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

## Способы передачи данных

Существует три способа передачи данных в USERDATA:
 * RAW - данные передаются в форме YaML как есть с использованием Base64
 * KV - в Userdata передаются только данные для KVS, а deployment info упаковывается уже в KVS
 * OTA+KV - так же как и (2), но KV не передается напряму, а передается OTA для KV.

Пример RAW отображен ранее.

**KV**. В этом режиме private-key-value-store используется для передачи остальных сведений, которые преобразуются из YaML в строки вида “a.b.c”  и передаются как данные KV

```yaml
deployment-info:
	mode: KV
	endpoint: https://kvs.com/
	name: UUID1
	secret: XXXXXXX
```

**OTA**. В этом режиме OTA используется для безопасной передачи данных подключения к private-key-value-store используется для передачи остальных сведений, которые преобразуются из YaML в строки вида “a.b.c”  и передаются как данные KV.

```yaml
deployment-info:
	mode: OTA
	endpoint: https://ota.com/
	token: XXXXXXX
```
  
## Типы переменных

Следующие типы переменных должны поддерживаться в формах:
 * Number - обычное целое число
 * String - строка
 * DomainName - доменное имя
 * IpAddress
 * URI - корректный URI
 * Email - корректный E-mail
 * Password - пароль с подтверждением
 * OneOf - радиокнопки или Dropdown 
 * Some - набор чекбоксов с описаниями
 * ServiceOffering - выбор сервисного предложения
 * Section - разделитель разделов

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
		predicate: AND|OR
		sources:
			field-name-2: X
			field-name-3: 
         - Y
         - Z
	default: value
```

* label - то, что отображается в качестве имени поля;
* description - то, что отображается в качестве tooltip при наведении на поле;
* type - тип поля, определяющий расширенные настройки;
* active-if - означает, что данная переменная активна для изменения тогда, когда выполняется предикат;
* default - значение по-умолчанию.

**Active-if** используется для контекстной активации части переменных в зависимости от значений других переменных. В настоящее время планируется поддержка только режимов:
* AND, который активирует переменную, если активирующие переменные все одновременно имеют указанные значения;
* OR, который активирует переменную, если хотя бы одна активирующая переменная имеет указанное значение.

В качестве значения активирующей переменной можно установить одно значение или список. В случае списка, активация происходит при совпадении значения с любым из элементов списка.

В случае, если переменная не активна, в результирующем YaML ей сопоставляется значение YaML NULL.

Должна поддерживаться многоуровневая активация: если активированная переменная имеет значение по-умолчанию, которое достаточно для активации новых переменных, они должны быть активированы тоже. На ум приходит реализация через реактивные компоненты.
Number
Дополнительно добавляются следующие атрибуты:
opt: minimum - минимально допустимое значение
opt: maximum - максимально допустимое значание
opt (true): is-integer: true|false - целое или float
String
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
DomainName
Однострочная STRING с валидатором корректного FQDN.
IpAddress
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
URI
Поле ввода URL. Дополнительный параметр:
opt(all): scheme - схема протокола (https://www.iana.org/assignments/uri-schemes/uri-schemes.xhtml)
opt(1024): maximum - максимальная длина URI 

E.g.

scheme:
http
https

Пользователь может вводить scheme://name:password@host.com:port/… 
Email
Однострочная STRING с валидатором корректного E-mail.
Password
Пароль с подтверждением и возможностью автогенерации. Дополнительные параметры:
opt(5): minimum - минимальное количество символов.
opt(16): maximum - максимальное количество симоволов.
opt(none): validator - regex

Компонент представляет собой классическое задание пароля с подтверждением, дополнительные функции - автозаполнение (генерация, которая работает только при отсутствии установленного валидатора) и кнопка “показать пароль”.
OneOf
Выбор одного значения из нескольких. Дополнительные параметры:
options - опции, между которыми производится выбор.
opt(dropdown) type - тип компонента - может быть набор radio-кнопок или dropdown. 

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
Some
Выбор нескольких значений. Дополнительные параметры:
options - опции, аналогично радио.
defaults - список, выбранных по-умолчанию компонентов. Унаследованное значение default так же может использоваться для выбора элементов, выбранных по умолчанию. 

defaults:
abc
def
ServiceOffering
Выбор сервисного предложения из доступных. Выбор SO производится аналогичным образом как и при обычном создании VM. Дополнительные параметры:
filters - фильтрующий компонент, определяющий допустимые SO.

Все SO, которые окажутся попадающими под критерии любого из фильтров участвуют в выборе. Custom SO в случае наличия фильтра по ресурсам отбирается еще после задания конкретного количества ресурсов. В каждом фильтре могут встречаться от 1 до всех критериев фильтрации:

OR-like:

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


AND-like

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
Section 
Вспомогательное поле, использующееся для разделения блоков переменных и для активации блоков по зависимости.

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


В приведенном выше примере активация секций и полей в этих секций выполняется в каскадном режиме после указания mysql-mode.
