# istioAuth0
> https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/

```
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

$ openssl req -out onprem.example.com.csr -newkey rsa:2048 -nodes -keyout onprem.example.com.key -subj "/CN=onprem.example.com/O=employee organization"
$ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in onprem.example.com.csr -out onprem.example.com.crt

$ kubectl create -n istio-system secret tls onprem-credential --key=onprem.example.com.key --cert=onprem.example.com.crt
```
