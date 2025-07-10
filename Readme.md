# Datadog Client IP ����ƥ��ȥ�ݡ���

## ����
���Υ�ݡ��Ȥϡ�Docker�Ķ��Ǥ�Datadog APM�ˤ����륯�饤�����IP���Ф�����ȥƥ��ȷ�̤�ޤȤ᤿��ΤǤ���

## �ƥ��ȴĶ�
- **Nginx**: ��С����ץ���
- **Tomcat**: Java���ץꥱ������󥵡��С�
- **Datadog Agent**: APM�ƻ�
- **Client**: curl����ƥʡ�10�ôֳ֤ǥꥯ������������

## �ƥ��ȷ��

### 1. ���饤�����IP��nginx access log�����
```
172.18.0.4 - - [10/Jul/2025:06:24:00 +0000] "GET /index.html HTTP/1.1" 200 93 "-" "curl/8.14.1" "-"
```

### 2. Nginx��Tomcat����������إå���
```
X-Real-IP: 172.18.0.4
X-Forwarded-For: 172.18.0.4
X-Forwarded-Proto: http
Host: nginx
Connection: close
User-Agent: curl/8.14.1
Accept: */*
```

### 3. Datadog Agent����������IP
```
peer.ipv4=172.18.0.2
http.client_ip=172.18.0.4
http.forwarded.ip=172.18.0.4
http.forwarded.proto=http
```

## ���פ�ȯ��

### ? ������������
- **Nginx����**: `proxy_set_header X-Real-IP $remote_addr;`
- **Tomcat����**: 
  - `-Ddd.trace.client-ip.header=x-real-ip`
  - `-Ddd.trace.client-ip.enabled=true`

### ? ���Ԥ�������
- **��ä��ѥ�᡼��̾**: `-Ddd.trace.client.ip.enabled=true` (�ɥåȶ��ڤ�)

## �ѥ�᡼��̾�ΰ㤤

### `-Ddd.trace.client-ip.enabled=true` ?
- **����������**: �ϥ��ե���ڤ�
- **�ºݤ�ư��**: ���饤�����IP���Ф�ͭ���ˤʤ�
- **���**: `http.client_ip=172.18.0.4` ����������Ͽ�����

### `-Ddd.trace.client-ip.header=x-real-ip` ?
- **����������**: �ϥ��ե���ڤ�
- **�ºݤ�ư��**: ���饤�����IP�إå��������
- **���**: X-Real-IP�إå������饯�饤�����IP���ɤ߼��

### `-Ddd.trace.client.ip.enabled=true` ?
- **��ä�����**: �ɥåȶ��ڤ�
- **�ºݤ�ư��**: �ѥ�᡼����ǧ������ʤ�
- **���**: ���饤�����IP���Ф�̵��

### `-Ddd.trace.client.ip.header=x-real-ip` ?
- **��ä�����**: �ɥåȶ��ڤ�
- **�ºݤ�ư��**: �ѥ�᡼����ǧ������ʤ�
- **���**: �إå������̵꤬��

## ������

### 1. Nginx���� (nginx.conf)
```nginx
location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    
    proxy_pass http://tomcat:8080/;
}
```

### 2. Tomcat���� (docker-compose.yaml)
```yaml
environment:
  JAVA_OPTS: >-
    -javaagent:/opt/dd-java-agent.jar
    -Ddd.trace.client-ip.header=x-real-ip
    -Ddd.trace.client-ip.enabled=true
```

## ������ˡ

### �Ķ���ư
```bash
# ����ƥʤΥӥ�ɤȵ�ư
docker-compose up -d --build
```

### �ȥ�ե��å��ƻ�
```bash
# HTTP�إå����γ�ǧ
docker-compose exec nginx tcpdump -i any -A -s 0 'tcp port 8080' | grep -A 20 "GET /index.html"

# Datadog trace log�γ�ǧ
docker-compose logs tomcat | grep "peer.ipv4" | tail -1
```

### ����ʬ�ϥ�����ץ�
```bash
# ���Ū��ʬ�ϡʥ��饤�����IP���إå�����Datadog trace log��
echo "=== Detailed Analysis ===" && echo "" && echo "1. Client's IP (from nginx access log):" && docker-compose logs nginx | grep "GET /index.html" | tail -1 && echo "" && echo "2. Headers nginx sending to tomcat:" && docker-compose exec nginx timeout 8 tcpdump -i any -A -s 0 'tcp port 8080' 2>/dev/null | grep -A 20 "GET /index.html" | head -25 && echo "" && echo "3. Datadog trace log (full entry):" && docker-compose logs tomcat | grep "peer.ipv4" | tail -1
```

### ���Ԥ������
- `http.client_ip=172.18.0.4` (�ºݤΥ��饤�����IP)
- `http.forwarded.ip=172.18.0.4` (ž�����줿IP)
- `peer.ipv4=172.18.0.2` (�ץ�������ƥʤ�IP - ����)

## ��ջ���

1. **�ѥ�᡼��̾**: `client-ip` (�ϥ��ե���ڤ�) ����� - ξ���Υѥ�᡼���Ȥ�
2. **����ƥʺƵ�ư**: Datadog Agent�ν�����˻��֤��������礬����
3. **�����**: ����塢����Υꥯ�����Ȥǰ��ꤹ��

## ����

�������ѥ�᡼��̾ `-Ddd.trace.client-ip.enabled=true` �� `-Ddd.trace.client-ip.header=x-real-ip` ����Ѥ��뤳�Ȥǡ�Datadog APM�ǼºݤΥ��饤�����IP�����Τ˸��ФǤ��뤳�Ȥ���ǧ����ޤ�����
