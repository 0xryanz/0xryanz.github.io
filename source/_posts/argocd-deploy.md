---
title: argo cd部署
date: 2023-09-06 14:42:06
tags:
- k8s
- argocd
---

### 安装argo cd
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

配置ingress暴露argocd ui
```yaml
piVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" 
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
    - host: argocd.xxx.top
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
  tls:
  - hosts:
    - argocd.xxx.top
    secretName: wildcard-letsencrypt-tls

```

获取agrocd管理员密码

账号admin
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

获取context地址
```
kubectl config view -o jsonpath='{.current-context}'
```

### 测试项目运行

编写application.yaml文件,替换`repoURL`为仓库地址
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/xxx/argocd-lab.git
    targetRevision: HEAD
    path: dev
  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true

```

运行以下命令,在argocd ui界面即可查看到项目情况,argo默认每3分钟同步git仓库与本地部署状态
```
kubectl apply -f application.yaml
```

### 配置webhook立马更新
以github为例

- 打开github项目设置页
- 配置项目webhook地址
- 配置webhook secret

webhook url格式
> `https://argocd.example.com/api/webhook`, url结尾必须为/api/webhook

webhook secret
> 和下方secret配置保持一致

创建webhook secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
data:
...

stringData:
  # github webhook secret
  webhook.github.secret: shhhh! it's a GitHub secret

  # gitlab webhook secret
  webhook.gitlab.secret: shhhh! it's a GitLab secret

  # bitbucket webhook secret
  webhook.bitbucket.uuid: your-bitbucket-uuid

  # bitbucket server webhook secret
  webhook.bitbucketserver.secret: shhhh! it's a Bitbucket server secret

  # gogs server webhook secret
  webhook.gogs.secret: shhhh! it's a gogs server secret
```

运行以下命令创建secret
```
kubectl apply -f secret.yaml
```