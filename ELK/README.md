# ELK (> 6.3) Hardening with X-Pack
1. Generate an SSL certificate
```
Openssl steps
Generate a CA
  1. openssl genrsa -des3 -out rootCA.key 4096
  2. openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
  3. openssl genrsa -out CERTIFICATE.key 2048
  4. openssl req -new -key CERTIFICATE.key -out CERTIFICATE.csr -config <(cat request)
  5. openssl x509 -req -in CERTIFICATE.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out CERTIFICATE.crt -days 1024 -sha256 -extensions req_ext -extfile request
	
Request file contents:
[req]
default_bits = 2048
prompt=no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
C=AU
ST=Some-State
O=Internet Widgits Pty Ltd
CN=HOSTNAME

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1=CERTIFICATE
IP=x.x.x.x

```
2. Copy CERTIFICATE.crt, CERTIFICATE.key, and rootCA.crt to the directory where elasticsearch.yml is located.
- Most likely /etc/elasticsearch
3. Edit elasticsearch.yml to include the SSL certificate information, enable X-Pack security, and set the X-Pack license to trial.
```
network.host: ["_local_", "x.x.x.x"]

xpack.security.enabled: true
xpack.license.self_generated.type: trial

xpack.security:
  http.ssl:
    key: <path to elasticsearch configuration directory>/CERTIFICATE.key
    certificate: <path to elasticsearch configuration directory>/CERTIFICATE.crt
    certificate_authorities: ["<path to elasticsearch configuration directory>/rootCA.crt"]
    enabled: true
  transport.ssl:
    key: <path to elasticsearch configuration directory>/CERTIFICATE.key
    certificate: <path to elasticsearch configuration directory>/CERTIFICATE.crt
    certificate_authorities: ["<path to elasticsearch configuration directory>/rootCA.crt"]
    enabled: true
```
4. Restart Elasticsearch.
5. Activate the trial license.
```bash
curl -XPOST -k -u elastic:elastic_password 'https://<host>:<port>/_xpack/license/start_trial?acknowledge=true'
curl -XPOST -k 'https://192.168.13.11:9200/_xpack/license/start_trial?acknowledge=true'
```
6. Set passwords for default accounts.
```bash
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```
7. Edit kibana.yml to enable SSL and set the user and password to interact with elasticsearch.
```
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.username: ""
elasticsearch.password: ""

server.ssl.enabled: true
server.ssl.certificate: <path to kibana configuration directory>/CERTIFICATE.crt
server.ssl.key: <path to kibana configuration directory>/CERTIFICATE.key
server.ssl.certificateAuthorities: ["<path to kibana configuration directory>/rootCA.crt"]

elasticsearch.ssl.certificate: <path to kibana configuration directory>/CERTIFICATE.crt
elasticsearch.ssl.key: <path to kibana configuration directory>/CERTIFICATE.key
elasticsearch.ssl.certificateAuthorities: ["<path to kibana configuration directory>/rootCA.crt"]
```
8. Restart Kibana.

# Winlogbeat Configuration Breakdown
### Windows Event Logs to forward
```
winlogbeat.event_logs:
  - name: Application
  - name: Security
  - name: System
  - name: Microsoft-Windows-Sysmon/Operational
  - name: Windows PowerShell
  - name: Microsoft-Windows-PowerShell/Operational
```

### Kibana Setup (only required for initial dashboard setup)
Note: The user must have the 'kibana_user' role assigned for dashboard setup. Additionally, it must have permission to create a new index in elasticsearch.
```
setup.kibana:
  host: "https://<host>:<port>"
  username: ""
  password: ""
  ssl.certificate_authorities: ["<path to root CA cert>"]
  ssl.certificate: "<path to ssl cert>"
  ssl.key: "<path to key>"
```

### Elasticsearch
Note: The user must have the remote_monitoring_agent role assigned.
```
output.elasticsearch:
  hosts: ["https://<host>:<port>"]
  username: ""
  password: ""
  ssl.certificate_authorities: ["<path to root CA cert>"]
  ssl.certificate: "<path to ssl cert>"
  ssl.key: "<path to key>"
```

# Elasticsearch Queries
Failed Logons: event ID 4625
Log Cleared: event ID 1102
Suspicious File Creates: event ID 11; File extensions such as exe, ps1, and dll
Suspicious Network Connections: Sysmon event ID 3; Filter down to foreign unknown remote IPs
Suspicious PowerShell: Sysmon event ID 1; Commandline includes rarely used switches such as no profile and base64 encoding
Suspicious Processes: Sysmon event ID 1; Normally unused executables such as mshta and msbuild
Sysmon Service Stopped: Sysmon event ID 4
Umanaged PowerShell: Sysmon event ID 7; Looks for powershell dlls in non-powershell processes
Registry Edits: Sysmon event IDs 12, 13, and 14; Looks for typical registry persistence mechanisms
Service/Task Creations: event IDs 4697, 4698, and 7045; New service and task creations
Remote Logons: event ID 4624; queries for logon types 3 and 10
User Modifications: event IDs 4724, 4726, 4738, and 4720; looks for changes to users

Documentation to add:
- Audit/Filebeat Config breakdown
- Dashboard breakdown
	- add screenshot