# TLS_Certificate
In this repository I am going to explain what I have done to get TLS Certificate for my domain. 

## Check if Traefik is accepting external traffic 

First of all we should make sure that our domain resolves to the public IP of cluster and request to that reaches Traefik.

```
dig nematdoust.osdl.ir

; <<>> DiG 9.18.39-0ubuntu0.22.04.6-Ubuntu <<>> nematdoust.osdl.ir
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53629
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;nematdoust.osdl.ir.		IN	A

;; ANSWER SECTION:
nematdoust.osdl.ir.	300	IN	A	193.176.242.20

;; Query time: 87 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Thu Sep 10 20:08:04 +0330 2026
;; MSG SIZE  rcvd: 63

```
This proves that domain resolves to IP of master node of cluster. and :
```
k get svc -n kube-system
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP                  PORT(S)                      AGE
kube-dns         ClusterIP      10.43.0.10     <none>                       53/UDP,53/TCP,9153/TCP       32d
metrics-server   ClusterIP      10.43.12.19    <none>                       443/TCP                      32d
traefik          LoadBalancer   10.43.186.40   193.176.242.20,37.32.15.33   80:30532/TCP,443:32414/TCP   32d
```

Proves that requests to that IP will reach Traefik LoadBalancer service. So when Let's encrypt executes the challenge on `http://nematdoust.osdl.ir/.well-known/acme-challenge/...` the request will reach traefik ( because its listening on port 80 ) 

## Configure helm values

The first step was to configure Traefik to use Let's Encrypt through its built-in ACME support.

First the currently configured Helm values were exported:

```
helm get values traefik -n kube-system -o yaml > traefik-values.yaml
```

The original values were extended with the Let's Encrypt configuration.

```
additionalArguments:
  - "--certificatesresolvers.letsencrypt.acme.email=alineamatdoost919@gmail.com"
  - "--certificatesresolvers.letsencrypt.acme.storage=/data/acme.json"
  - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"

persistence:
  enabled: true
  path: /data
  size: 10Mi
```

### Explaining Important parts

1. Creates an ACME certificate resolver named letsencrypt
2. Tells Traefik to store ACME information in `/data/acme.json`, including certificate/account data.
3. Configures Traefik to use the HTTP-01 challenge through port 80 ( web entrypoint ).
   
Let's Encrypt validates domain ownership by requesting a special HTTP challenge URL.

Traefik handles this challenge automatically.

4. The ACME data must survive Traefik pod restarts, so /data was made persistent. So certificate, private key and ... will be stored in /data/acme.json on Traefik container's file system which is mounted by PVC and will survive if pod restarts.


After that, the Traefik release could be upgraded using the new values:

```
helm upgrade traefik traefik/traefik -n kube-system -f traefik-values.yaml
```

First time executing this command got this error :
```
Error: repo traefik not found
```

This means traefik helm repo is not available. For that I checked the list of helm repositories and the result proved that traefik helm repo is not available:
```
helm repo list
NAME	URL                                           
vm  	https://victoriametrics.github.io/helm-charts/
```

For that reason first added helm repo :
```
helm repo add traefik https://traefik.github.io/charts
helm repo update
"traefik" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "traefik" chart repository
...Successfully got an update from the "vm" chart repository
Update Complete. ⎈Happy Helming!⎈
```

and after that executing upgrade command was successful. ( To make sure I got a `k get pods -n kube-system` and age of traefik pod proved its restart with new helm values. 

**Note:** 

Just changing helm values of traefik would not trigger it to fetch certificate, an Ingress must tell Traefik that it wants TLS and which certificate resolver to use.

## Configuring Ingress Rule of Frontend Service

So for triggering Traefik to get TLS certificate, I changed Frontend Ingress rule to this :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: front-ingress
  namespace: application
  annotations:
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: "letsencrypt"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
      - nematdoust.osdl.ir
  rules:
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: react-service
                port:
                  number: 80
```

### Explanation

```
traefik.ingress.kubernetes.io/router.tls: "true"
```

- Enables TLS for this Traefik router.

```
traefik.ingress.kubernetes.io/router.tls.certresolver: "letsencrypt"
```

Tells traefik to use the letsencrypt ACME resolver to get the certificate for this hostname.

```
tls:
  - hosts:
      - nematdoust.osdl.ir
```

Declares the hostname that should use TLS certificate.

After these changes applied the ingress rule : `k apply -f manifest.yaml`

## Traefik automatically started the ACME process

After these changes, I checked the log of Traefik pod and it proved that traefik has already started the process of getting TLS certificate:
```
2026-09-10T13:20:34Z INF Register... providerName=letsencrypt.acme
2026-09-10T13:20:34Z INF Registering the account. email=alineamatdoost919@gmail.com lib=lego
2026-09-10T13:20:35Z INF Obtaining bundled SAN certificate. domains=nematdoust.osdl.ir lib=lego
2026-09-10T13:20:35Z INF Use solver. domain=nematdoust.osdl.ir lib=lego type=http-01
2026-09-10T13:20:35Z INF http01: Trying to solve HTTP-01. domain=nematdoust.osdl.ir lib=lego
2026-09-10T13:20:40Z INF The server validated our request. domain=nematdoust.osdl.ir lib=lego
2026-09-10T13:20:40Z INF Validations succeeded; requesting certificates. domains=nematdoust.osdl.ir lib=lego
2026-09-10T13:20:43Z INF Server responded with a certificate. domains=nematdoust.osdl.ir lib=lego
```

## Some changes needed in Backend and Frontend services

Base URL is changed to `https://nematdoust.osdl.ir` in Frontend. 

I have already added a new config to django settings:
```
CORS_ALLOWED_ORIGINS = [
    "http://localhost:5173",
    "https://nematdoust.osdl.ir"
]
```

and just changed it to `https` instead of `http`
