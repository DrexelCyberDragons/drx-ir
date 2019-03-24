# ELK (> 6.3) Hardening with X-Pack
1. Generate an SSL certificate
```
Openssl steps
Generate a CA
	1. Openssl genrsa –des3 –out rootCA.key 4096
	2. Openssl req –x509 –new –nodes –key rootCA.key -sha256 –days 1024 –out rootCA.crt
	3. Openssl genrsa –out CERTIFICATE.key 2048
	4. Openssl req –new –key CERTIFICATE.key -out CERTIFICATE.csr -config <(cat request)
	5. Openssl x509 –req –in CERTIFICATE.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial –out CERTIFICATE.crt -days 1024 –sha256 –extensions req_ext
	
Request file contents:
[req]
default_bits = 2048
prompt=no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
C=US
ST=Pennsylvania
L=Philadelphia
O=CyberDragons
OU=LinuxTeam
CN=LOGS

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1=CERTIFICATE
IP=xxx.xxx.xxx.xxx
```
2. Copy CERTIFICATE.crt, CERTIFICATE.key, and rootCA.crt to the directory where elasticsearch.yml is located.
- Most likely /etc/elasticsearch
3. Edit elasticsearch.yml to include the SSL certificate information, enable X-Pack security, and set the X-Pack license to trial.
```
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
curl -XPOST -u elastic:elastic_password 'http://<host>:<port>/_xpack/license/start_trial'
```
6. Set passwords for default accounts.
```bash
/usr/share/elasticsearch/bin/setup-passwords interactive
```
7. Edit kibana.yml to enable SSL and set the user and password to interact with elasticsearch.
```
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
Note: The user must have the 'kibana_user' role assigned for dashboard setup.
```
setup.kibana:
  host: "https://<host>:<port>"
  protocol: https
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

  protocol: "https"
  username: ""
  password: ""
  ssl.certificate_authorities: ["<path to root CA cert>"]
  ssl.certificate: "<path to ssl cert>"
  ssl.key: "<path to key>"
```

Documentation to add:
- Saved Searches
	- detail each query
- Beat Config breakdown
- Dashboard breakdown
	- add screenshot
- Elasticsearch config breakdown
	- note: we are using a trial license, a license is required to utilize the authentication module
- Kibana config breakdown