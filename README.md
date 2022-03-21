# istioAuth0
> https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/

# 0. Create a L7 Aware Access Gateway, Nginx and Employee Application
This Web server is just HTTP not HTTPS. What I'm gonna chage is make the Gateway available for HTTPS access.
> https://github.com/developer-onizuka/hybridCloud#6-l7-aware-access

# 1. Create a root certificate and private key to sign the certificates for your services
```
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
```

# 2. Create a certificate and a private key for onprem.example.com
```
$ openssl req -out onprem.example.com.csr -newkey rsa:2048 -nodes -keyout onprem.example.com.key -subj "/CN=onprem.example.com/O=employee organization"
$ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in onprem.example.com.csr -out onprem.example.com.crt
```

# 3. Configure a TLS ingress gateway
```
$ kubectl create -n istio-system secret tls onprem-credential --key=onprem.example.com.key --cert=onprem.example.com.crt
$ kubectl describe -n istio-system secrets onprem-credential 
Name:         onprem-credential
Namespace:    istio-system
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1054 bytes
tls.key:  1704 bytes
```
```
$ kubectl apply -f ingress-gateway-L7-https.yaml
```

# 4. cURL
```
$ curl https://onprem.example.com -k -v --head
*   Trying 192.168.33.220:443...
* TCP_NODELAY set
* Connected to onprem.example.com (192.168.33.220) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=onprem.example.com; O=employee organization
*  start date: Mar 21 04:15:26 2022 GMT
*  expire date: Mar 21 04:15:26 2023 GMT
*  issuer: O=example Inc.; CN=example.com
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x5574016973b0)
> HEAD / HTTP/2
> Host: onprem.example.com
> user-agent: curl/7.68.0
> accept: */*
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 200 
HTTP/2 200 
< server: istio-envoy
server: istio-envoy
< date: Mon, 21 Mar 2022 05:33:33 GMT
date: Mon, 21 Mar 2022 05:33:33 GMT
< content-type: text/html; charset=utf-8
content-type: text/html; charset=utf-8
< set-cookie: .AspNetCore.Mvc.CookieTempDataProvider=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=/; samesite=lax; httponly
set-cookie: .AspNetCore.Mvc.CookieTempDataProvider=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=/; samesite=lax; httponly
< strict-transport-security: max-age=2592000
strict-transport-security: max-age=2592000
< x-envoy-upstream-service-time: 15
x-envoy-upstream-service-time: 15

< 
* Connection #0 to host onprem.example.com left intact
```
