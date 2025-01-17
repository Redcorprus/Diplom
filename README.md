
#  Дипломная работа по профессии «Системный администратор»

<details> 

Содержание
==========
* [Задача](#Задача)
* [Инфраструктура](#Инфраструктура)
    * [Сайт](#Сайт)
    * [Мониторинг](#Мониторинг)
    * [Логи](#Логи)
    * [Сеть](#Сеть)
    * [Резервное копирование](#Резервное-копирование)
    * [Дополнительно](#Дополнительно)
* [Выполнение работы](#Выполнение-работы)
* [Критерии сдачи](#Критерии-сдачи)
* [Как правильно задавать вопросы дипломному руководителю](#Как-правильно-задавать-вопросы-дипломному-руководителю) 

---------

## Задача
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в [Yandex Cloud](https://cloud.yandex.com/) и отвечать минимальным стандартам безопасности: запрещается выкладывать токен от облака в git. Используйте [инструкцию](https://cloud.yandex.ru/docs/tutorials/infrastructure-management/terraform-quickstart#get-credentials).

**Перед началом работы над дипломным заданием изучите [Инструкция по экономии облачных ресурсов](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD).**

## Инфраструктура
Для развёртки инфраструктуры используйте Terraform и Ansible.  

Не используйте для ansible inventory ip-адреса! Вместо этого используйте fqdn имена виртуальных машин в зоне ".ru-central1.internal". Пример: example.ru-central1.internal  

Важно: используйте по-возможности **минимальные конфигурации ВМ**:2 ядра 20% Intel ice lake, 2-4Гб памяти, 10hdd, прерываемая. 

**Так как прерываемая ВМ проработает не больше 24ч, перед сдачей работы на проверку дипломному руководителю сделайте ваши ВМ постоянно работающими.**

Ознакомьтесь со всеми пунктами из этой секции, не беритесь сразу выполнять задание, не дочитав до конца. Пункты взаимосвязаны и могут влиять друг на друга.

### Сайт
Создайте две ВМ в разных зонах, установите на них сервер nginx, если его там нет. ОС и содержимое ВМ должно быть идентичным, это будут наши веб-сервера.

Используйте набор статичных файлов для сайта. Можно переиспользовать сайт из домашнего задания.

Создайте [Target Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/target-group), включите в неё две созданных ВМ.

Создайте [Backend Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/backend-group), настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

Создайте [HTTP router](https://cloud.yandex.com/docs/application-load-balancer/concepts/http-router). Путь укажите — /, backend group — созданную ранее.

Создайте [Application load balancer](https://cloud.yandex.com/en/docs/application-load-balancer/) для распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип auto, порт 80.

Протестируйте сайт
`curl -v <публичный IP балансера>:80` 

### Мониторинг
Создайте ВМ, разверните на ней Zabbix. На каждую ВМ установите Zabbix Agent, настройте агенты на отправление метрик в Zabbix. 

Настройте дешборды с отображением метрик, минимальный набор — по принципу USE (Utilization, Saturation, Errors) для CPU, RAM, диски, сеть, http запросов к веб-серверам. Добавьте необходимые tresholds на соответствующие графики.

### Логи
Cоздайте ВМ, разверните на ней Elasticsearch. Установите filebeat в ВМ к веб-серверам, настройте на отправку access.log, error.log nginx в Elasticsearch.

Создайте ВМ, разверните на ней Kibana, сконфигурируйте соединение с Elasticsearch.

### Сеть
Разверните один VPC. Сервера web, Elasticsearch поместите в приватные подсети. Сервера Zabbix, Kibana, application load balancer определите в публичную подсеть.

Настройте [Security Groups](https://cloud.yandex.com/docs/vpc/concepts/security-groups) соответствующих сервисов на входящий трафик только к нужным портам.

Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh.  Эта вм будет реализовывать концепцию  [bastion host]( https://cloud.yandex.ru/docs/tutorials/routing/bastion) . Синоним "bastion host" - "Jump host". Подключение  ansible к серверам web и Elasticsearch через данный bastion host можно сделать с помощью  [ProxyCommand](https://docs.ansible.com/ansible/latest/network/user_guide/network_debug_troubleshooting.html#network-delegate-to-vs-proxycommand) . Допускается установка и запуск ansible непосредственно на bastion host.(Этот вариант легче в настройке)

### Резервное копирование
Создайте snapshot дисков всех ВМ. Ограничьте время жизни snaphot в неделю. Сами snaphot настройте на ежедневное копирование.

### Дополнительно
Не входит в минимальные требования. 

1. Для Zabbix можно реализовать разделение компонент - frontend, server, database. Frontend отдельной ВМ поместите в публичную подсеть, назначте публичный IP. Server поместите в приватную подсеть, настройте security group на разрешение трафика между frontend и server. Для Database используйте [Yandex Managed Service for PostgreSQL](https://cloud.yandex.com/en-ru/services/managed-postgresql). Разверните кластер из двух нод с автоматическим failover.
2. Вместо конкретных ВМ, которые входят в target group, можно создать [Instance Group](https://cloud.yandex.com/en/docs/compute/concepts/instance-groups/), для которой настройте следующие правила автоматического горизонтального масштабирования: минимальное количество ВМ на зону — 1, максимальный размер группы — 3.
3. В Elasticsearch добавьте мониторинг логов самого себя, Kibana, Zabbix, через filebeat. Можно использовать logstash тоже.
4. Воспользуйтесь Yandex Certificate Manager, выпустите сертификат для сайта, если есть доменное имя. Перенастройте работу балансера на HTTPS, при этом нацелен он будет на HTTP веб-серверов.

## Выполнение работы
На этом этапе вы непосредственно выполняете работу. При этом вы можете консультироваться с руководителем по поводу вопросов, требующих уточнения.

⚠️ В случае недоступности ресурсов Elastic для скачивания рекомендуется разворачивать сервисы с помощью docker контейнеров, основанных на официальных образах.

**Важно**: Ещё можно задавать вопросы по поводу того, как реализовать ту или иную функциональность. И руководитель определяет, правильно вы её реализовали или нет. Любые вопросы, которые не освещены в этом документе, стоит уточнять у руководителя. Если его требования и указания расходятся с указанными в этом документе, то приоритетны требования и указания руководителя.

## Критерии сдачи
1. Инфраструктура отвечает минимальным требованиям, описанным в [Задаче](#Задача).
2. Предоставлен доступ ко всем ресурсам, у которых предполагается веб-страница (сайт, Kibana, Zabbix).
3. Для ресурсов, к которым предоставить доступ проблематично, предоставлены скриншоты, команды, stdout, stderr, подтверждающие работу ресурса.
4. Работа оформлена в отдельном репозитории в GitHub или в [Google Docs](https://docs.google.com/), разрешён доступ по ссылке. 
5. Код размещён в репозитории в GitHub.
6. Работа оформлена так, чтобы были понятны ваши решения и компромиссы. 
7. Если использованы дополнительные репозитории, доступ к ним открыт. 

## Как правильно задавать вопросы дипломному руководителю
Что поможет решить большинство частых проблем:
1. Попробовать найти ответ сначала самостоятельно в интернете или в материалах курса и только после этого спрашивать у дипломного руководителя. Навык поиска ответов пригодится вам в профессиональной деятельности.
2. Если вопросов больше одного, присылайте их в виде нумерованного списка. Так дипломному руководителю будет проще отвечать на каждый из них.
3. При необходимости прикрепите к вопросу скриншоты и стрелочкой покажите, где не получается. Программу для этого можно скачать [здесь](https://app.prntscr.com/ru/).

Что может стать источником проблем:
1. Вопросы вида «Ничего не работает. Не запускается. Всё сломалось». Дипломный руководитель не сможет ответить на такой вопрос без дополнительных уточнений. Цените своё время и время других.
2. Откладывание выполнения дипломной работы на последний момент.
3. Ожидание моментального ответа на свой вопрос. Дипломные руководители — работающие инженеры, которые занимаются, кроме преподавания, своими проектами. Их время ограничено, поэтому постарайтесь задавать правильные вопросы, чтобы получать быстрые ответы :)

</details>

# Решение

<details>

### Подготовка локальной VM для работы с Yandex Cloud

В первую очередь, нам необходимо установить `YC Cli`, ориентируясь на документацию на сайте.

Следующим шагом мы устанавливаем и инициализируем terraform
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img1.png)

### Настройка Terraform и поднятие инфраструктуры

Далее мы конфигурируем нашу будущую облачную инфраструктуру для поднятия её через terraform в файле [main.tf](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/terraform/main.tf) и проверяем командой `terraform plan`
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img2.png)

Далее мы проверяем результат вывода команды `terraform apply` уже в самой консоли YC
Как мы видим, все необходимые ВМ для выполнения задачи поднялись и работают
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img3.png)

### Сайт

#### Создание 2 VM для веб серверов

Первым шагом нам необходимо установить и настроить ansible для работы с облачной инфраструктурой. Настраиваем список ВМ в файле [hosts.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/hosts.yml)
и проверяем доступность наших web серверов
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img8.png)

следующим шагом мы запускаем наш ansible-playbook [nginx.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/nginx.yml) для установки nginx на подготовленные облачные ВМ для web серверов c  копированием фала `index.html`
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img9.png)

#### Создание target group
На этом шаге мы создаем target group и вносим в неё наши web серверы. 
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img4.png)

#### Создание backend group
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img5.png)

#### HTTTP-router
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img6.png)

#### Application load-balancer
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img7.png)

#### Протестируем сайт
`curl -v 158.160.130.127:80` 

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img10.png)

Так же проверим сайт через web

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img11.png)



### Мониторинг
Первым шагом мы проверям доступность нашей ВМ, подготовленной для Zabbix 
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img12.png)

Следующим шагом мы запускаем наш ansible-playbook [zabbix.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/zabbix.yml) для установки zabbix сервера

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img13.png)

Далее нам необходимо организовать сбор метрик со всех серверов. Реализуем эту задачу через запуск ansible-playbook [zabbix-agent.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/zabbix-agent.yml) для установки zabbix-agent на все сервера с копированием [настроек](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/config/zabbix_agent.conf)

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img14.png)

Настроим дашборд согласно поставленной задачи:

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img15.png)

#### Дашборд доступен на сервере Zabbix адресу:
http://158.160.139.152:8080/zabbix.php?action=dashboard.view

- Логин Admin
- Пароль zabbix

### Логи

#### Разворачиваем Elasticsearch 
Запускаем установику через наш ansible-playbook [elasticsearch.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/elasticsearch.yml) и проверяем работоспособность сервиса командой `CURL --user morzin:123456 -X GET http"//192.168.3.10:9200?pretty"`
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img16.png)

#### Разворачиваем Kibana
Запускаем установику через наш ansible-playbook [kibana.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/kibana.yml)
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img17.png)

#### Устанавливаем Filebeat
Запускаем установику через наш ansible-playbook [filebeat.yml](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/ansible/filebeat.yml)
![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img18.png)

#### Проверяем работу стека для сбора логов

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img19.png)

Стек доступен по ссылке http://158.160.147.28:5601/app/discover#/?_g=(filters:!(),query:(language:kuery,query:''),refreshInterval:(pause:!t,value:0),time:(from:now%2Fd,to:now%2Fd))&_a=(columns:!(),filters:!(),index:'filebeat-*',interval:auto,query:(language:kuery,query:''),sort:!(!('@timestamp',desc)))


### Сеть
Создадим VPC и поместим ВМ в подсети, согласно задаче:
Карта сети

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img30.png)

Настроим Security Groups

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img31.png)

Доступ к хостам осуществляется по ssh через bastion.
Например, для подключение к серверу Elasticsearch нам необходимо выполнить команду:
`ssh -i ~/.ssh/id_ed25519 -J morzin@158.160.146.42 morzin@192.168.3.10`

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img32.png)


### Резервное копирование

Для настройки резервного копирования запускаем [snapshots.tf](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/terraform/snapshots.tf)

Проверяем работу

![image](https://github.com/Redcorprus/Diplom/blob/diplom-zabbix/images/img20.png)

</details>

