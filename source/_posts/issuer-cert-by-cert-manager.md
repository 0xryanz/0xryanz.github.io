---
title: cert-manager签发证书
date: 2023-09-06 14:40:45
tags:
- k8s
---

### 安装cert-manager
cert-manager 将证书和证书颁发者作为资源类型添加到 Kubernetes 集群中，并简化了获取、更新和使用这些证书的过程

它可以从各种受支持的来源颁发证书，包括 Let’s Encrypt、HashiCorp Vault 和 Venafi 以及私有 PKI

它将确保证书有效且是最新的，并在到期前的配置时间尝试更新证书

#### 添加 cert-manager 仓库
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

生成 values.yaml
```
helm show values jetstack/cert-manager > values.yaml
```

修改 values.yaml
```yaml
installCRDs: true

prometheus:
  enabled: false

webhook:
  timeoutSeconds: 10
```

如果想查看生成的清单，可以使用
```
helm template cert-manager jetstack/cert-manager -n cert-manager -f values.yaml > cert-manager.yaml
```

安装 cert-manager
```
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace -f values.yaml
```

等待
```
kubectl wait --for=condition=Ready pods --all -n cert-manager

# pod/cert-manager-55d85487cb-bmn5k condition met
# pod/cert-manager-cainjector-58448cbc99-wwrbg condition met
# pod/cert-manager-webhook-89f77959f-gmxvs condition met
```

### 创建 Issuer
Let’s Encrypt 利用 ACME (Automated Certificate Management Environment) 协议校验域名的归属，校验成功后可以自动颁发免费证书。免费证书有效期只有 90 天，需在到期前再校验一次实现续期。使用 cert-manager 可以自动续期，即实现永久使用免费证书。校验域名归属的两种方式分别是 HTTP-01 和 DNS-01，校验原理详情可参见 Let's Encrypt 的运作方式

DNS-01 校验支持泛域名， 但是是不同 DNS 提供商的配置方式不同，DNS 提供商过多而 cert-manager 的 Issuer 不能全部支持。部分可以通过部署实现 cert-manager 的 Webhook 服务来扩展 Issuer 进行支持。例如阿里 DNS 就是通过 Webhook 的方式进行支持

以cloudflare为例
1. 添加 DNS 记录
2. 前往个人资料 -> API 令牌， 使用编辑区域 DNS 模版， 创建一个 token

测试令牌是否能正常工作：
```sh
curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
     -H "Authorization: Bearer <yout api token>" \
     -H "Content-Type:application/json"
```

将得到的 token，保存到 .env 文件中
```
# .env
# api-token=<token>
api-token=wq
```

使用 Kustomize 来管理该 token

编写 kustomization.yaml
```yaml
# kustomization.yaml
resources:
  - letsencrypt-issuer.yaml
namespace: cert-manager
secretGenerator:
  - name: cloudflare-api-token-secret
    envs:
      - .env # token 就存放在这里
generatorOptions:
  disableNameSuffixHash: true
```
我们使用 Kustomize 来生成一个名为 cloudflare-api-token-secret 的 Secret, 该 Secret 被下面的清单使用

```yaml
# letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01
spec:
  acme:
    privateKeySecretRef:
      name: letsencrypt-dns01
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
      - dns01:
          cloudflare:
            email: xx@163.com # 替换成你的 cloudflare 邮箱账号
            apiTokenSecretRef:
              key: api-token
              name: cloudflare-api-token-secret # 引用保存 cloudflare 认证信息的 Secret
```
所生成的 Secret 可以使用下面的命令来检查
```
kubectl kustomize ./
```

如无问题，运行如下命令，创建 issuer
```
kubectl apply -k ./
```

查看 Let't Encrypt 注册状态
```
kubectl describe clusterissuer letsencrypt-dns01
```

输出Status:True 表示注册成功

### 创建 Certificate

```
kubectl apply -f certificate.yaml
```

```yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-todoit-tech
  namespace: default
spec:
  dnsNames:
    - "*.xxx.tech" # 要签发证书的域名，替换成你自己的
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-dns01 # 引用 ClusterIssuer，名字和 letsencrypt-issuer.yaml 中保持一致
  secretName: wildcard-letsencrypt-tls # 最终签发出来的证书会保存在这个 Secret 里面
```

查看证书是否签发成功
```
kubectl get certificate -w

# NAME                   READY   SECRET                     AGE
# wildcard-todoit-tech   False   wildcard-letsencrypt-tls   8s
# wildcard-todoit-tech   False   wildcard-letsencrypt-tls   94s
# wildcard-todoit-tech   False   wildcard-letsencrypt-tls   94s
# wildcard-todoit-tech   True    wildcard-letsencrypt-tls   94s
```

注意事项：Let's Encrypt 一个星期内只为同一个域名颁发 5 次证书，xxx.tech 和 whoami.xxx.tech 被视为不同的域名

### 测试
```
kubectl apply -f whoami.yaml
```

```yaml
# cert/whoami.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  labels:
    app: containous
    name: whoami
spec:
  replicas: 2
  selector:
    matchLabels:
      app: containous
      task: whoami
  template:
    metadata:
      labels:
        app: containous
        task: whoami
    spec:
      containers:
        - name: containouswhoami
          image: containous/whoami
          resources:
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  ports:
    - name: http
      port: 80
  selector:
    app: containous
    task: whoami
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
        - "whoami.xxx.tech"
      secretName: wildcard-letsencrypt-tls
  rules:
    - host: whoami.xxx.tech
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

```
kubectl get ingress
```