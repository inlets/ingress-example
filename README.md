## Ingress example with HTTPS

This repository shows you how to create various Kubernetes resources for serving a simple HTTP service over HTTPS using Let's Encrypt.

* Deployment - this creates a Pod in Kubernetes based upon a simple HTTP server image from OpenFaaS
* Service - this maps a stable IP address to any Pods created by the Deployment
* Ingress - integrates with an Ingress Controller to route traffic from the Internet to the Service, and to set up TLS termination
* Issuer - a custom CRD from cert-manger to request certificates from Let's Encrypt

## Steps

You'll first install the pre-requisites, then create the Deployment, Service, Issuer, and Ingress objects.

If you want to use traefik instead, just switch out the `ingressClassName` and skip the `arkade install ingress-nginx` step.

## Setup ingress-nginx, cert-manager, etc

```sh
curl -sLS https://get.arkade.dev | sudo sh
arkade install ingress-nginx
arkade install cert-manager
```

Or install these packages using their various README files or Helm charts.

## Create the Deployment

```sh
kubectl apply -f deployment.yml
```

## Create the Service

```sh
kubectl apply -f service.yml
```

## Create the Issuer

```sh
export DOMAIN=nodeinfo.example.com
export CLASS=nginx

cat > issuer.yml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
    acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: webmaster@${DOMAIN}
        privateKeySecretRef:
            name: letsencrypt-prod
        solvers:
        - http01:
            ingress:
                class: $CLASS
EOF

kubectl apply -f issuer.yml
```

## Create the Ingress

```sh
export DOMAIN=nodeinfo.example.com
export CLASS=nginx
export NAME=nodeinfo

cat > ingress.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: $NAME
  annotations:
    cert-manager.io/issuer: "letsencrypt-prod"
spec:
  ingressClassName: $CLASS
  rules:
  - host: "$DOMAIN"
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: $NAME
              port:
                number: 8080
  tls:
  - hosts:
    - "$DOMAIN"
    secretName: ${NAME}-tls
EOF

kubectl apply -f ingress.yml
```

## Check the progress of the certificates

```sh
kubectl get certificate
kubectl describe certificate
```

## Access the service

```sh
export DOMAIN=nodeinfo.example.com

echo "https://$DOMAIN"
```
