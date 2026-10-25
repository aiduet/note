+++
title = "A new way to deploy the blog"
date = "2023-02-12"
slug = "a-new-way-to-deploy-blog"
tags = ["notes"]
+++

I moved my code repo in some free time.

I used to keep private code on Coding, for three reasons:

1. I knew it from work.
2. Push and pull in China was fast.
3. I had private image repos and a few hundred free minutes for deploy.

Then it became paid, and deploy to servers abroad was slow. I tried my own nodes. They were still slow, likely because those servers were weak.

Now I run the servers with k3s. Image deploy is simpler:

Local → GitHub → Docker Hub → reset the pod → k3s pulls

## Build and push the image

`.github/workflows/build-and-deploy.yml`:

```yaml
name: Build and Deploy

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.110.0'

      - name: Remove old public folder
        run: rm -rf public

      - name: Build website
        run: hugo --minify

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }} # create a on docker

      - name: Build image and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: avacooper/blog:latest # your img path

      - name: Cleanup
        run: |
          docker system prune -af --volumes
          rm -rf ./public
```

Hugo writes static files into `public/`. nginx can serve that folder as-is.

`Dockerfile`:

```dockerfile
FROM nginx:alpine

COPY public/ /usr/share/nginx/html/

EXPOSE 80
```

## k3s

I already had cert-manager, so I did not use LoadBalancer. I used nginx-ingress.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: org-domain-blog
  labels:
    app: blog
spec:
  ports:
    - port: 80
      name: blog
      targetPort: 80
  selector:
    app: blog
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
        - name: blog
          image: avacooper/blog:latest
          ports:
            - containerPort: 80
          imagePullPolicy: Always
    #   nodeSelector:
    #     nn: sp
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: org-domain-blog-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
spec:
  ingressClassName: nginx
  tls:
  - hosts: 
    - "*.domain.org"
    - "blog.domain.org"
    secretName: tls
  rules:
  - host: blog.domain.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: org-domain-blog
            port:
              number: 80
```

The image on Docker Hub is public. I may buy a larger server later so I can keep a private image.
