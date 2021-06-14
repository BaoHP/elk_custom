# ELK on Docker
### Host setup

* [Docker Engine](https://docs.docker.com/install/) version >= **17.05**
* [Docker Compose](https://docs.docker.com/compose/install/) version >= **1.20.0**
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

Để có thể dùng version khác, thay đổi tại file `.env`. Cẩn thận khi update version và đọc note dưới đây.

**:warning: [official upgrade instructions][upgrade] **

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

## Cài đặt

### Setting tài khoản đăng nhập


Đăng nhập bằng tài khoản và mật khẩu sau:

* user: *elastic*
* password: *admin*

Để tăng cường bảo mật nên sử dụng [built-in users][builtin-users].

1. Tự sinh password cho user

    ```console
    $ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
    ```

    Mật khẩu được tạo ngẫu nhiên. Note lại chúng

1. Không set password (_optional_)

    Xóa bỏ `ELASTIC_PASSWORD` trong `elasticsearch` service nằm trong Compose file
    (`docker-compose.yml`).

1. Thay đổi usernames và passwords trong config files

    Dùng `kibana_system` user (`kibana` cho bản <7.8.0) trong Kibana config
    (`kibana/config/kibana.yml`) và `logstash_system` trong Logstash config
    (`logstash/config/logstash.yml`) thay cho `elastic` user.

    Thay thế password của `elastic` trong Logstash pipeline file (`logstash/pipeline/logstash.conf`).

    *:information_source: Không dùng `logstash_system` user trong Logstash **pipeline** file, nó không đủ quyền để tạo chỉ mục. Tài liệu tham khảo [Configuring Security in Logstash][ls-security].*

    Xem [Configuration](#configuration) dưới đây.

1. Khởi động lại Kibana và Logstash để chạy các thay đổi

    ```console
    $ docker-compose restart kibana logstash
    ```

    *:information_source: Tài liệu tham khảo [Tutorial: Getting started with
    security][sec-tutorial].*

### Injecting data

Để Kibana vài phút khởi tạo, sau đó Kibana UI sẽ hiện thị trên đường dẫn <http://localhost:5601> và dùng các thông tin dưới để login:

* user: *elastic*
* password: *\<your generated elastic password>*

Gói ELK đã chạy, bạn có thể đẩy 1 số log vào. Logstash config sẽ cho phép bạn gửi dữ liệu thông qua TCP:

```console
# Dùng BSD netcat (Debian, Ubuntu, MacOS system, ...)
$ cat /path/to/logfile.log | nc -q0 localhost 5000
```

```console
# Dùng GNU netcat (CentOS, Fedora, MacOS Homebrew, ...)
$ cat /path/to/logfile.log | nc -c localhost 5000
```

Bạn có thể sử dụng data mẫu (sample data) từ Kibana.

Ở đây chúng ta sẽ sử dụng Filebeat. ( Xem phần Filebeat )

### Kibana index pattern mặc định

Lần đầu tiên chạy Kibana, bạn cần thêm index pattern.

#### Kibana web UI

*:information_source: Bạn cần tiêm data vào Logstash trước khi config 1 Logstash index pattern thông qua Kibana web UI.*

Chọn _Discover_ view trên Kibana from the ở bên trái sidebar. Bạn sẽ được nhắc tạo 1 index pattern. Enter
`logstash-*` để match với Logstash indices, tiếp tục chọn `@timestamp` như là time filter field. Ở bước chọn timestamp bạn có thể không chọn khi bạn đã xác định được field thay thế cho timestamp này. Cuối cùng click _Create index pattern_ và quay về _Discover_ view để xem log được đẩy ra.

Tham khảo [Connect Kibana with Elasticsearch][connect-kibana] và [Creating an index pattern][index-pattern].

#### Command line

Tạo index pattern thông qua Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.9.2' \
    -u elastic:<your generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

Pattern đã tạo sẽ tự động được đánh dấu là mẫu chỉ mục mặc định ngay sau khi giao diện người dùng Kibana được mở lần đầu tiên.

## Configuration

*:information_source: Config được sẽ không được thay đổi tự động. Bạn cần restart các component lại khi config được thay đổi. *

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

Chú ý đứng tại forder chứa filebeat.docker.yml

Mount volume "--volume="/logs:/logs:ro" \" chứa log sang container.

### Elasticsearch

Elasticsearch config ở file [`elasticsearch/config/elasticsearch.yml`][config-es].

Bạn cũng có thể chỉ định các tùy chọn bạn muốn ghi đè bằng cách đặt các biến môi trường bên trong tệp file Compose:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Tài liệu tham khảo: [Install Elasticsearch with Docker][es-docker].

### Kibana

Kibana config được lưu tại [`kibana/config/kibana.yml`][config-kbn].

Tài liệu tham khảo:

[Install Kibana with Docker][kbn-docker].

### Logstash

Logstash config được lưu tại [`logstash/config/logstash.yml`][config-ls].

Tài liệu tham khảo:

[Configuring Logstash for Docker][ls-docker].

Cách config, bố cục và cú pháp được trình bày rõ ràng ở link tài liệu Config Logstash phía trên.

Đặc biệt việc config hay xử lý những pattern trong đoạn log 1 cách phức tạp... bạn có thể áp dụng code ruby trong Logstash config.

```console
ruby {
		code => "
		require 'date'
        //event.get('timelog') get giá trị của field timelog và gán cho biến b, tương tự đối với biến c
		b = event.get('timelog')
		c = event.get('date')
        //Convert chuỗi string tháng ra số sau đó parse lại nó dưới dạng string.
		month = Date::ABBR_MONTHNAMES.index(c[0,3]).to_s
		month_cus = month.size > 1 ? month : '0' + month
		zero_str = c[4].strip.empty? ? '0' + c[5] : c[4,5]
        //xử lý để lấy được giá trị ngày tháng năm đúng chuẩn và gán cho biến d
		d = b[0,4] + ':' + month_cus + ':' + zero_str + c[6, c.size]
        //tạo field time và gán cho nó giá trị biến là biến d
		event.set('time', d);"
	}
```

[Grok-debugger][grock-debugger].

Dùng Grok để phân giải log ra các pattern. Chúng ta có thể dùng link trên để debugger xem các pattern đã chính xác chưa, hoặc sử dụng mục Discover để tự động phân giải log từ đó tìm ra cách cấu trúc pattern tốt nhất cho đoạn log của mình.

[Regex][regex].

Áp dụng thêm regex để có thể xử lý đoạn log 1 cách tối ưu nhất.

### Tắt tính năng trả phí

Thay đổi giá trị của Elasticsearch's `xpack.license.self_generated.type` từ `trial` sang `basic` ([License
settings][trial-license]).

Hoặc có thể thay đổi tại giao diện Kibana

### Đặt lại mật khẩu theo chương trình

Nếu vì bất kỳ lý do gì bạn không thể sử dụng Kibana để thay đổi mật khẩu của người dùng (bao gồm [built-in
users][builtin-users])), bạn có thể sử dụng API Elasticsearch để thay thế và đạt được kết quả tương tự

Trong ví dụ dưới đây, chúng tôi đặt lại mật khẩu của user `elastic` (notice "/user/elastic" trong URL):

```console
$ curl -XPOST -D- 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<your current elastic password> \
    -d '{"password" : "<your new password>"}'
```
### Bật các extensions

Một số tiện ích mở rộng có sẵn bên trong [`extensions`](extensions). Các tiện ích mở rộng này cung cấp các tính năng không phải là một phần của Elastic stack chuẩn, nhưng có thể được sử dụng để làm phong phú thêm với các tích hợp bổ sung.

Tài liệu cho các phần mở rộng này được cung cấp bên trong mỗi thư mục con riêng lẻ, trên cơ sở mỗi phần mở rộng. Một số yêu cầu thay đổi thủ công đối với cấu hình ELK mặc định.

### Chỉ định dung lượng bộ nhớ được sử dụng bởi một service

Mặc định, cả Elasticsearch và Logstash đều bắt đầu với [1/4 tổng bộ nhớ host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) được phân bổ cho Kích thước Heap JVM

Các tập lệnh khởi động cho Elasticsearch và Logstash có thể thêm các tùy chọn JVM bổ sung từ giá trị của một biến môi trường, cho phép người dùng điều chỉnh lượng bộ nhớ có thể được sử dụng bởi mỗi thành phần:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

Để đáp ứng các môi trường khan hiếm bộ nhớ (Docker cho Mac chỉ có sẵn 2 GB theo mặc định), phân bổ Kích thước đống được giới hạn theo mặc định là 256 MB cho mỗi dịch vụ trong `docker-compose.yml` file. Nếu bạn muốn ghi đè cấu hình JVM mặc định, hãy chỉnh sửa (các) biến môi trường phù hợp trong `docker-compose.yml` file.

Ví dụ, để tăng max JVM Heap Size cho Logstash:

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
