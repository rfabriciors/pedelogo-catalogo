# Deploy do Pede logo catálogo

## Banco de dados (MongoDB)

O banco de dados é executado em um namespace próprio através de um *Statefulset*. Seu service não tem endereço ip e é acessado através de resolução interna de nomes:

`mongodb-svc.mongodb.svc.cluster.local`

É criado um volume persistente de `1Gi`

```bash
kubectl apply -f mongo/namespace-mongo.yaml
kubectl apply -f mongo/secret-mongo.yaml
kubectl apply -f mongo/statefulset-mongodb.yaml
kubectl apply -f mongo/service-mongo.yaml
```

## API (Swagger)

A API também é executada em namespace próprio e para testes, seu deploy é publicado através de um service do tipo NodePort.

Acessar via (http://\<ip do node\>:\<porta\>/swagger/index.html)

```bash
kubectl apply -f api/namespace-api.yaml
kubectl apply -f api/secret-api.yaml
kubectl apply -f api/deployment-apidb.yaml
kubectl apply -f api/service-api.yaml
```

# NGINX Ingress Controller

### Verificando o funcionamento

```bash
➜ kubectl get pods -n ingress-nginx
```

### Obtendo o IP externo do ingress para cadastrar no DNS

```bash
➜ kubectl get service -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.245.240.128   64.225.91.213   80:31100/TCP,443:30916/TCP   9m21s
ingress-nginx-controller-admission   ClusterIP      10.245.4.24      <none>          443/TCP                      9m21s
```
O endereço ip a ser cadastrado seria nesse caso `64.225.91.213`

Foi criada a entrada para testes

```text
A	*.pedelogo	64.225.91.213	600 segundos
```

# Instalando o Cert-manager

1. Criando o namespace `cert-manager`
```bash
➜ kubectl create namespace cert-manager
```

2. Adicionando o repositório `Jetstack Helm repository` ao [Helm](https://helm.sh/docs/intro/install/) e atualizando o repositório do Helm
```bash
➜ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

➜ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
Update Complete. ⎈Happy Helming!⎈
```

3. Instalando o cert-manager no name space criado no passo 1
```bash
➜ helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.2.0 --set installCRDs=true

NAME: cert-manager
LAST DEPLOYED: Tue Jan 18 16:26:07 2022
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```

4. Como indica as notas de instalação, é necessário agora criar um `Issuer` para emitir certificados TLS

<details>
<summary>production_issuer.yaml</summary>

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <nome do issuer>
spec:
  acme:
    email: your_email_address
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-private-key
    solvers:
    - http01:
        ingress:
          class: nginx
```
</details>

### Aplicando as configurações

```bash
➜ kubectl apply -f production_issuer.yaml.yaml
```

### Para listar os `issuer` e `ClusterIssuer`

```bash
➜ kubectl get issuer --all-namespaces
➜ kubectl get clusterissuer --all-namespaces
```

Com o cert-manager instalado configura-se os certificados o Ingress controller instalado anteriormente

A configuração é inserir o `annotation` e o bloco `tls` como o exemplo:

```yaml
annotations:
  kubernetes.io/ingress.class: nginx
  cert-manager.io/cluster-issuer: <nome do issuer>
spec:
  tls:
  - hosts:
    - hw1.your_domain
    - hw2.your_domain
    secretName: hello-kubernetes-tls
```

### Exemplo completo

<details>
<summary>ingress-host.yaml</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
  namespace: api
  labels:
    component: http-server
    reason: aprendizado
spec:
  tls:
  - hosts:
    - www.subdominio.dominio.io
    secretName: pedelogo-tls-cert
  rules:
    - host: www.subdominio.dominio.io
      http:
        paths:
        - backend:
            service:
              name: api
              port:
                number: 80
          path: /
          pathType: ImplementationSpecific
```
</details>

> O bloco `tls` sob `spec` define o `secret` o qual os certificados para seus sites (listados em hosts) serão armazenados. Esses certificados serão emitidos pelo ClusterIssuer do letsencrypt-prod emite. Isso deve ser diferente para cada Ingress que você cria.

### Aplicando as configurações

```bash
➜ kubectl apply -f api/ingress-host.yaml
ingress.networking.k8s.io/ingress-path created
```
#

## Problemas ocorridos na DigitalOcean

Ao fazer a requisição de certificado o mesmo fica preso na etapa "Waiting on certificate issuance from order"

```bash
kubectl describe certificaterequest/pedelogo-tls-cert-8t92g -n api
.
.
 Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  OrderCreated  50s   cert-manager  Created Order resource api/pedelogo-tls-cert-8t92g-4248390933
  Normal  OrderPending  50s   cert-manager  Waiting on certificate issuance from order namespace/pedelogo-tls-cert-8t92g-4248390933: ""
```
 e o certificate request ficar parado em `False`

 ```bash
 ➜ kubectl get certificaterequest -n api
NAME                      READY   AGE
pedelogo-tls-cert-5whwr   False   2m10s
```

Deve-se executar o passo 5 de [DigitalOcean troubleshooting](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes#step-2-%E2%80%94-setting-up-the-kubernetes-nginx-ingress-controller)

Que basicamente é:
* Criar um registro do tipo `A` no DNS para o sub-domínio ao qual o host pertence apontando para o endereço IP do ingress (já feito anteriormente)

* Obter o manifesto do serviço `ingress-nginx-controller`

* Adicionar o anotation `service.beta.kubernetes.io/do-loadbalancer-hostname: "<sub-domínio do passo 1>"

* Aplicar o manifesto

### Obtenção do manifesto

```bash
➜ kubectl get service/ingress-nginx-controller -n ingress-nginx -o yaml > ingress-nginx.yaml
```
<details>
<summary>ingress-nginx.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubernetes.digitalocean.com/load-balancer-id: beb0dc78-1f96-4dea-9c83-987537f10d16
    service.beta.kubernetes.io/do-loadbalancer-enable-proxy-protocol: "true"
    service.beta.kubernetes.io/do-loadbalancer-hostname: "subdominio.dominio.io"
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-4.0.6
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  clusterIP: 10.245.14.192
  clusterIPs:
  - 10.245.14.192
  externalTrafficPolicy: Local
  healthCheckNodePort: 30322
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    nodePort: 31454
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    nodePort: 31446
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  sessionAffinity: None
  type: LoadBalancer
```
</details>

#

## Referências

[How To Set Up an Nginx Ingress on DigitalOcean Kubernetes Using Helm
](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-on-digitalocean-kubernetes-using-helm)

[How to Set Up an Nginx Ingress with Cert-Manager on DigitalOcean Kubernetes](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes#step-2-%E2%80%94-setting-up-the-kubernetes-nginx-ingress-controller)

[Troubleshooting cert-manager](https://cert-manager.io/docs/faq/troubleshooting/)