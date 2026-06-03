# SOAR Pipeline — Setup Notes

Full installation reference for the Wazuh → Shuffle → TheHive pipeline. Documents every component, configuration decision, and issue encountered during deployment on Ubuntu 24.04 LTS.

**VM:** Ubuntu 24.04 LTS, 14GB RAM, SECURITY segment

---

## Component 1 — Docker

### Installation
Ubuntu 24.04 has compatibility issues with Docker's official repository — the standard GPG key verification fails using the `.gpg` binary format, and `docker-compose` from Docker's repo requires the `distutils` Python module which was removed in Python 3.12.

Installed from Ubuntu's repo instead:

```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
```

**Final versions:** Docker 29.1.3, Docker Compose 1.29.2

---

## Component 2 — Shuffle

### Installation

```bash
cd /opt
sudo git clone https://github.com/Shuffle/Shuffle shuffle
cd shuffle
sudo mkdir -p shuffle-database
sudo chown -R 1000:1000 shuffle-database
```

### .env Configuration

Create `/opt/shuffle/.env`:

```env
ENVIRONMENT_NAME=Shuffle
LIQUID_SANITIZE_INPUT=true
SHUFFLE_APP_DOWNLOAD_LOCATION=https://github.com/shuffle/python-apps
SHUFFLE_APP_FORCE_UPDATE=false
SHUFFLE_DEFAULT_USERNAME=admin
SHUFFLE_DEFAULT_PASSWORD=password
SHUFFLE_APP_HOTLOAD_FOLDER=./shuffle-apps
SHUFFLE_APP_HOTLOAD_LOCATION=./shuffle-apps
SHUFFLE_FILE_LOCATION=./shuffle-files
SHUFFLE_ENCRYPTION_MODIFIER=
BASE_URL=http://shuffle-backend:5001
SSO_REDIRECT_URL=http://localhost:3001
BACKEND_HOSTNAME=shuffle-backend
BACKEND_PORT=5001
FRONTEND_PORT=3001
FRONTEND_PORT_HTTPS=3443
OUTER_HOSTNAME=10.10.10.4
DB_LOCATION=./shuffle-database
DOCKER_API_VERSION=1.44
HTTP_PROXY=
HTTPS_PROXY=
SHUFFLE_PASS_WORKER_PROXY=TRUE
SHUFFLE_PASS_APP_PROXY=TRUE
SHUFFLE_INTERNAL_HTTP_PROXY=noproxy
SHUFFLE_INTERNAL_HTTPS_PROXY=noproxy
TZ=Europe/Amsterdam
SHUFFLE_SKIPSSL_VERIFY=true
IS_KUBERNETES=false
SHUFFLE_BASE_IMAGE_REPOSITORY=frikky
SHUFFLE_USE_GCHR_OVERRIDE_FOR_AUTODEPLOY=true
SHUFFLE_SWARM_BRIDGE_DEFAULT_INTERFACE=eth0
SHUFFLE_SWARM_BRIDGE_DEFAULT_MTU=1500
SHUFFLE_SWARM_CONFIG=inactive
SHUFFLE_CONTAINER_AUTO_CLEANUP=true
SHUFFLE_ORBORUS_EXECUTION_CONCURRENCY=5
SHUFFLE_HEALTHCHECK_DISABLED=false
SHUFFLE_ELASTIC=true
SHUFFLE_LOGS_DISABLED=true
SHUFFLE_WORKER_VERSION=latest
SHUFFLE_APP_SDK_VERSION=latest
SHUFFLE_RERUN_SCHEDULE=300
SHUFFLE_OPENSEARCH_URL=https://shuffle-opensearch:9200
SHUFFLE_OPENSEARCH_SKIPSSL_VERIFY=true
SHUFFLE_OPENSEARCH_USERNAME=admin
SHUFFLE_OPENSEARCH_PASSWORD=<opensearch_password>
OPENSEARCH_INITIAL_ADMIN_PASSWORD=<opensearch_password>
```

### docker-compose.yml Changes

Two changes from the default:

1. OpenSearch Java heap — Xms and Xmx **must be equal** or bootstrap check fails:
```yaml
- "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
```

2. Orborus swarm config — compose file hardcodes `run`, must be changed directly:
```yaml
- SHUFFLE_SWARM_CONFIG=inactive
```

Also add to the orborus environment section:
```yaml
- AUTH=<auth_token_from_shuffle_ui>
- ORG=<org_id_from_shuffle_ui>
- SHUFFLE_WORKER_VERSION=latest
- SHUFFLE_APP_SDK_VERSION=latest
- BASE_URL=http://shuffle-backend:5001
```

### Start

```bash
cd /opt/shuffle
sudo docker-compose up -d
```

Dashboard: `http://10.10.10.4:3001`

### Docker DNS Fix

`systemd-resolved` stub (`127.0.0.53`) was misbehaving, causing all container DNS lookups to fail. Fixed by pointing Docker directly to the pfSense gateway and Google DNS:

```bash
echo '{"dns": ["10.10.10.1", "8.8.8.8"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

---

## Component 3 — Java 11 (Cassandra / TheHive dependency)

Cassandra 4.1 uses JVM flags removed in Java 21 (`UseBiasedLocking`), causing immediate JVM crash. Java 11 is required.

```bash
sudo apt install -y openjdk-11-jdk
sudo update-alternatives --config java   # select Java 11
java -version  # verify: openjdk 11
```

---

## Component 4 — Cassandra

### Installation

```bash
wget -qO - https://downloads.apache.org/cassandra/KEYS | sudo apt-key add -
echo "deb https://debian.cassandra.apache.org 41x main" | sudo tee /etc/apt/sources.list.d/cassandra.list
sudo apt update && sudo apt install -y cassandra
```

### Issues Encountered

**Issue 1 — JVM flag removed in Java 21:**
`UseBiasedLocking` causes crash. Resolved by Java 11 pin above.

**Issue 2 — Deprecated JVM logging flags:**
Comment out in `/etc/cassandra/cassandra-env.sh`:
```bash
# JVM_OPTS="$JVM_OPTS -Xlog:gc=info..."
# JVM_OPTS="$JVM_OPTS -XX:+UseConcMarkSweepGC"
```

**Issue 3 — Missing data directories:**
```bash
sudo mkdir -p /var/lib/cassandra/{data,commitlog,saved_caches}
sudo chown -R cassandra:cassandra /var/lib/cassandra
sudo mkdir -p /var/log/cassandra
sudo chown cassandra:cassandra /var/log/cassandra
```

**Issue 4 — RAM exhaustion:**
Cap heap in `/etc/cassandra/jvm.options`:
```
-Xms512m
-Xmx512m
```

### Start

```bash
sudo systemctl enable --now cassandra
# Verify: nodetool status → should show UN (Up/Normal)
```

---

## Component 5 — TheHive 5

### Why Lucene Instead of Elasticsearch

TheHive 5's bundled JanusGraph is incompatible with both ES 7.x and ES 8.x in non-standard port configurations. Issues encountered:

- ES 7.17: `ElasticSearchIndex` instantiation failures, connection refused after 10 retries
- ES 8.19: Port 9200 conflict with Shuffle's OpenSearch; compatibility mode not recognised
- ES 8 downgrade prevention: after removing ES 8, reinstalling ES 7 failed with `IllegalStateException: nodes is a file written by ES 8.19.16 to prevent downgrade`

Resolved by removing Elasticsearch entirely and switching to Lucene — StrangeBee's own recommendation for single-node deployments.

### Installation

StrangeBee's Debian repository was unreachable from the lab VM. Downloaded directly:

```bash
wget -O /tmp/thehive.deb https://thehive.download.strangebee.com/5.4/deb/thehive_5.4.9-1_all.deb
sudo apt install -y /tmp/thehive.deb
```

### Configuration

`/etc/thehive/application.conf`:

```hocon
db.janusgraph {
  storage {
    backend = cql
    hostname = ["127.0.0.1"]
    cql {
      cluster-name = "Test Cluster"
      keyspace = thehive
    }
  }
  index.search {
    backend = lucene
    directory = /opt/thp/thehive/index
  }
}

storage {
  provider = localfs
  localfs.location = /opt/thp/thehive/files
}

play.http.secret.key = "<generated_secret>"
```

### JanusGraph Keyspace Issue

After initially configuring Elasticsearch, JanusGraph wrote `elasticsearch` as the index backend into the `thehive` Cassandra keyspace. This persisted even after switching to Lucene in the config file — JanusGraph reads the stored value and overrides local settings.

Fix — drop and recreate the keyspace:

```bash
~/.local/bin/cqlsh 127.0.0.1 9042 -e "DROP KEYSPACE thehive;"
sudo systemctl restart thehive
```

### Start

```bash
sudo systemctl enable --now thehive
```

Dashboard: `http://10.10.10.4:9000` — change the default admin password immediately on first login

### Organisation Setup

1. Change admin password on first login
2. **Admin → Organisations** → create: `SOC`
3. **Admin → Users** → create: `shuffle@thehive.local`
   - Type: Service
   - Add to SOC org with `org-admin` role
   - Set SOC as default organisation
   - Remove admin org membership
   - Generate API key → copy for Shuffle auth

**Why SOC org is required:** TheHive 5's `manageAlert` permission only exists in a non-admin organisation context. The global admin account cannot create alerts via API — a service account in a scoped org is required.

---

## Component 6 — Wazuh Integration

Add to `/var/ossec/etc/ossec.conf` inside `<ossec_config>`:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://10.10.10.4:3001/api/v1/hooks/<webhook_id></hook_url>
  <level>3</level>
  <alert_format>json</alert_format>
</integration>
```

```bash
sudo systemctl restart wazuh-manager
# Verify: sudo tail -f /var/ossec/logs/integrations.log
```

---

## Component 7 — Shuffle Workflow

### Orborus Registration

After setting `SHUFFLE_SWARM_CONFIG=inactive`, Orborus must be registered to the Shuffle backend with an auth token. Navigate to **Settings → Locations** in the Shuffle UI, copy the generated `docker run` command, and run it with `SHUFFLE_SWARM_CONFIG=inactive` substituted.

The compose-managed Orborus picks up the auth/org values from the environment variables added in the orborus section of `docker-compose.yml`.

### Workflow Nodes

**Node 1 — Webhook**
- Enable (click Start)
- Copy webhook URL for Wazuh ossec.conf

**Node 2 — Http (VirusTotal)**
- Method: GET
- URL: `https://www.virustotal.com/api/v3/ip_addresses/$exec.all_fields.data.srcip`
- Headers: `x-apikey=YOUR_VT_API_KEY`

**Node 3 — Http (TheHive)**
- Method: POST
- URL: `http://10.10.10.4:9000/api/v1/alert`
- Headers:
  ```
  Authorization=Bearer <shuffle_api_key>
  Content-Type=application/json
  X-Organisation=SOC
  ```
- Body:
  ```json
  {
    "title": "$exec.title",
    "description": "$exec.pretext - Rule: $exec.rule_id\n\nSource IP: $exec.all_fields.data.srcip\n\nVirusTotal Results:\n- Malicious: $http_2.body.data.attributes.last_analysis_stats.malicious\n- Suspicious: $http_2.body.data.attributes.last_analysis_stats.suspicious\n- Reputation: $http_2.body.data.attributes.reputation\n- Country: $http_2.body.data.attributes.country",
    "type": "external",
    "source": "wazuh",
    "sourceRef": "$exec.id",
    "severity": 2,
    "tags": ["wazuh", "$exec.all_fields.data.srcip"]
  }
  ```
- Timeout: 30

**Important:** The connection must be strictly Webhook → VT → TheHive. Any direct Webhook → TheHive connection will cause `$http_2` to be inaccessible in the TheHive node.

---

## Persistence Configuration

All Shuffle containers (frontend, backend, opensearch, orborus) are managed by `docker-compose` with `restart: unless-stopped`. The stack auto-starts on boot via Docker's systemd service.

OpenSearch takes 2-3 minutes to fully initialise on boot due to security plugin loading. The backend retries the connection automatically — no manual intervention needed.

### Memory Optimisation

After deployment the VM was using 11GB of 14GB RAM. Changes made:

- OpenSearch: `OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g` (down from default ~3.7GB)
- Cassandra: `-Xms512m -Xmx512m` in `jvm.options`
- Tenzir node (Shuffle add-on, unused): stopped and removed
- Steady-state usage: ~7.5-8GB, ~5.5GB available

---

## OpenSearch Troubleshooting Commands

```bash
# Check cluster health
curl -sk https://localhost:9200/_cluster/health -u admin:<opensearch_password> | python3 -m json.tool

# Clear stale Orborus leader lock
curl -s -X POST "https://localhost:9200/environments-000001/_update/<doc_id>" \
  -u admin:<opensearch_password> -k \
  -H "Content-Type: application/json" \
  -d '{"doc":{"orborus_uuid":"","running_ip":""}}'

# Clear stuck execution queue
curl -s -X POST "https://localhost:9200/workflowqueue-shuffle/_delete_by_query" \
  -u admin:<opensearch_password> -k \
  -H "Content-Type: application/json" \
  -d '{"query":{"match_all":{}}}'

# Delete stuck executions by ID
curl -s -X DELETE "https://localhost:9200/workflowexecution-000001/_doc/<execution_id>" \
  -u admin:<opensearch_password> -k
```

---

## Test Commands

### Send a test webhook with network IOC
```bash
curl -s -X POST "http://10.10.10.4:3001/api/v1/hooks/<webhook_id>" \
  -H "Content-Type: application/json" \
  -d "{\"severity\":3,\"pretext\":\"WAZUH Alert\",\"title\":\"Multiple failed SSH login attempts\",\"text\":\"Brute force detected\",\"rule_id\":\"5763\",\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%S.000-0400)\",\"id\":\"test-001\",\"all_fields\":{\"data\":{\"srcip\":\"<test_ip>\",\"dstip\":\"10.10.10.3\",\"dstport\":\"22\"}}}"
```

### Query TheHive alerts via API
```bash
curl -s "http://10.10.10.4:9000/api/alert" \
  -H "Authorization: Bearer <thehive_api_key>" \
  -H "X-Organisation: SOC" | python3 -m json.tool | grep '"title"'
```

### Check Shuffle health
```bash
curl -s http://10.10.10.4:5001/api/v1/health | python3 -m json.tool | grep '"success"'
```
