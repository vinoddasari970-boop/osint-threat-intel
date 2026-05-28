# MongoDB → Elasticsearch: Threat Intelligence Pipeline

Pipes threat intelligence data from MongoDB into Elasticsearch to create a searchable, visual threat landscape via Kibana.

## What This Does

- Reads threat data from MongoDB (`Threat.osint_threat` collection)
- Flattens nested `ips` and `domains` arrays into individual ES documents
- Indexes 300 IP records + 300 Domain records per document
- Makes data searchable and visualizable in Kibana

## Data Flow

```
MongoDB (Threat.osint_threat)
        ↓
Python Script (mongo_to_es.py)
        ↓
Elasticsearch (index: threat-osint)
        ↓
Kibana (Discover / Dashboards)
```

## Prerequisites

- Ubuntu 22.04+
- ELK Stack 8.x installed and running
- MongoDB running on `localhost:27017`
- Python 3.x

## Install Dependencies

```bash
# Install correct elasticsearch client version (must match ES version)
sudo pip3 install pymongo "elasticsearch==8.17.0" --break-system-packages
```

## Configuration

Edit `mongo_to_es.py` and update these values:

```python
MONGO_URI = "mongodb://localhost:27017"
MONGO_DB  = "Threat"
MONGO_COLLECTION = "osint_threat"

ES_HOST = "http://localhost:9200"
ES_USER = "elastic"
ES_PASS = "YOUR_ELASTIC_PASSWORD"
ES_INDEX = "threat-osint"
```

## Run

```bash
python3 mongo_to_es.py
```

Expected output:
```
Connecting to MongoDB...
MongoDB documents found: 1
Connecting to Elasticsearch...
ES cluster: elasticsearch
Index 'threat-osint' created.
Syncing data...
IP records: 300 | Domain records: 300 | Total: 600
Indexed: 600 | Failed: 0
```

## Verify in Elasticsearch

```bash
# Check document count
curl -X GET "http://localhost:9200/threat-osint/_count" -u elastic:YOUR_PASSWORD

# View sample document
curl -X GET "http://localhost:9200/threat-osint/_search?pretty&size=1" -u elastic:YOUR_PASSWORD
```

## View in Kibana

1. Open `http://localhost:5601`
2. Go to **Stack Management → Data Views**
3. Create data view:
   - Name: `threat-osint`
   - Index pattern: `threat-osint`
   - Time field: `ingested_at`
4. Go to **Analytics → Discover** to browse all records

## Index Schema

| Field | Type | Description |
|-------|------|-------------|
| record_type | keyword | `ip` or `domain` |
| ip | ip | IP address (IP records only) |
| domain | keyword | Domain name (domain records only) |
| source | keyword | Threat feed source (e.g. AlienVault OTX) |
| country | keyword | Country of IP |
| asn | long | Autonomous System Number |
| malicious_score | integer | Malicious score from threat feed |
| suspicious_score | integer | Suspicious score from threat feed |
| reputation | integer | Reputation score |
| added_at | date | When added to threat feed |
| generated_at | date | When MongoDB document was generated |
| ingested_at | date | When indexed into Elasticsearch |

## Common Issues

| Error | Fix |
|-------|-----|
| `TlsError` | Change `ES_HOST` to `http://` not `https://` |
| `BadRequestError 400 version 9` | Run `sudo pip3 install "elasticsearch==8.17.0" --break-system-packages --force-reinstall` |
| `date parse error` | Script handles this — replaces space with `T` in datetime strings |
| `Domain records: 0` | Domains are in separate `domains` array — script reads both `ips` and `domains` |