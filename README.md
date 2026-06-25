# Домашнее задание к занятию 15 «Система сбора логов Elastic Stack» - Юрочкин В.А.

## Описание

В рамках домашнего задания был развёрнут на своём сервере Elastic Stack в Docker Compose без использования директории `help`.

<img width="3071" height="1410" alt="image" src="https://github.com/user-attachments/assets/33ff8752-4979-4981-aa34-df71eff33fd4" />


Состав стека:

- Elasticsearch hot node;
- Elasticsearch warm node;
- Logstash;
- Kibana;
- Filebeat.

Filebeat собирает Docker-логи с хоста и отправляет их в Logstash.
Logstash принимает события от Filebeat по Beats input на порту `5044`.
Также Logstash настроен на приём JSON-сообщений по TCP на порту `5000`.

Для удобства проверки используются два индекса:

- `logstash-*` — Docker-логи, собранные Filebeat;
- `tcp-json-*` — ручные JSON-события, отправленные в Logstash по TCP.

## Инфраструктура

Виртуальная машина:

- ОС: Debian 12;
- IP-адрес: `192.168.1.96`;
- Kibana: `http://192.168.1.96:5601`;
- Elasticsearch API: `http://192.168.1.96:9200`.

<img width="2552" height="974" alt="image" src="https://github.com/user-attachments/assets/9c97cf9f-3a6d-43ab-814f-1f361b36957a" />

<img width="2556" height="687" alt="image" src="https://github.com/user-attachments/assets/65302387-a60a-43e5-bc57-addae55b0598" />

<img width="3071" height="1748" alt="image" src="https://github.com/user-attachments/assets/c0df2c65-bde2-4cba-9739-6ff136cea17e" />


## Структура проекта

```text
.
├── docker-compose.yml
├── filebeat
│   └── filebeat.yml
├── logstash
│   └── pipeline
│       └── logstash.conf
├── .gitignore
└── README.md
```

<img width="1133" height="565" alt="image" src="https://github.com/user-attachments/assets/6edf38c3-55fa-4973-9268-ef7ab7c4917b" />


## Подготовка системы

Перед запуском Elasticsearch был увеличен параметр `vm.max_map_count`.

```bash
cat >/etc/sysctl.d/99-elasticsearch.conf <<'SYSCTL'
vm.max_map_count=262144
SYSCTL

/usr/sbin/sysctl --system
/usr/sbin/sysctl vm.max_map_count
```

Результат:

```text
vm.max_map_count = 262144
```

## Запуск стека

```bash
cd /opt/15-elk-stack

docker compose pull
docker compose up -d
```

## Проверка контейнеров

Через несколько минут после старта были запущены 5 контейнеров:

```bash
docker ps
```

Результат:

```text
CONTAINER ID   IMAGE                                                   COMMAND                  CREATED          STATUS              PORTS                                                                                                NAMES
ac515d571ef7   docker.elastic.co/beats/filebeat:7.17.28                "/usr/bin/tini -- /u…"   19 minutes ago   Up About a minute                                                                                                        filebeat
8dcdd24e185e   docker.elastic.co/logstash/logstash:7.17.28             "/usr/local/bin/dock…"   19 minutes ago   Up 3 minutes        0.0.0.0:5000->5000/tcp, [::]:5000->5000/tcp, 0.0.0.0:5044->5044/tcp, [::]:5044->5044/tcp, 9600/tcp   logstash
d36fb518957d   docker.elastic.co/kibana/kibana:7.17.28                 "/bin/tini -- /usr/l…"   19 minutes ago   Up 19 minutes       0.0.0.0:5601->5601/tcp, [::]:5601->5601/tcp                                                          kibana
2217f3ad1381   docker.elastic.co/elasticsearch/elasticsearch:7.17.28   "/bin/tini -- /usr/l…"   19 minutes ago   Up 19 minutes       9200/tcp, 9300/tcp                                                                                   elasticsearch-warm
973ecdd794da   docker.elastic.co/elasticsearch/elasticsearch:7.17.28   "/bin/tini -- /usr/l…"   19 minutes ago   Up 19 minutes       0.0.0.0:9200->9200/tcp, [::]:9200->9200/tcp, 9300/tcp                                                elasticsearch-hot
```

<img width="2554" height="534" alt="image" src="https://github.com/user-attachments/assets/2256072c-928d-4230-a394-173d03f440ef" />

Контейнеры:

- `elasticsearch-hot`;
- `elasticsearch-warm`;
- `logstash`;
- `kibana`;
- `filebeat`.

## Проверка Elasticsearch

Проверка состояния кластера:

```bash
curl -s http://127.0.0.1:9200/_cluster/health?pretty
```

<img width="2553" height="717" alt="image" src="https://github.com/user-attachments/assets/ab3e9bff-7415-41f2-96d0-98232843b53e" />


Результат:

```json
{
  "cluster_name" : "netology-elk",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 2,
  "number_of_data_nodes" : 2,
  "active_primary_shards" : 10,
  "active_shards" : 10,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 2,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 83.33333333333334
}
```

Статус `yellow` связан с тем, что для пользовательских индексов настроена одна реплика, которая не размещается в текущей учебной hot/warm-схеме. На работу стека это не влияет: индексы создаются, документы записываются и доступны для поиска.

Проверка нод:

```bash
curl -s http://127.0.0.1:9200/_cat/nodes?v
```

<img width="2552" height="212" alt="image" src="https://github.com/user-attachments/assets/29be3ad2-b551-48ff-be48-922af1d43cd9" />


Результат:

```text
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.18.0.2           27          96  18    3.73    2.15     1.81 hims      *      elasticsearch-hot
172.18.0.3           20          96  18    3.73    2.15     1.81 imw       -      elasticsearch-warm
```

Проверка индексов:

```bash
curl -s http://127.0.0.1:9200/_cat/indices?v
```

<img width="2551" height="425" alt="image" src="https://github.com/user-attachments/assets/eb41f8b0-d3fd-4a84-9f24-e9f033704dce" />


Результат:

```text
health status index                            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases                 wM-9WhKUQdGNyUKd7pLSNA   1   0         43            0     40.3mb         40.3mb
green  open   .kibana_task_manager_7.17.28_001 2LdTg-JBS2erI6tuwvhr8A   1   0         17         1014      177kb          177kb
green  open   .apm-custom-link                 GdIMAj7dTgWRdujhQJba7g   1   0          0            0       227b           227b
yellow open   tcp-json-2026.06.25              aqs1ZQ7IS06x6zKMAgp24Q   1   1          2            0     17.2kb         17.2kb
green  open   .kibana_7.17.28_001              JSjfThGjRVmcGiBizG9Nfg   1   0        171            7      2.3mb          2.3mb
green  open   .apm-agent-configuration         doLEIb4wTm-CCc4Uqth03g   1   0          0            0       227b           227b
yellow open   logstash-2026.06.25              3DqH_UqzSBS-aVmI0FlJQg   1   1      75776            0     19.9mb         19.9mb
```

В результате были созданы пользовательские индексы:

- `logstash-2026.06.25`;
- `tcp-json-2026.06.25`.

## Проверка TCP JSON input в Logstash

Logstash настроен на приём JSON-сообщений по TCP на порту `5000`.

Тестовое сообщение:

```bash
TEST_ID="tcp-json-test-$(date +%s)"

printf '{"message":"%s","level":"info","service":"tcp-test","homework":"netology-elk","source":"manual-tcp"}' "$TEST_ID" | nc -w 5 127.0.0.1 5000
```

Проверка поиска события в Elasticsearch:

```bash
curl -s "http://127.0.0.1:9200/tcp-json-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "size": 5,
    "_source": [
      "@timestamp",
      "message",
      "level",
      "service",
      "homework",
      "source",
      "tags"
    ],
    "query": {
      "match_phrase": {
        "homework": "netology-elk"
      }
    }
  }'
```

Результат:

```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.36464313,
    "hits" : [
      {
        "_index" : "tcp-json-2026.06.25",
        "_type" : "_doc",
        "_id" : "soZE_Z4BtLHLq5h6fjnY",
        "_score" : 0.36464313,
        "_source" : {
          "@timestamp" : "2026-06-25T05:33:04.149Z",
          "homework" : "netology-elk",
          "level" : "info",
          "service" : "tcp-test",
          "source" : "manual-tcp",
          "message" : "tcp-json-test-1782365583",
          "tags" : [
            "tcp-json"
          ]
        }
      }
    ]
  }
}
```

## Проверка Filebeat

Filebeat собирает Docker-логи с хоста из каталога:

```text
/var/lib/docker/containers/*/*.log
```

и отправляет их в Logstash на порт `5044`.

После запуска Filebeat в Elasticsearch появился индекс:

```text
logstash-2026.06.25
```

Проверка:

```bash
curl -s http://127.0.0.1:9200/_cat/indices?v
```

Фрагмент результата:

```text
yellow open logstash-2026.06.25 3DqH_UqzSBS-aVmI0FlJQg 1 1 75776 0 19.9mb 19.9mb
```

## Kibana

Kibana доступна по адресу:

```text
http://192.168.1.96:5601
```

<img width="3071" height="1434" alt="image" src="https://github.com/user-attachments/assets/41b8bca4-f135-49f8-90ca-487206fdbdf7" />


В Kibana были созданы два index pattern / data view:

```text
logstash-*
tcp-json-*
```

Для обоих index pattern выбрано поле времени:

```text
@timestamp
```

<img width="2048" height="1163" alt="image(527)" src="https://github.com/user-attachments/assets/c0fc100b-df4b-4f64-9210-58465ca6b599" />

<img width="2048" height="1166" alt="image(528)" src="https://github.com/user-attachments/assets/0eee3b47-9da9-469f-8aa1-96b814b0a252" />


В разделе Discover проверен просмотр логов:

- `logstash-*` — Docker-логи, собранные Filebeat;
- `tcp-json-*` — JSON-события, отправленные в Logstash по TCP.

## Конфигурация Logstash

Файл `logstash/pipeline/logstash.conf`:

```conf
input {
  beats {
    port => 5044
  }

  tcp {
    port => 5000
    codec => json
    tags => ["tcp-json"]
  }
}

filter {
  if [container][name] {
    mutate {
      add_field => {
        "docker_container" => "%{[container][name]}"
      }
    }
  }
}

output {
  if "tcp-json" in [tags] {
    elasticsearch {
      hosts => ["http://elasticsearch-hot:9200"]
      index => "tcp-json-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts => ["http://elasticsearch-hot:9200"]
      index => "logstash-%{+YYYY.MM.dd}"
    }
  }
}
```

## Конфигурация Filebeat

Файл `filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
  - type: container
    enabled: true
    paths:
      - /var/lib/docker/containers/*/*.log

processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
```

## Docker Compose

Основные сервисы описаны в файле `docker-compose.yml`:

- `elasticsearch-hot`;
- `elasticsearch-warm`;
- `logstash`;
- `kibana`;
- `filebeat`.

## Вывод

В результате выполнения домашнего задания был развёрнут Elastic Stack в Docker Compose.

Выполнено:

- поднят Elasticsearch с двумя нодами: hot и warm;
- поднят Logstash;
- поднята Kibana;
- поднят Filebeat;
- Filebeat настроен на сбор Docker-логов;
- Logstash настроен на приём событий от Filebeat;
- Logstash настроен на приём JSON-сообщений по TCP;
- в Elasticsearch созданы индексы `logstash-*` и `tcp-json-*`;
- в Kibana созданы index patterns;
- в Discover проверен просмотр и поиск логов.
