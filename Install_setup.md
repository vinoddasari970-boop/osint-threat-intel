# ELK Stack Installation & Configuration Guide (Ubuntu)

## Overview

This guide explains how to install and configure the ELK Stack:

* Elasticsearch
* Logstash
* Kibana

It also includes:

* MongoDB integration with Logstash
* Elasticsearch index verification
* Kibana data visualization setup

---

# Architecture

```text
OSINT Scripts
      ↓
MongoDB
      ↓
Logstash
      ↓
Elasticsearch
      ↓
Kibana Dashboard
```

---

# System Requirements

| Component | Recommended                 |
| --------- | --------------------------- |
| RAM       | 4 GB Minimum                |
| CPU       | 2 Core                      |
| OS        | Ubuntu 22.04 / 24.04        |
| Java      | Included with Elastic Stack |

---

# Step 1: Update System

```bash
sudo apt update && sudo apt upgrade -y
```

---

# Step 2: Install Required Packages

```bash
sudo apt install apt-transport-https wget curl gnupg -y
```

---

# Step 3: Import Elastic GPG Key

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
```

---

# Step 4: Add Elastic Repository

```bash
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

---

# Step 5: Update Package List

```bash
sudo apt update
```

---

# Step 6: Install Elasticsearch

```bash
sudo apt install elasticsearch -y
```

---

# Step 7: Configure Elasticsearch

Open configuration file:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Add or modify:

```yaml
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node
```

---

# Step 8: Start Elasticsearch

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

Check status:

```bash
sudo systemctl status elasticsearch
```

---

# Step 9: Verify Elasticsearch

```bash
curl localhost:9200
```

Expected output:

```json
{
  "name" : "ubuntu",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xxxx",
  "version" : {
    "number" : "8.x.x"
  }
}
```

---

# Step 10: Install Kibana

```bash
sudo apt install kibana -y
```

---

# Step 11: Configure Kibana

Open configuration:

```bash
sudo nano /etc/kibana/kibana.yml
```

Modify:

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

---

# Step 12: Start Kibana

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

Check status:

```bash
sudo systemctl status kibana
```

---

# Step 13: Access Kibana

Open browser:

```text
http://YOUR-IP:5601
```

---

# Step 14: Install Logstash

```bash
sudo apt install logstash -y
```

---

# Step 15: Verify Logstash

```bash
/usr/share/logstash/bin/logstash --version
```

---

# Step 16: Install MongoDB Plugin for Logstash

```bash
cd /usr/share/logstash
sudo bin/logstash-plugin install logstash-input-mongodb
```

Verify plugin:

```bash
sudo bin/logstash-plugin list | grep mongodb
```

---

# Step 17: Create Logstash Pipeline

Create configuration file:

```bash
sudo nano /etc/logstash/conf.d/mongodb.conf
```

Add configuration:

```ruby
input {
  mongodb {
    uri => "mongodb://localhost:27017/OSINT_Treat"

    collection => "osint_data"

    batch_size => 500

    placeholder_db_dir => "/var/lib/logstash/mongodb/"
    placeholder_db_name => "logstash_sqlite.db"
  }
}

filter {
  mutate {
    remove_field => ["_id"]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "threatintel"
  }

  stdout {
    codec => rubydebug
  }
}
```

---

# Step 18: Create Placeholder Directory

```bash
sudo mkdir -p /var/lib/logstash/mongodb
sudo chown -R logstash:logstash /var/lib/logstash/mongodb
```

---

# Step 19: Test Logstash Configuration

```bash
sudo /usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/mongodb.conf
```

Expected output:

```text
Configuration OK
```

---

# Step 20: Start Logstash

```bash
sudo systemctl enable logstash
sudo systemctl restart logstash
```

Check status:

```bash
sudo systemctl status logstash
```

---

# Step 21: Monitor Logstash Logs

```bash
sudo journalctl -u logstash -f
```

---

# Step 22: Verify Elasticsearch Indices

```bash
curl localhost:9200/_cat/indices?v
```

Expected output:

```text
threatintel
```

---

# Step 23: Verify Documents in Elasticsearch

```bash
curl -X GET "localhost:9200/threatintel/_search?pretty"
```

---

# Step 24: Create Data View in Kibana

Open Kibana:

```text
http://YOUR-IP:5601
```

Navigate:

```text
Stack Management → Data Views
```

Click:

```text
Create data view
```

Enter:

| Field         | Value        |
| ------------- | ------------ |
| Name          | threatintel  |
| Index pattern | threatintel* |

Save data view.

---

# Step 25: View Data in Kibana

Navigate:

```text
Analytics → Discover
```

Select:

```text
threatintel
```

Now MongoDB data will appear in Kibana.

---

# Example MongoDB Document

```json
{
  "ip": "8.8.8.8",
  "domain": "example.com",
  "source": "AlienVault OTX",
  "timestamp": "2026-05-13T09:00:00"
}
```

---

# Useful Commands

## Restart Services

```bash
sudo systemctl restart elasticsearch
sudo systemctl restart kibana
sudo systemctl restart logstash
```

---

## Check Service Status

```bash
sudo systemctl status elasticsearch
sudo systemctl status kibana
sudo systemctl status logstash
```

---

## View Elasticsearch Indices

```bash
curl localhost:9200/_cat/indices?v
```

---

## View Elasticsearch Documents

```bash
curl -X GET "localhost:9200/threatintel/_search?pretty"
```

---

# Common Errors & Fixes

## Logstash Plugin Missing

Install plugin:

```bash
sudo bin/logstash-plugin install logstash-input-mongodb
```

---

## Elasticsearch Connection Refused

Check Elasticsearch:

```bash
sudo systemctl status elasticsearch
```

---

## Kibana Not Opening

Check Kibana:

```bash
sudo systemctl status kibana
```

Allow firewall:

```bash
sudo ufw allow 5601
```

---

## Logstash Not Reading MongoDB

Check:

* Database name
* Collection name
* MongoDB service status

---

# Final Working Pipeline

```text
Python OSINT Scripts
        ↓
MongoDB Database
        ↓
Logstash Pipeline
        ↓
Elasticsearch Index
        ↓
Kibana Dashboard
```

---

# Conclusion

This setup creates a complete Threat Intelligence Platform (TIP) pipeline using the ELK Stack and MongoDB. Logstash collects data from MongoDB, processes it, sends it to Elasticsearch, and Kibana visualizes the threat intelligence data.
