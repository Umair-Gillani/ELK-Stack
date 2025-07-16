# Pre-Requisites

**step1:** Stop the firewall so ELK ports are not blocked.

```bash
systemctl stop firewalld.service
```

**step2:** Temporarily disable SELinux enforcement.

```bash
setenforce 0
```

**step3:** Set the host’s name for easy identification.

```bash
hostnamectl set-hostname ELK-1
```

**step4:** Start a fresh shell session so the new hostname shows up everywhere.

```bash
bash
```

---

# Elastic search

**step1:** Import Elastic’s official GPG key so RPM signatures are trusted.

```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

**step2:** Download the Elasticsearch 9.0.3 RPM and its checksum.

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.0.3-x86_64.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.0.3-x86_64.rpm.sha512
```

**step3:** (Optional) Verify the checksum for integrity.

```bash
# shasum -a 512 -c elasticsearch-9.0.3-x86_64.rpm.sha512
```

**step4:** Install Elasticsearch.

```bash
sudo rpm --install elasticsearch-9.0.3-x86_64.rpm
# output of the above command: The generated password for the elastic built-in superuser is : WrN9yZrv=0hJxbw-CShc
```

**step5:** Edit the main config to form a single-node, unsecured demo cluster.

```bash
vim /etc/elasticsearch/elasticsearch.yml
```
```
# Inside the file set/ensure:
cluster.name: elasticsearch-demo
network.host: 0.0.0.0
transport.host: 0.0.0.0
#cluster.initial_master_nodes: ["localhost.localdomain"]   (comment out this line for single node ELK cluster)
xpack.security.enabled: false
discovery.type: single-node
```

**step6:** Reload unit files and start Elasticsearch at boot right away.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch.service
```

**step7:** Test cluster health with that password.

```bash
curl -u elastic:'<password_here>' http://localhost:9200/_cluster/health?pretty
```

**step8:** Give the JVM a fixed 2 GB heap (adjust to suit your box).

```bash
echo '-Xmx2g' >> /etc/elasticsearch/jvm.options.d/heap.options # adjust 2g → value for your box
```


## **Elasticsearch – Multi-DC & Replication**

**step1:** Configure both VMs to join the same cluster.

```bash
vim /etc/elasticsearch/elasticsearch.yml      # on BOTH VMs
cluster.name: elk-demo-cluster
node.name: elk-node-1            # ← change to elk-node-2 on node-2
network.host: 0.0.0.0            # or the node’s exact IP
discovery.seed_hosts: ["10.1.122.150","10.1.122.250"]
cluster.initial_master_nodes: ["elk-node-1","elk-node-2"]
xpack.security.enabled: false    # keep off in lab; enable in prod
```

**step2:** Start (or restart) Elasticsearch on each node.

```bash
sudo systemctl start elasticsearch.service
```

**step3:** Patch any pre-existing indices so they gain one replica.

```bash
for idx in $(curl -s 10.1.122.150:9200/_cat/indices?h=index); do
  curl -XPUT 10.1.122.150:9200/"$idx"/_settings \
       -H 'Content-Type: application/json' \
       -d'{"number_of_replicas":1}'
done
```

**step4:** Validate cluster settings, node list, health, and shard layout.

```bash
curl 10.1.122.150:9200/_cluster/settings?pretty
curl -s http://10.1.122.150:9200/_cat/nodes?v
curl -s http://10.1.122.150:9200/_cluster/health?pretty   # <- "number_of_data_nodes" : 2
curl -s 10.1.122.150:9200/_cat/shards?v
```
---

## **Point Logstash & Beats at the New Cluster**

Update every **output** section so data is sent to both Elasticsearch nodes:

```ruby
output {
  elasticsearch {
    hosts => ["http://10.1.122.150:9200","http://10.1.122.250:9200"]
  }
}
```

For Beats (Filebeat, Metricbeat, etc.) the equivalent block is:

```yaml
output.elasticsearch:
  hosts: ["http://10.0.0.11:9200","http://10.0.0.12:9200"]
```

After editing, restart Logstash or the relevant Beat to apply the change.

---

# Kibana

**step1:** Download the Kibana 9.0.3 RPM and its checksum.

```bash
wget https://artifacts.elastic.co/downloads/kibana/kibana-9.0.3-x86_64.rpm
wget https://artifacts.elastic.co/downloads/kibana/kibana-9.0.3-x86_64.rpm.sha512
```

**step2:** Verify the RPM’s checksum.

```bash
shasum -a 512 -c kibana-9.0.3-x86_64.rpm.sha512
```

**step3:** Install Kibana.

```bash
sudo rpm --install kibana-9.0.3-x86_64.rpm
```

**step4:** Point Kibana at the local cluster and open it to all IPs.

```bash
vim /etc/kibana/kibana.yml
```
```
server.host: 0.0.0.0
```
Kibana Failed when adding below lines
```
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "elastic"
elasticsearch.password: "WrN9yZrv=0hJxbw-CShc"
```

**step5:** Reload unit files and start Kibana at boot.

```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable --now kibana.service
```

**step6:** (Optional) Regenerate an enrollment token if you later enable security.

```bash
/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

---

### logstash

**step1:** Install Java 17 (Logstash’s runtime).

```bash
sudo yum install -y java-17-openjdk
```

**step2:** Add the Elastic 9.x YUM repo for Logstash packages.

```bash
cat > /etc/yum.repos.d/logstash.repo <<'EOF'
[logstash-9.x]
name=Elastic repository for 9.x packages
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
EOF
```

**step3:** Install Logstash.

```bash
sudo yum install -y logstash
```

**step4:** Define a pipeline that parses Nginx access-/error-logs and ships to Elasticsearch.

```bash
cat > /etc/logstash/conf.d/nginx_logs.conf <<'EOF'
input {
  file {
    path => ["/var/log/nginx/access.log", "/var/log/nginx/error.log"]
    start_position => "beginning"
    sincedb_path   => "/var/lib/logstash/.sincedb_nginx"
    type           => "nginx"
  }
}
filter {
  if [path] =~ "access" {
    grok {
      match => { "message" =>
        '%{IPORHOST:client} - %{DATA:ident} %{DATA:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:method} %{DATA:request}(?: HTTP/%{NUMBER:http_version})?)" %{NUMBER:status} (?:%{NUMBER:bytes}|-) "(?:%{URI:referrer}|-)" "(?:%{DATA:agent}|-)"'
      }
    }
    mutate { convert => { "bytes" => "integer" "status" => "integer" } }
  } else if [path] =~ "error" {
    grok {
      match => { "message" =>
        '(?<timestamp>%{YEAR}/%{MONTHNUM}/%{MONTHDAY} %{TIME}) \[%{DATA:level}\] %{NUMBER:pid}#%{NUMBER:tid}: (\*%{NUMBER:connection})? %{GREEDYDATA:error_message}, client: %{IPORHOST:client}, server: %{DATA:server}, request: "%{DATA:request}", host: "%{DATA:host}"'
      }
    }
  }
  date {
    match  => ["timestamp","dd/MMM/YYYY:H:m:s Z","YYYY/MM/dd HH:mm:ss"]
    target => "@timestamp"
  }
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-%{+YYYY.MM.dd}"
  }
  # stdout { codec => rubydebug }  # optional local debug
}
EOF
```

**step5:** Enable and start Logstash immediately.

```bash
sudo systemctl enable --now logstash.service
```

**step6:** Tail the last 60 Logstash journal lines for quick status.

```bash
journalctl -u logstash -n 60 --no-pager
```

**step7:** Confirm the pipeline is loaded via the HTTP API.

```bash
curl -s http://localhost:9600/_node/pipelines?pretty | head -n 40
```

**step8:** Grant Logstash read access to existing and future Nginx logs.

```bash
# a. One-time access for current files
sudo setfacl -m u:logstash:r /var/log/nginx/*.log

# b. Allow directory traversal
sudo setfacl -m u:logstash:rx /var/log/nginx

# c. Default ACL so new files inherit read permissions
sudo setfacl -d -m u:logstash:r /var/log/nginx
```

**step9:** Verify Logstash can read the logs.

```bash
sudo -u logstash head -n 3 /var/log/nginx/access.log
sudo -u logstash head -n 3 /var/log/nginx/error.log
```

**step10:** Restart Logstash to pick up any changes.

```bash
systemctl restart logstash
```

**step11:** Confirm the daily Nginx index now exists in Elasticsearch.

```bash
curl -s http://localhost:9200/_cat/indices/nginx-*?v
```
