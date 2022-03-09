Writing and install filebeat module for modsecurity JSON audit log

- [1\. Module Create Hint](#1-module-create-hint)
    - [1.1 JSON Format Log Parsing](#11-json-format-log-parsing)
    - [1.2 @timestamp in Ingest Pipeline Parsing](#12-timestamp-in-ingest-pipeline-parsing)
- [2\. Install modsecurity Filebeat Module](#2-install-modsecurity-filebeat-module)
    - [2.1 Enable modsecurity module of filebeat](#21-enable-modsecurity-module-of-filebeat)
    - [2.2 Import Ingest Pipeline into Elasticsearch](#22-import-ingest-pipeline-into-elasticsearch)
- [3\. Feeding Log into Elasticsearch](#3-feeding-log-into-elasticsearch)
    - [3.1 Feed the first log and check successful or not](#31-feed-the-first-log-and-check-successful-or-not)
    - [3.2 Complete the module setup](#32-complete-the-module-setup)

## 1\. Module Create Hint

1.  Naming module (cannot duplicate with [here](https://github.com/elastic/beats/tree/main/filebeat/module) )
2.  Naming the dataset inside module (eg. modsecurity -> audit )
3.  Create yml files (check modules in references). You will have directory structure like this:
    ![daa7ca6a2ad1d2439ddbd3426cc4eea3.png](../_resources/5f959d1abd0c4dddb34a7fb0cd6399fb.png)
4.  Most setting should inside "dataset" config. In this case, it's `audit -> config -> audit.yml`

### 1.1 JSON Format Log Parsing

In dataset config file (`audit.yml` here), you can decode JSON event if your whole event is encoded by JSON.

For example, an event means one on these lines of your log:

```JSON
{"aaa":"bbb", "ddd":"ccc"}
{"aaa":"333", "ddd":"444"}
```

or each file is single event (like modsecurity audit log JSON format):

```JSON
{
  "transaction": {
    "client_ip": "11.22.33.44",
    "time_stamp": "Tue Mar  8 10:34:39 2022",
  }
}
```

Then in your config file, you can add processors like this:

```yaml
processors:
  - decode_json_fields:
      fields: [message]
      process_array: true
      target: "modsecurity"
```

The above setting will decode original event (which saved in field "`message`") into JSON, and set to variable `modsecurity` for further use.

**NOTE that**, the whole JSON structure above will also import to Elasticsearch fields mapping of filebeat automatically. After this config, when you setup filebeat, fields mapping will like this in kibana:
![056b78921ebd130e866525b7d2b6a3b2.png](../_resources/2efab4ea37df4ff6896a00fa8412ce3d.png)

You should also check document of [decode\_json\_fields](https://www.elastic.co/guide/en/beats/filebeat/current/decode-json-fields.html).

### 1.2 @timestamp in Ingest Pipeline Parsing

Even the JSON data will be import to Elasticsearch automatically, we still want to add some pre-defined fields in Elasticsearch like `IP` or `HTTP Status Code`. Here we should define in Ingest Pipeline.

For example, in modsecurity we need to extract exactly datetime in event to save to `@timestamp` field of Elasticsearch. It's very important we have correct `@timestamp` saved in database. Which most of Kibana use this field to filter log, event, and discover.
The original datetime format of modsecurity like this:

```
Tue Mar  8 10:34:39 2022
```

Which is strange because the day of month have leading space. We need to use multiple processors in pipeline to extract field `modsecurity.transaction.time_stamp`.

[**gsub**](https://www.elastic.co/guide/en/elasticsearch/reference/master/gsub-processor.html) processor
The date of month in modsecurity have leading space which can't be parsed by [JAVA date](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html). So we got to replace additional space by using processor gsub.

```yaml
- gsub:
    field: modsecurity.transaction.time_stamp
    pattern: "  "
    replacement: " "
```

[**date**](https://www.elastic.co/guide/en/elasticsearch/reference/master/date-processor.html) processor
After replace additional space, we will have correct date format `Tue Mar 8 10:34:39 2022`. Elasticseach date processor can be used for mapping your date by custom defined format. In this case it's `EEE MMM d HH:mm:ss yyyy`. The date processor define will be like this:

```yaml
- date:
    field: modsecurity.transaction.time_stamp
    formats:
    - "EEE MMM d HH:mm:ss yyyy"
```

But if you looked into document of date processor, default timezone will be UTC. But in modsecurity, `modseucrity.transaction.time_stamp` will be system timezone. We got to add timezone config in date processor. The config will be like this:

```yaml
- date:
    field: modsecurity.transaction.time_stamp
    formats:
    - "EEE MMM d HH:mm:ss yyyy"
    timezone: "-0800"
```

This still have issue that filebeat module will be use in different machine. You shouldn't hard-coded timezone like that. After searching document, we can add system timezone into variable `event.timezone` when config `audit.yml`. Use `add_locale` in audit.yml will achive this:

```yaml
processors:
  - add_locale: ~
  - decode_json_fields:
      fields: [message]
      process_array: true
      target: "modsecurity"
```

And the above date processor in ingest pipeline should written like this:

```yaml
- date:
    field: modsecurity.transaction.time_stamp
    target_field: '@timestamp'
    formats:
    - "EEE MMM d HH:mm:ss yyyy"
    timezone: "{{event.timezone}}"
```

We also add target_field to `@timestamp`, then the timestamp finally correct.

For writing other ingest pipeline processors, you should check document [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/pipeline-processor.html#pipeline-processor).

## 2\. Install modsecurity Filebeat Module

### 2.1 Enable modsecurity module of filebeat

The default location of filebeat will be in `/usr/share/filebeat`. You need to

1.  Copy your module to /usr/share/filebeat/module/modsecurity
    
2.  Change owner and permission of module
    Filebeat will verify directory permission for security reason. You should use command to make sure it's correct.
    

```bash
sudo chown -R root:root /usr/share/filebeat/module/modsecurity
sudo chmod -R g-w /usr/share/filebeat/module/modsecurity
```

3.  Create filebeat module yml setting in `/etc/filebeat/modules.d/modsecurity.yml`.
    In case your modsecurity JSON files locate in directory `/var/log/modsecurity/20220309/202203..../123123132.1231232`, then the yml will be like this:

```yaml
- module: modsecurity
  audit:
    enabled: true
    var.paths: ["/var/log/modsecurity/**"]
```

4.  Enable module by command and force-reload

```bash
sudo filebeat modules enable modsecurity
sudo /etc/init.d/filebeat force-reload
```

5.  Enable debug log
    In case you don't know if your filebeat module enabled or not, this settings in /etc/filebeat/filebeat.yml may help you identify problem:

```yaml
logging.level: debug
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

You can check log in `/var/log/filebeat/filebeat`.

### 2.2 Import Ingest Pipeline into Elasticsearch

After enable module without error in debug log of filebeat, we can finally lode ingest pipeline into Elasticseach

```bash
sudo filebeat setup --pipelines
```

After execute above command, you will see ingest pipeline in Kibana "Stack Management -> Ingest Node Pipelines":
![c99b5610ad2ae3d842f3e966c34b4ba3.png](../_resources/f59bf11ab7574532a40395c891ba32ba.png)

Click the line above, you can see ingest pipeline defined in module now imported.
![8c584a98832f49ee535a2ff3c043e220.png](../_resources/fc68d84546c44216a04ef0f9b30fb68f.png)

You can always delete the ingest pipeline definition above and re-import by command `filebeat setup --pipelines`.

## 3\. Feeding Log into Elasticsearch

### 3.1 Feed the first log and check successful or not

**In filebeat**, you should check debug log. There will be log like this:

```txt
2022-03-08T22:37:39.626+0800    DEBUG   [input] log/input.go:465        Check file for harvesting: /var/log/modsecurity/20220308/20220308-2201/20220308-220136-164674809611.578729   {"input_id": "2a72f7cc-cb63-4dc7-a02d-b776ac22748f"}
```

And this:

```txt
2022-03-08T22:11:10.135+0800    DEBUG   [processors]    processing/processors.go:203    Publish event: {
  "@timestamp": "2022-03-08T14:11:10.132Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "_doc",
    "version": "7.xx.0",
    "pipeline": "filebeat-xxxx-modsecurity-audit-pipeline"
  },
  "event": {
    ...
  }
}
```

It's means that you have success publish an event to Elasticsearch instanse.

**In Elasticsearch**, you should use query to check if record saved without error. Example query:

```
GET filebeat-*/_search
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "event.module":"modsecurity" } }
      ]
    }
  }
}
```

And the query result will have correct structure of your JSON data and you defined in ingest pipeline:
![aecb51d7999d107044c1eab71f4c8613.png](../_resources/8704d35ccd57444e860db518e06b609e.png)

**In Kibana**, you should check your filebeat index have correct fields mapping in "Stack Management -> Index Patterns"
![056b78921ebd130e866525b7d2b6a3b2.png](../_resources/2efab4ea37df4ff6896a00fa8412ce3d.png)

Adn "Index Management -> \[your current index\] -> Mappings"
![e53216a3d568292ea9a0687b62b368d0.png](../_resources/9091ca55a89a44b8af8ff9e32c493ef7.png)

**In Kibana Discover**, you can now search event with filter `event.module: modsecurity`
![9f58f9ef17539c126d190f95ba903a1d.png](../_resources/86b90c16983f4f94bb26a222ea8dd885.png)

### 3.2 Complete the module setup

Do some clear stuff to prevent further problem:

1.  Remember turn off debug log in /etc/filebeat/filebeat.yml
2.  modsecurity will periodly create directory and logs, write some cronjob to clear them. It will also prevent filebeat watch too many files.
3.  Practice [nested query syntax in Kibana](https://www.elastic.co/guide/en/kibana/current/kuery-query.html#_nested_field_queries). It's helpful when you want to filter logs and building dashboard.
4.  In case you want to remove test logs imported above, here is quick hint

```
POST filebeat-*/_delete_by_query
{
  "query": {
    "term": {
      "event.module": "modsecurity"
    }
  }
}
```