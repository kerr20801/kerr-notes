# ELK 三節點叢集用 Docker host-network 的原因與坑

> **環境**：Elasticsearch 三節點，每個節點是獨立 VM，各自跑 Docker

---

## 為什麼選 host-network

ELK 叢集節點之間要互相溝通有兩條通道：

- **9200**：HTTP API（Kibana、應用程式查詢）
- **9300**：Transport（節點之間的 cluster formation、shard 同步）

用 bridge 網路的問題：Docker 的 bridge 是 NAT，container 看到的是虛擬 IP（`172.17.x.x`）。Elasticsearch 在 `network.publish_host` 公告自己的位址給其他節點時，會公告這個虛擬 IP——其他 VM 上的節點根本連不到。

要讓三節點跨 VM 正確 form cluster，最直接的方式是 host-network：container 直接共用 host 的網路介面，ES 看到的是真實 IP，公告給其他節點的也是真實 IP。

效能也是原因之一：host-network 少了 NAT 和 veth pair 的 overhead，對 ELK 這種 I/O 密集的服務有意義。

---

## docker-compose 設定

```yaml
services:
  elasticsearch:
    image: elasticsearch:8.x.x
    network_mode: host
    environment:
      - node.name=es01
      - cluster.name=my-cluster
      - discovery.seed_hosts=192.168.1.11,192.168.1.12,192.168.1.13
      - cluster.initial_master_nodes=es01,es02,es03
      - network.host=0.0.0.0
      - xpack.security.enabled=true
    volumes:
      - ./data:/usr/share/elasticsearch/data
      - ./certs:/usr/share/elasticsearch/config/certs
```

`network_mode: host` 之後 `ports:` 那段完全沒用，不要寫。寫了不會報錯，但也不會生效，只會讓人以為有在做 port mapping。

---

## 最常踩的坑

### 1. `network.host` 沒改

預設值是 `_local_`，等於只 bind 到 `127.0.0.1`。其他節點連不進來，cluster 永遠 form 不起來，log 裡會一直看到：

```
master not discovered yet, this node has not previously joined a bootstrapped cluster
```

改成 `0.0.0.0`（接受所有介面），或直接填 VM 的實際 IP。

### 2. `discovery.seed_hosts` 填 container IP

有時候從 bridge 環境移過來，`seed_hosts` 還留著 `172.17.x.x` 的 container IP。host-network 模式下這些位址根本不存在，discovery 會靜默失敗，cluster 只有一個節點，health 是 yellow。

填各節點 VM 的真實 IP。

### 3. TLS 憑證的 SAN 沒包真實 IP

啟用 `xpack.security` 後，節點之間的 transport 連線會驗證憑證。憑證的 Subject Alternative Name 要包含：

- 節點的 VM IP
- `localhost`（本機 health check 用）

只填 `localhost` 的話，跨節點連線時 TLS handshake 會失敗，`_cluster/health` 會顯示節點連不上，但 log 裡的錯誤訊息不夠直白，容易以為是 firewall 問題。

產憑證時確認 SAN：

```bash
openssl x509 -in node.crt -text -noout | grep -A1 "Subject Alternative"
```

### 4. Kibana 的 `elasticsearch.hosts` 填 localhost

Kibana 如果也是 host-network，本機連 ES 用 `localhost:9200` 沒問題。但如果 Kibana 在另一台 VM，要填 ES 節點的真實 IP，不能填 `localhost`。

---

## 驗證叢集正確 form 起來

```bash
# 確認三個節點都在
curl -sk -u elastic:yourpassword https://localhost:9200/_cat/nodes?v

# 確認 health 是 green
curl -sk -u elastic:yourpassword https://localhost:9200/_cluster/health?pretty
```

yellow = 有 shard unassigned（通常是節點數不足或憑證問題）
red = 有 primary shard 遺失（資料問題）

新叢集剛起來時 yellow 是正常的（replica shard 還沒分配）。等三個節點都 join 之後應該自動變 green。如果一直 yellow，去看 `_cluster/allocation/explain`。
