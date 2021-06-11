# ELK on Docker
### Host setup

* [Docker Engine](https://docs.docker.com/install/) version **17.05** hoặc cao hơn
* [Docker Compose](https://docs.docker.com/compose/install/) version **1.20.0** hoặc cao hơn
* 1.5 GB of RAM

*:information_source: Đảm bảo các quyền cần thiết tương tác với Docker daemon.*

Ports mặc định:

* 5044: Logstash Beats input
* 5000: Logstash TCP input
* 9600: Logstash monitoring API
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

## Sử dụng
### Lựa chọn Version

Bộ ELK Version (7.x).

To use a different version of the core Elastic components, simply change the version number inside the `.env` file. If
you are upgrading an existing stack, please carefully read the note in the next section.

**:warning: Always pay attention to the [official upgrade instructions][upgrade] for each individual component before
performing a stack upgrade.**

### Cài đặt

Clone repository và khởi chạy nó bằng Docker Compose

```console
$ docker-compose up
```
### Gỡ cài đặt

Tiến hành tắt docker-compose

```console
$ docker-compose down
```

Sau đó xóa hết container, images liên quan ELK

## Initial setup

### Setting tài khoản đăng nhập


Đăng nhập bằng tài khoản và mật khẩu sau:

* user: *elastic*
* password: *admin*

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security.

1. Initialize passwords for built-in users

    ```console
    $ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
    ```

    Passwords for all 6 built-in users will be randomly generated. Take note of them.

1. Unset the bootstrap password (_optional_)

    Remove the `ELASTIC_PASSWORD` environment variable from the `elasticsearch` service inside the Compose file
    (`docker-compose.yml`). It is only used to initialize the keystore during the initial startup of Elasticsearch.

1. Replace usernames and passwords in configuration files

    Use the `kibana_system` user (`kibana` for releases <7.8.0) inside the Kibana configuration file
    (`kibana/config/kibana.yml`) and the `logstash_system` user inside the Logstash configuration file
    (`logstash/config/logstash.yml`) in place of the existing `elastic` user.

    Replace the password for the `elastic` user inside the Logstash pipeline file (`logstash/pipeline/logstash.conf`).

    *:information_source: Do not use the `logstash_system` user inside the Logstash **pipeline** file, it does not have
    sufficient permissions to create indices. Follow the instructions at [Configuring Security in Logstash][ls-security]
    to create a user with suitable roles.*

    See also the [Configuration](#configuration) section below.

1. Restart Kibana and Logstash to apply changes

    ```console
    $ docker-compose restart kibana logstash
    ```

    *:information_source: Learn more about the security of the Elastic stack at [Tutorial: Getting started with
    security][sec-tutorial].*

### Injecting data

Give Kibana about a minute to initialize, then access the Kibana web UI by opening <http://localhost:5601> in a web
browser and use the following credentials to log in:

* user: *elastic*
* password: *\<your generated elastic password>*

Now that the stack is running, you can go ahead and inject some log entries. The shipped Logstash configuration allows
you to send content via TCP:

```console
# Dùng BSD netcat (Debian, Ubuntu, MacOS system, ...)
$ cat /path/to/logfile.log | nc -q0 localhost 5000
```

```console
# Dùng GNU netcat (CentOS, Fedora, MacOS Homebrew, ...)
$ cat /path/to/logfile.log | nc -c localhost 5000
```

Bạn có thể sử dụng data mẫu từ Kibana.

### Default Kibana index pattern creation

Lần đầu tiên chạy Kibana, bạn cần thêm index pattern.

#### Via the Kibana web UI

*:information_source: You need to inject data into Logstash before being able to configure a Logstash index pattern via
the Kibana web UI.*

Navigate to the _Discover_ view of Kibana from the left sidebar. You will be prompted to create an index pattern. Enter
`logstash-*` to match Logstash indices then, on the next page, select `@timestamp` as the time filter field. Finally,
click _Create index pattern_ and return to the _Discover_ view to inspect your log entries.

Refer to [Connect Kibana with Elasticsearch][connect-kibana] and [Creating an index pattern][index-pattern] for detailed
instructions about the index pattern configuration.

#### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.9.2' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

The created pattern will automatically be marked as the default index pattern as soon as the Kibana UI is opened for the
first time.

## Configuration

*:information_source: Configuration is not dynamically reloaded, you will need to restart individual components after
any configuration change.*

### Filebeat

Sử dụng output trong filebeat.yml

```console
# Dùng filebeat docker run
$ docker run -d \
--name=filebeat \
--user=root \
--volume="$(pwd)/filebeat.docker.yml:/usr/share/filebeat/filebeat.yml:ro" \
--volume="/logs:/logs:ro" \
docker.elastic.co/beats/filebeat:7.10.0 filebeat -e -strict.perms=false
```

Mount volume "--volume="/logs:/logs:ro" \" chứa log sang container.

### Elasticsearch

The Elasticsearch configuration is stored in [`elasticsearch/config/elasticsearch.yml`][config-es].

You can also specify the options you want to override by setting environment variables inside the Compose file:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Please refer to the following documentation page for more details about how to configure Elasticsearch inside Docker
containers: [Install Elasticsearch with Docker][es-docker].

### Kibana

The Kibana default configuration is stored in [`kibana/config/kibana.yml`][config-kbn].

Tài liệu tham khảo:

[Install Kibana with Docker][kbn-docker].

### Logstash

The Logstash configuration is stored in [`logstash/config/logstash.yml`][config-ls].

Tài liệu tham khảo:

[Configuring Logstash for Docker][ls-docker].

[Grok-debugger][grock-debugger].

[Regex][regex].

### How to disable paid features

Switch the value of Elasticsearch's `xpack.license.self_generated.type` option from `trial` to `basic` (see [License
settings][trial-license]).
### Reset a password programmatically

If for any reason your are unable to use Kibana to change the password of your users (including [built-in
users][builtin-users]), you can use the Elasticsearch API instead and achieve the same result.

In the example below, we reset the password of the `elastic` user (notice "/user/elastic" in the URL):

```console
$ curl -XPOST -D- 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<your current elastic password> \
    -d '{"password" : "<your new password>"}'
```
### How to enable the provided extensions

A few extensions are available inside the [`extensions`](extensions) directory. These extensions provide features which
are not part of the standard Elastic stack, but can be used to enrich it with extra integrations.

The documentation for these extensions is provided inside each individual subdirectory, on a per-extension basis. Some
of them require manual changes to the default ELK configuration.

### How to specify the amount of memory used by a service

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If you want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

[elastdocker]: https://github.com/sherifabdlnaby/elastdocker

[linux-postinstall]: https://docs.docker.com/install/linux/linux-postinstall/

[booststap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

[builtin-users]: https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
[ls-security]: https://www.elastic.co/guide/en/logstash/current/ls-security.html
[sec-tutorial]: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-getting-started.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html
[index-pattern]: https://www.elastic.co/guide/en/kibana/current/index-patterns.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html
[grock-debugger]: https://grokdebug.herokuapp.com
[regex]: https://regexr.com/

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html
