# Datadog Client IP 設定テストレポート

## 概要
このレポートは、Docker環境でのDatadog APMにおけるクライアントIP検出の設定とテスト結果をまとめたものです。

## テスト環境
- **Nginx**: リバースプロキシ
- **Tomcat**: Javaアプリケーションサーバー
- **Datadog Agent**: APM監視
- **Client**: curlコンテナ（10秒間隔でリクエスト送信）

## テスト結果

### 1. クライアントIP（nginx access logから）
```
172.18.0.4 - - [10/Jul/2025:06:24:00 +0000] "GET /index.html HTTP/1.1" 200 93 "-" "curl/8.14.1" "-"
```

### 2. NginxがTomcatに送信するヘッダー
```
X-Real-IP: 172.18.0.4
X-Forwarded-For: 172.18.0.4
X-Forwarded-Proto: http
Host: nginx
Connection: close
User-Agent: curl/8.14.1
Accept: */*
```

### 3. Datadog Agentが受信したIP
```
peer.ipv4=172.18.0.2
http.client_ip=172.18.0.4
http.forwarded.ip=172.18.0.4
http.forwarded.proto=http
```

## 重要な発見

### ? 成功した設定
- **Nginx設定**: `proxy_set_header X-Real-IP $remote_addr;`
- **Tomcat設定**: 
  - `-Ddd.trace.client-ip.header=x-real-ip`
  - `-Ddd.trace.client-ip.enabled=true`

### ? 失敗した設定
- **誤ったパラメータ名**: `-Ddd.trace.client.ip.enabled=true` (ドット区切り)

## パラメータ名の違い

### `-Ddd.trace.client-ip.enabled=true` ?
- **正しい形式**: ハイフン区切り
- **実際の動作**: クライアントIP検出が有効になる
- **結果**: `http.client_ip=172.18.0.4` が正しく記録される

### `-Ddd.trace.client-ip.header=x-real-ip` ?
- **正しい形式**: ハイフン区切り
- **実際の動作**: クライアントIPヘッダーを指定
- **結果**: X-Real-IPヘッダーからクライアントIPを読み取り

### `-Ddd.trace.client.ip.enabled=true` ?
- **誤った形式**: ドット区切り
- **実際の動作**: パラメータが認識されない
- **結果**: クライアントIP検出が無効

### `-Ddd.trace.client.ip.header=x-real-ip` ?
- **誤った形式**: ドット区切り
- **実際の動作**: パラメータが認識されない
- **結果**: ヘッダー指定が無効

## 設定手順

### 1. Nginx設定 (nginx.conf)
```nginx
location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    
    proxy_pass http://tomcat:8080/;
}
```

### 2. Tomcat設定 (docker-compose.yaml)
```yaml
environment:
  JAVA_OPTS: >-
    -javaagent:/opt/dd-java-agent.jar
    -Ddd.trace.client-ip.header=x-real-ip
    -Ddd.trace.client-ip.enabled=true
```

## 検証方法

### 環境起動
```bash
# コンテナのビルドと起動
docker-compose up -d --build
```

### トラフィック監視
```bash
# HTTPヘッダーの確認
docker-compose exec nginx tcpdump -i any -A -s 0 'tcp port 8080' | grep -A 20 "GET /index.html"

# Datadog trace logの確認
docker-compose logs tomcat | grep "peer.ipv4" | tail -1
```

### 統合分析スクリプト
```bash
# 包括的な分析（クライアントIP、ヘッダー、Datadog trace log）
echo "=== Detailed Analysis ===" && echo "" && echo "1. Client's IP (from nginx access log):" && docker-compose logs nginx | grep "GET /index.html" | tail -1 && echo "" && echo "2. Headers nginx sending to tomcat:" && docker-compose exec nginx timeout 8 tcpdump -i any -A -s 0 'tcp port 8080' 2>/dev/null | grep -A 20 "GET /index.html" | head -25 && echo "" && echo "3. Datadog trace log (full entry):" && docker-compose logs tomcat | grep "peer.ipv4" | tail -1
```

### 期待される結果
- `http.client_ip=172.18.0.4` (実際のクライアントIP)
- `http.forwarded.ip=172.18.0.4` (転送されたIP)
- `peer.ipv4=172.18.0.2` (プロキシコンテナのIP - 正常)

## 注意事項

1. **パラメータ名**: `client-ip` (ハイフン区切り) を使用 - 両方のパラメータとも
2. **コンテナ再起動**: Datadog Agentの初期化に時間がかかる場合がある
3. **一貫性**: 設定後、数回のリクエストで安定する

## 結論

正しいパラメータ名 `-Ddd.trace.client-ip.enabled=true` と `-Ddd.trace.client-ip.header=x-real-ip` を使用することで、Datadog APMで実際のクライアントIPを正確に検出できることが確認されました。
