# Deploy API with JWT authentication with Istio and Auth0
> https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/

# 1. Create a L7 Aware Access Gateway, Nginx and Employee Application
You can use the [URL](https://github.com/developer-onizuka/hybridCloud#6-l7-aware-access) to create these resources.
But the gateway is not HTTPS but HTTP. First of all, What I'm gonna chage is make the Gateway available for HTTPS access.

  1) [Configure a TLS ingress gateway access](https://github.com/developer-onizuka/istioAuth0#2-configure-a-tls-ingress-gateway-gateway-available-for-https-access) <br>
  2) [Create RequestAuthentication and AuthorizationPolicy](https://github.com/developer-onizuka/istioAuth0#3-create-requestauthentication-and-authorizationpolicy) <br>
  3) [Create JWT for Employee](https://github.com/developer-onizuka/istioAuth0#5-create-jwt-for-employee) <br>
  4) [Access with JWT](https://github.com/developer-onizuka/istioAuth0#7-access-with-jwt) <br>

# 2. Configure a TLS ingress gateway (HTTPS)
# 2-1. Create a root certificate and private key to sign the certificates for your services
This is self-signed certificate with openssl. Not suitable for public web servers.
>https://business.xserver.ne.jp/option/ssl/about_ssl.php
```
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
```

# 2-2. Create a certificate and a private key for onprem.example.com
```
$ openssl req -out onprem.example.com.csr -newkey rsa:2048 -nodes -keyout onprem.example.com.key -subj "/CN=onprem.example.com/O=employee organization"
$ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in onprem.example.com.csr -out onprem.example.com.crt
```

# 2-3. Configure a TLS ingress gateway
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

# 2-4. cURL
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

# 2-5. 404 Error
```
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 404 
< date: Mon, 21 Mar 2022 09:05:40 GMT
< server: istio-envoy
< 
* Connection #0 to host animals.example.com left intact
```

If you find 404 error, then you might use the same TLS certificate. Configuring more than one gateway using the same TLS certificate will cause browsers that leverage HTTP/2 connection reuse (i.e., most browsers) to produce 404 errors when accessing a second host after a connection to another host has already been established.

> https://istio.io/latest/docs/ops/common-problems/network-issues/#404-errors-occur-when-multiple-gateways-configured-with-same-tls-certificate


# 3. Create RequestAuthentication and AuthorizationPolicy 
```
$ git clone https://github.com/developer-onizuka/istioAuth0
$ cd istioAuth0
$ kubectl apply -f auth0-jwt-onprem.yaml 
```

# 4. Start Auth0 aware Pod
```
$ kubectl delete deployment nginx-onprem
$ kubectl apply -f nginx-onprem-auth0.yaml 
```

# 5. Create JWT for Employee
Refer to the URL below: <br>
> https://github.com/developer-onizuka/Istio_ingressGateway#7-1-create-jwt


# 6. Access without Bearer token
<img src="https://github.com/developer-onizuka/istioAuth0/blob/main/istioAuth0_1.png" width="480">

# 7. Access with Bearer token
You can use [Modheader](https://chrome.google.com/webstore/detail/modheader/idgpnmonknjnojddfkpgkljpfnnfcklj) so that you can set Bearer token in google-chrome.

<img src="https://github.com/developer-onizuka/Istio_ingressGateway/blob/main/ingress_auth0_3.png" width="480">

Let's access!<br>

<img src="https://github.com/developer-onizuka/istioAuth0/blob/main/istioAuth0_2.png" width="480">


# 8. How to work
> https://note.minna-no-ginko.com/n/n9e752ad2ab4c <br>
> https://programmaticponderings.com/2019/01/06/securing-kubernetes-withistio-end-user-authentication-using-json-web-tokens-jwt/


with Auth
---
1) Client sets Bearer token in Browser. 
2) Client's Browser sends Bearer token to Auth0 and asks Auth0 to provide JWT to access Server. (Authentication)
3) Auth0 sends JWT (Token to access to Server) to Client if Bearer token is as expected.
4) Client obtains JWT.
5) Client requests to Server with JWT.
6) Server asks if Auth0 authenticated the JWT.
7) Auth0 responds it has been already authenticated. <br>
   But, the JWT might have been expired in some cases. Then, the responce will be negative.
9) Server finally responds to the request Client did in #5.
10) Client finally gets the responce from Server. (Fin) <br>
   If the JWT has been expired, Client's Browser might say "JWT is expired."

```
         #1.    #2.    #4.    #5.                  #9.
 Client -+------+------+------+--------------------+------
                |      ^      |                    ^
                |      |      |                    |
                |      |      V      #6.           | #8.
 Server --------|------|------+------+------+------+------ require-auth0: enabled
                |      |             |      ^
                |      |             |      |
                V      | #3.         V      | #7.
 Auth0  --------+------+-------------+------+-------------
 (https://xxx.us.auth0.com/)
```

without Auth (require-auth0: enabled)
---
1) No Bearer token in Browser. 
2) Client asks Auth0 to provide JWT to access Server without Bearer token. (Authentication)
3) Auth0 responds "RBAC: access denied". ie, Auth0 itself already knows Clinet has not had any RBAC to access to Server.

```
         #1.    #2.    "RBAC: access denied"
 Client -+------+------+----------------------------------
                |      ^  
                |      | 
                |      | 
 Server --------|------|---------------------------------- require-auth0: enabled
                |      |
                |      |
                V      | #3.
 Auth0  --------+------+----------------------------------
 (https://xxx.us.auth0.com/)
```

without Auth (no label of require-auth0)
---
1) No Bearer token in Browser. 
2) Client requests to Server. Not necessary to access Auth0 before it.
3) Server responds to the request Client, because Server does not need to be controlled by auth0.

```
         #1.    #2.    #3.    
 Client -+------+------+----------------------------------
                |      ^     
                |      |    
                V      |     
 Server ---------------+---------------------------------- no label of require-auth0
 
               
               
 Auth0  --------------------------------------------------
 (https://xxx.us.auth0.com/)
```

