## InfluxDB and Grafana:

### Overview of the Stack:
- **InfluxDB**: Time-series database for storing metrics.
- **Telegraf** (optional but recommended): Metric collection agent. Agent that collects metrics from server and writes to InfluxDB.
- **Grafana**: Visualization platform to display metrics.


### Architecture: 



### Install InfluxDB:


#### For Rocky Linux / RHEL-based systems:

_Add the InfluxData repository to the repository list:_

```
cat <<EOF | sudo tee /etc/yum.repos.d/influxdata.repo
[influxdata]
name = InfluxData Repository - Stable
baseurl = https://repos.influxdata.com/stable/\$basearch/main
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdata-archive_compat.key
EOF
```


```
cat /etc/yum.repos.d/influxdata.repo
```


_Install influxdb:_

```
yum install -y influxdb2
```


```
influxd version

InfluxDB v2.7.11 (git: fbf5d4ab5e) build_date: 2024-12-02T17:48:15Z
```




#### For Ubuntu and Debian:


_Add the InfluxData key to verify downloads and add the repository:_

```
curl --silent --location -O https://repos.influxdata.com/influxdata-archive.key
```


```
echo "943666881a1b8d9b849b74caebf02d3465d6beb716510d86a39f6c8e8dac7515  influxdata-archive.key" \
| sha256sum --check - && cat influxdata-archive.key \
| gpg --dearmor \
| sudo tee /etc/apt/trusted.gpg.d/influxdata-archive.gpg > /dev/null \
&& echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main' \
| sudo tee /etc/apt/sources.list.d/influxdata.list
```


_Install influxdb:_

```
sudo apt update

sudo apt install -y influxdb2
```



```
influx version

Influx CLI dev (git: a79a2a1b825867421d320428538f76a4c90aa34c) build_date: 2024-04-16T14:34:32Z
```



```
sudo systemctl start influxdb
sudo systemctl status influxdb
```


```
netstat -tlpn | grep 8086

tcp6       0      0 :::8086                 :::*                    LISTEN      281537/influxd
```





#### Web Access InfluxDB: 
Visit `http://<server_ip>:8086` and follow the web-based onboarding:

- Setup InfluxDB (UI or CLI):
  - Create a initial: `user`, `organization`, and `bucket`. 
  - Get/copy the API token – you'll need it for Telegraf.





#### Verify Data Influx:


```
influx org list --token <YOUR_API_TOKEN>
```


```
influx bucket list --org technbd --token <YOUR_API_TOKEN>
```


```
influx query 'from(bucket: "<BUCKET_NAME>") |> range(start: -5m)' --org <YOUR_ORG_NAME> --token <YOUR_API_TOKEN>
```






### Install Telegraf:

The **Telegraf agent** is a lightweight, plugin-driven server agent used to collect, process, and send metrics to outputs like **InfluxDB**, **Prometheus**, and many others.


_Install telegraf agent to each server:_

```
yum install -y telegraf
```



```
sudo apt install -y telegraf
```


```
systemctl status telegraf
systemctl enable telegraf
```



#### Configure Telegraf:

Set these fields in the `[[outputs.influxdb_v2]]` section and make sure the following plugins are enabled (they usually are by default):

```
vim /etc/telegraf/telegraf.conf


[global_tags]
[agent]
  interval = "10s"
  round_interval = true
  flush_interval = "10s"
  flush_jitter = "0s"

### OUTPUT PLUGINS:
[[outputs.influxdb_v2]]
  urls = ["http://127.0.0.1:8086"]
  token = "your_influxdb_token"
  organization = "technbd"
  bucket = "telegraf"


### INPUT PLUGINS:
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]

[[inputs.mem]]
[[inputs.system]]
[[inputs.net]]


```


```
systemctl restart telegraf
systemctl status telegraf
```



```
ls /etc/telegraf/telegraf.d/
```





---
---






## Install and setup InfluxDB in a container:

If you don’t specify InfluxDB initial setup options, you can set up manually later using the UI or CLI in a running container.


_Basic Docker Compose `docker-compose.yml` file:_

```
services:
  influxdb2:
    image: influxdb:2
    container_name: influxdb_ct1
    ports:
      - "8086:8086"
    volumes:
      - ./influxdb_data:/var/lib/influxdb2
      - ./influxdb_config:/etc/influxdb2
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: admin123
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: token123
      DOCKER_INFLUXDB_INIT_ORG: technbd
      DOCKER_INFLUXDB_INIT_BUCKET: telegraf
  grafana:
    image: grafana/grafana
    container_name: grafana_ct1
    ports:
      - '3000:3000'
    user: '0'
    volumes:
      - ./grafana_data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin321
    depends_on:
      - influxdb2

```



```
docker compose -f docker-compose.yml config
docker compose up -d
```


```
docker exec -it influxdb_ct1 influx config ls
```


```
docker exec -it influxdb_ct1 influx server-config
```



```
docker exec -it influxdb_ct1 bash
```


```
influx org list

ID                      Name
6657ed77ef624004        technbd
```



```
influx bucket list --org technbd

ID                      Name            Retention       Shard group duration    Organization ID         Schema Type
a95d325dafe936c9        _monitoring     168h0m0s        24h0m0s                 6657ed77ef624004        implicit
31c307ca904f7632        _tasks          72h0m0s         24h0m0s                 6657ed77ef624004        implicit
b3d27b4cf5659fe0        telegraf        infinite        168h0m0s                6657ed77ef624004        implicit
```





### Web Access InfluxDB: 
Visit `http://<server_ip>:8086` and follow the web-based onboarding:

- Create a initial: `user`, `organization`, and `bucket`. 
- Get/copy the API token – you'll need it for Telegraf.

1. Go to InfluxDB Web UI: `Home` - click  `Load Data` - `Sources` - search: [ ]


2. Create Token: `Home` - click  `Load Data` - `API Tokens` - click `Generate API Token` - select `Custom API Token`:
    - Buckets:
        - select your bucket: `telegraf`
        - click `Generate`





### Setup Grafana Dashboard:

1. Go to Grafana Web UI: `Home` - click `Data sources` - search: [influxdb]:
    - Name: `influxdb-technbd` 
    - Query language: `Flux`
    - HTTP:
        - URL: `http://192.168.10.193:8086`
    - Auth:
        - Basic auth: [ ] (-> Uncheck it)
    - Basic Auth Details:
        - User: N/A
        - Password: 
    - InfluxDB Details:
        - Organization: `technbd`
        - Token: token123
        - Default Bucket: `telegraf`
    - Click `Save`

2. Go to Grafana Web UI: `Home` - `Dashboards` - click `New` - `Import`: 
    - ID: `5955` or `11912`
    - Click `Load`
    - Name: Telegraf - system metrics
    - Select your DataSource: `influxdb-technbd`
    - Click `Import`




### Links:
- [InfluxDB v2 Docs](https://docs.influxdata.com/influxdb/v2/)
- [InfluxData - Package Repository](https://repos.influxdata.com/rhel/8/x86_64/)
- [InfluxDB Install on Linux](https://docs.influxdata.com/influxdb/v2/install/?t=Linux)
- [InfluxDB Install on Docker](https://docs.influxdata.com/influxdb/v2/install/?t=Docker)
- [Influxdb | hub.docker.com](https://hub.docker.com/_/influxdb)
- [Telegraf - system metrics](https://grafana.com/grafana/dashboards/5955-telegraf-system-metrics/)
- [InfluxDB Linux Server Telegraf](https://grafana.com/grafana/dashboards/11912-linux-server/)


