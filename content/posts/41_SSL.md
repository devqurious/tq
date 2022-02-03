---
title: "Part 41: SSL"
date: 2021-02-05T12:10:51+05:30
thumb_image: "/posts/attachments/lets-encrypt.png"
omit_header_text: true
draft: false
tags: ["letsencrypt", "ssl"]
categories: ["HomeCloud"]
---

So far, all the services in the cluster have been accessed through HTTP. This is 2021, and everything should be HTTPS, so let's fix that. But before doing so, let's understand some basic concepts here.

- Traefik is the web server, hosts a domain, and is the ingress point for all the services running in the homecloud cluster. 
- You need to get a certificate for the above domain that is hosted by Traefik. 
- `Cert-manager` can be used to obtain certificate from a CA, like Let's Encrypt (LE), using the ACME protocol. 
- The ACME protocol supports various challenge mechanisms which are used to prove ownership of a domain so that a valid certificate can be issued for that domain.
- So the only thing you need to do is somehow prove to the CA that you own the domain. Then the CA will happily give you a cert.

So far so good? 

- The first step is to install the cert-manager. This is a simple one-line command. 
- Create a secret (i.e. the private key) that will be used to get the final certificate.
- Create a ClusterIssuer object that can issue certificates for the cluster. 
- Create the Certificate object. This will trigger a chain reaction of certificaterequests -> orders -> challenges
- IF the challenges pass, then a certificate is issued by the CA (LE, in this case). If not, happy troubleshooting. 

But...what is this challenge? 

- The ACME protocol has two two type of challenges it used to ensure a ACME client (like the Cert-manager) can prove that it owns the domain. 
- The first type is the HTTP01 challenge. LE will try to use the HTTP protocol to place a file in your domain. It if is able to it will pass the challenge.
- But this type of challenge is of no use in this case - our services are running in a private cluster (behind 172.16.16.16) so there is no way the LE server can access our domain over HTTP.
- The second type is the DNS01 challenge. Here it will attempt to create a TXT record in your domain by contacting the public nameservers that host your domain. This is the challenge that will work in our case. 

There are some other things you need to do outside the cluster. 

- You need a domain name. You can get a free one for 12 months at freenom. 
- Point the nameservers of your domain to Cloudflare. Yes, Cloudflare is the easiest to get started with for this tutorial. I could not get GoDaddy to work. 
- You do need to a secret API key to be able to interact with your domain. You get this from the cloudflare dashboard.
- Create your A and CNAME records. For example, 172.16.16.16 is the IP address for yourdomain.in

## Install Cert-manager

```
sudo kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
```

## Create the secret

Create a secret that will be used by the Cluster Issuer (which we will create in the next step). This secret will allow the ClusterIssuer to pass on the the secret token needed by the LetsEncrypt server to create a TXT record in your DNS zone, in order to prove that you own the doain. 

cf_secret.yml
```
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
type: Opaque
stringData:
        api-token: somethingthatisabigsecrethere
```

```
sudo kubectl apply -f cf_secret.yml -n cert-manager
secret/cloudflare-api-token created
```

## Create the cluster

Create the ClusterIssuer. It's manifest file includes the secret created in the step above. The ClusterIssuer applies to the entire cluster, so you do NOT (and should NOT) specify a namespace.

letsencrypt.yml 
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    email: youremail@somewhere
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          cloudflare:
            email: youremail@somewhere
            apiTokenSecretRef: 
              name: cloudflare-api-token
              key: api-token
        selector: {}
```

```
sudo kubectl apply -f letsencrypt.yml 
clusterissuer.cert-manager.io/letsencrypt-stg configured
```

## Create the certificate

Create a certificate. This will trigger a chain reaction of other objects that will be created - certificaterequests, orders, and challenges. Important: Create this certificate in the namespace where you intend to use it. We will use it for the pihole web application which is in the pihole namespace. So the certificate also should be created in the pihole namespace.

yourdomain_cert.yml
```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: yourdomain-in
  namespace: pihole
spec:
  secretName: yourdomain-in-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - yourdomain.in
    - www.yourdomain.in
```

```
sudo kubectl apply -f yourdomain_cert.yml -n pihole
certificate.cert-manager.io/yourdomain-in created
```

## Verify

If all goes well, you have your certificate now. 

```
sudo kubectl get certificates -A -o wide
NAMESPACE   NAME           READY   SECRET             ISSUER            STATUS                                          AGE
pihole      yourdomain-in   True    yourdomain-in-tls   letsencrypt-stg   Certificate is up to date and has not expired   80s
```

## Attach the certificate to the domain

Use this certificate at your ingress point. We have an ingress point defined for the pihole application. 

ph_web.yml
```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pihole-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.frontend.rule.type: PathPrefixStrip
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - secretName: yourdomain-in-tls
  rules:
  - http:
      paths:
      - path: /pihole
        pathType: Prefix
        backend:
          service:
            name: pihole-web
            port:
              number: 80
```

```
sudo kubectl apply -f ph_web.yml -n pihole
ingress.networking.k8s.io/pihole-ingress created
```

## Open the browser and check

![](/images/pi/le_prod.png)
![](/images/pi/le_prod_details.png)


## Troubleshooting

### Where does the error lie?
```
sudo kubectl get certificaterequests -A -o wide
sudo kubectl get orders -A -o wide
sudo kubectl get challenges -A -o wide
```
Often times, it's the last command will display an error of some sort. For example: 

```
sudo kubectl get certificate -A -o wide
NAMESPACE   NAME         READY   SECRET       ISSUER            STATUS                                         AGE
pihole      pihole-tls   False   pihole-tls   letsencrypt-stg   Issuing certificate as Secret does not exist   5m46s


sudo kubectl get certificaterequests -A -o wide
NAMESPACE   NAME               READY   ISSUER            STATUS                                                                                    AGE
pihole      pihole-tls-8jmv2   False   letsencrypt-stg   Waiting on certificate issuance from order pihole/pihole-tls-8jmv2-102108207: "pending"   5m23s

sudo kubectl get orders -A -o wide
NAMESPACE   NAME                         STATE     ISSUER            REASON   AGE
pihole      pihole-tls-8jmv2-102108207   pending   letsencrypt-stg            7m7s

sudo kubectl get challenges -A -o wide
NAMESPACE   NAME                                    STATE     DOMAIN         REASON                                                                                                                                                                                                                                                                                                                                        AGE
pihole      pihole-tls-8jmv2-102108207-3389462006   pending   yourdomain.in   Waiting for HTTP-01 challenge propagation: failed to perform self check GET request 'http://yourdomain.in/.well-known/acme-challenge/9Lal7EdekeWH8ZE-IdGHG9bZKPLQBVvKLGQIcgZLam8': Get "http://yourdomain.in/.well-known/acme-challenge/9Lal7EdekeWH8ZE-IdGHG9bZKPLQBVvKLGQIcgZLam8": dial tcp 172.16.16.16:80: connect: connection timed out   7m31s

```

### Useful commands

List all resources:

```
kubectl get Issuers,ClusterIssuers,Certificates,CertificateRequests,Orders,Challenges --all-namespaces
```

[Delete all](https://serverfault.com/questions/1045741/deleting-all-instances-of-resource-type-across-multiple-all-kubernetes-namespace) and restart:

```
sudo kubectl delete crd certificates.cert-manager.io
```

Describe an order (for finding a faulty one)

```
sudo kubectl describe orders -A
```

### References:

- https://sysadmins.co.za/https-using-letsencrypt-and-traefik-with-k3s/
- https://itnext.io/automated-tls-with-cert-manager-and-letsencrypt-for-kubernetes-7daaa5e0cae4
- https://cert-manager.io/docs/installation/kubernetes/
- https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation
- https://stackoverflow.com/questions/63346728/issuing-certificate-as-secret-does-not-exist#:~:text=So%20one%20of%20the%20common,and%20redirects%20to%20443%20directly.
- https://vocon-it.com/2019/01/08/kubernetes-automatic-tls-certificates-with-lets-encrypt/ 
- https://medium.com/flant-com/cert-manager-lets-encrypt-ssl-certs-for-kubernetes-7642e463bbce

