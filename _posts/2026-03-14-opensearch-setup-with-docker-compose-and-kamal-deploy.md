---
layout: post
title: OpenSearch setup with Docker Compose and Kamal Deploy
categories: Posts
tags:
- docker
- kamal
- rails
- opensearch
date: 2026-03-14 20:59 +0700
---

โพสต์นี้พาเชื่อม [OpenSearch](https://opensearch.org/) เข้ากับ Rails app — รัน cluster แบบ 2 node บนเครื่อง dev ด้วย Docker Compose แล้ว deploy ขึ้น production แบบ single-node เป็น accessory ของ [Kamal](https://kamal-deploy.org/) เมื่อจบ Rails app จะ index และ search ข้อมูลผ่าน Searchkick ได้ทั้งสอง environment โดยใช้โค้ดชุดเดียวกัน

## Prerequisites

- Rails 7+ application
- ติดตั้ง Docker และ Docker Compose บนเครื่อง dev
- พื้นฐาน Kamal deployment
- มี server สำหรับ Kamal เตรียมไว้แล้ว (สำหรับส่วน production)

## Getting Started

### Searchkick

[Searchkick](https://github.com/ankane/searchkick) เป็น search library ที่ใช้ได้กับทั้ง Elasticsearch และ OpenSearch เพิ่มลงใน `Gemfile` พร้อมกับ OpenSearch client:

```ruby
gem "searchkick"
gem "opensearch-ruby"
```
{: file="Gemfile"}

```sh
bundle install
```

ตั้งค่า Searchkick ให้ใช้ OpenSearch client ทั้ง dev (Docker Compose) และ prod (Kamal) ต่างใช้ HTTPS กับ self-signed certificate จึงปิด SSL verification และอ่าน credential จาก environment variable:

```ruby
Searchkick.client = OpenSearch::Client.new(
  host: ENV.fetch("OPEN_SEARCH_HOST", "https://blog-opensearch:9200"),
  user: "admin",
  password: ENV.fetch("OPENSEARCH_INITIAL_ADMIN_PASSWORD", "SamplePassword1!"),
  retry_on_failure: true,
  transport_options: {
    ssl: {
      verify: false
    },
    request: {
      timeout: 250
    }
  }
)

Searchkick.index_prefix = Rails.application.class.module_parent_name.underscore
```
{: file="config/initializers/searchkick.rb"}

อธิบาย option ที่สำคัญ:

- **`host`** — ค่า default `https://blog-opensearch:9200` ตรงกับ hostname ของ Kamal accessory (ดูในส่วน Kamal ด้านล่าง) ตอน dev override ด้วย `OPEN_SEARCH_HOST=https://localhost:9200`
- **`ssl: verify: false`** — OpenSearch มาพร้อม self-signed demo certificate ใช้แบบนี้พอสำหรับ dev และ prod ขนาดเล็ก ถ้าเปิดให้เข้าจากภายนอกควรเปลี่ยนเป็น cert จริง
- **`retry_on_failure: true`** — retry เวลา connection error ชั่วคราว ช่วยมากตอน container เพิ่ง restart
- **`index_prefix`** — เพิ่ม prefix ให้ index ตามชื่อ app/engine เพื่อให้หลาย app ใช้ cluster เดียวกันได้โดยไม่ชนกัน

เพิ่ม `searchkick` ใน model ที่ต้องการให้ค้นหาได้:

```ruby
class Post < ApplicationRecord
  searchkick
end
```
{: file="app/models/post.rb"}

### Docker Compose

สำหรับ dev จะรัน cluster แบบ 2 node พร้อม OpenSearch Dashboards การมี 2 node ใกล้เคียงกับ production มากกว่า single-node และช่วยให้เจอปัญหา cluster formation ตั้งแต่ตอนพัฒนา

```yaml
---
services:
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${OPENSEARCH_INITIAL_ADMIN_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ./certs:/usr/share/opensearch/config/certs
      - opensearch-data1:/usr/share/opensearch/data
    ports:
      - 9200:9200
      - 9600:9600
    networks:
      - opensearch-net
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node2
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m
      - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${OPENSEARCH_INITIAL_ADMIN_PASSWORD}
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ./certs:/usr/share/opensearch/config/certs
      - opensearch-data2:/usr/share/opensearch/data
    networks:
      - opensearch-net
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - 5601:5601
    expose:
      - '5601'
    environment:
      OPENSEARCH_HOSTS: '["https://opensearch-node1:9200","https://opensearch-node2:9200"]'
    networks:
      - opensearch-net

volumes:
  opensearch-data1:
  opensearch-data2:

networks:
  opensearch-net:
```
{: file="docker-compose.yml"}

อธิบายส่วนที่สำคัญ:

- **`bootstrap.memory_lock=true` + `ulimits.memlock: -1`** — ล็อก JVM heap ไว้ใน RAM ไม่ให้ swap ลง disk จำเป็นสำหรับ OpenSearch เพื่อให้ latency คงที่
- **`OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m`** — กำหนด JVM heap 512 MB เพียงพอสำหรับ dev ถ้าข้อมูลเยอะค่อยเพิ่ม (และเผื่อ RAM เครื่องด้วย)
- **`ulimits.nofile: 65536`** — OpenSearch เปิด file ค้างไว้เยอะ (หนึ่งไฟล์ต่อ shard segment) ค่า default ของ Linux น้อยเกินไป
- **Port `9200` / `9600`** — REST API และ performance analyzer เปิดเฉพาะ node1 ออกมาที่ host ส่วน node2 อยู่ใน network ภายในเท่านั้น
- **Dashboards ที่ `5601`** — UI ไว้ดู index และทดลอง query

เก็บ admin password ไว้ใน `.env` ข้างๆ ไฟล์ compose (และเพิ่มใน `.gitignore` ด้วย):

```sh
OPENSEARCH_INITIAL_ADMIN_PASSWORD=SamplePassword1!
```
{: file=".env"}

> Initial password ต้องผ่านเงื่อนไขความซับซ้อนของ OpenSearch — อย่างน้อย 8 ตัวอักษร มีทั้งพิมพ์ใหญ่ พิมพ์เล็ก ตัวเลข และอักขระพิเศษ ไม่งั้น container จะไม่ยอมเริ่มทำงาน
{: .prompt-warning }

เริ่ม cluster:

```sh
docker compose up -d
```

ปิด cluster:

```sh
docker compose down
```

### Verify it works

เช็ค cluster health จาก host:

```sh
curl -k -u admin:SamplePassword1! https://localhost:9200/_cluster/health
```

ควรได้ JSON ที่มี `"status": "green"` (หรือ `"yellow"` ถ้ามีแค่ node เดียวบูตขึ้นมา)

จากนั้นเปิด Rails console ลอง index และ search:

```ruby
Post.reindex
Post.search "hello"
```

### Kamal

ฝั่ง production รัน OpenSearch แค่ node เดียวเป็น Kamal accessory เพื่อให้ deployment ง่าย และเพียงพอกับ workload ขนาดเล็กถึงกลาง ค่อยขยายเป็น multi-node cluster (หรือใช้ managed service) เมื่อโตเกินแล้ว

ก่อนอื่นย้าย admin password ไปไว้ใน `.kamal/secrets` แทนการ commit ลง repo:

```sh
OPENSEARCH_INITIAL_ADMIN_PASSWORD=$(op read "op://Vault/OpenSearch/admin-password")
```
{: file=".kamal/secrets" }

แล้วประกาศ accessory ใน `config/deploy.yml`:

```yaml
accessories:
  opensearch:
    image: opensearchproject/opensearch:latest
    host: your-server.example.com
    port: 9200
    env:
      clear:
        discovery.type: single-node
      secret:
        - OPENSEARCH_INITIAL_ADMIN_PASSWORD
    directories:
      - data:/usr/share/opensearch/data
```
{: file="config/deploy.yml"}

ตั้ง env ของ Rails service ให้ชี้ไปที่ accessory ด้วย hostname ของ Kamal (`<app>-<accessory>` เช่น `blog-opensearch`):

```yaml
env:
  clear:
    OPEN_SEARCH_HOST: https://blog-opensearch:9200
  secret:
    - OPENSEARCH_INITIAL_ADMIN_PASSWORD
```
{: file="config/deploy.yml"}

Boot accessory แล้วเช็ค log เพื่อยืนยันว่าเริ่มทำงานเรียบร้อย:

```sh
kamal accessory boot opensearch
kamal accessory logs opensearch
```

หลังจาก `kamal deploy` ครั้งถัดไป Rails app จะคุยกับ OpenSearch ผ่าน internal network ของ Docker ได้เลย รัน `Post.reindex` หนึ่งครั้งบน production console เพื่อ populate ข้อมูลเข้า index

## Troubleshooting

**`max virtual memory areas vm.max_map_count [65530] is too low`**
OpenSearch ต้องใช้ memory map area เยอะกว่าค่า default ของ Linux แก้บน host ด้วย:

```sh
sudo sysctl -w vm.max_map_count=262144
```

ให้ค่าคงอยู่หลัง reboot ด้วยการเพิ่ม `vm.max_map_count=262144` ใน `/etc/sysctl.conf`

**Container exit ทันทีพร้อม error เรื่อง password**
Initial admin password ต้องมีพิมพ์ใหญ่ พิมพ์เล็ก ตัวเลข และอักขระพิเศษ (อย่างน้อย 8 ตัว) เลือก password ที่แข็งแรงขึ้นแล้วสร้าง container ใหม่ — เงื่อนไขนี้บังคับเฉพาะตอนบูตครั้งแรกเท่านั้น ถ้าจะเปลี่ยนทีหลังต้อง reset data volume

**`SSL_connect returned=1 errno=0 ... certificate verify failed`**
เลือกอย่างใดอย่างหนึ่ง — คงค่า `ssl: verify: false` ใน Searchkick initializer (โอเคสำหรับ self-signed cert) หรือ mount certificate จริงเข้า container แล้วชี้ Searchkick ไปที่ CA bundle ด้วย `ssl: ca_file: ...`

## Conclusion

ตอนนี้เรามี OpenSearch รันสองแบบโดยใช้โค้ด Rails ชุดเดียวกัน — cluster 2 node ผ่าน Docker Compose สำหรับ dev และ single-node เป็น Kamal accessory สำหรับ production Searchkick ซ่อน detail ของ client ไว้ ทำให้ใน model แค่เขียน `searchkick` แล้วเรียก `reindex` ก็พอ

การ self-host แบบนี้เหมาะกับงานที่อยากคุมทุกอย่างเองและคุมค่าใช้จ่ายได้ ถ้าไม่อยากดูแล cluster เอง (อัปเกรด backup snapshot scaling) ใช้ managed service อย่าง [AWS OpenSearch Service](https://aws.amazon.com/opensearch-service/) หรือ [Bonsai](https://bonsai.io/) คุ้มกว่า

## References

- [OpenSearch downloads](https://opensearch.org/downloads/)
- [OpenSearch Docker installation guide](https://opensearch.org/docs/latest/install-and-configure/install-opensearch/docker/)
- [Kamal accessories documentation](https://kamal-deploy.org/docs/configuration/accessories/)
- [Searchkick README](https://github.com/ankane/searchkick)
- <https://youtu.be/uEv8qxQv7OM?si=oAhhnVjx2SY9p141>
- <https://github.com/Instadeploy/logging-opensearch>
