# Calico Live

## Requirements
Make sure you have the [session_002](https://github.com/frozenprocess/calico_live/tree/main/sessions/session_002) cluster up and running

### Fluent-bit
Helm installation command
```
helm upgrade --namespace monitoring --install fluent-bit fluent/fluent-bit  --set=image.repository=private-repo.multipass:5000/fluent/fluent-bit  --create-namespace
```

Fluent bit config used in the session:
```
kubectl apply -f -<<EOF
apiVersion: v1
data:
  custom_parsers.conf: |
    [PARSER]
        Name docker_no_time
        Format json
        Time_Keep Off
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
    [PARSER]
        Name ipt_parser
        Format regex
        Regex  /^(?<time>[^ ]* +[^ ]* +[^ ]*) (?<hostname>[^ ]*).*?calico-packet: IN=(?<ingress>.*?) OUT=(?<egress>.*?) (?:MAC=(?<mac>.*?) )?SRC=(?<srcip>.*?) DST=(?<dstip>.*?) LEN=(.*?) TOS=(.*?) PREC=(.*?) TTL=(?<TTL>.*?) ID=(?<ID>.*?) (?<flags>.*? )?PROTO=(?<protocol>.*?) SPT=(?<srcport>.*?) DPT=(?<dstport>.*?) (?:WINDOW=(?<window>.*?) RES=(?<RES>.*?) (?<tcpflags>.*?) URGP=(.*?)|LEN=(?<LEN>.*?))(?: MARK=(?<MARK>.*?))?$/
        Time_Keep On
        Time_Key time
        Time_Format %m %d %H:%M:%S
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File parsers.conf
        Parsers_File custom_parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020
        Health_Check On

    [INPUT]
        Name tail
        Path /var/log/kern.log
        Tag kernel
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
        Read_from_Head Off
    [FILTER]
        Name parser
        Match kernel
        Parser ipt_parser
        Key_Name log

    [OUTPUT]
        Name es
        Match kernel
        Host 192.168.122.1
        HTTP_User elastic
        HTTP_Passwd changeme
        Index calico-logs
        Retry_Limit False
        Suppress_Type_Name On
        
kind: ConfigMap
metadata:
  annotations:
    meta.helm.sh/release-name: fluent-bit
    meta.helm.sh/release-namespace: monitoring
  labels:
    app.kubernetes.io/instance: fluent-bit
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: fluent-bit
  name: fluent-bit
  namespace: monitoring
EOF
```
## Recording
https://www.youtube.com/watch?v=WBqU_8swpcM

## Resources
[elk stack](https://github.com/deviantony/docker-elk)
[Fluent-bit](https://github.com/fluent/fluent-bit-sandbox)