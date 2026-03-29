## 0. Create kind config with NodePort mapping**
```
- extraPortMappings on ALL nodes (control-plane + workers)
- map containerPort 30000 → hostPort 30000
```
*This is how your file should look [config.yaml](codes/config/config.yaml)*

## 1. Create all manifests**
```
- namespace.yaml
- mongodb-pv.yaml + pvc.yaml
- mongodb-deployment.yaml + service (ClusterIP)
- server-deployment.yaml + service (NodePort: 30001)
- client-deployment.yaml + service (NodePort: 30000)
- remove tag from image in deployments (kustomize handles it)
```
*This is how your manifests should look like [manifests](codes/manifests/)*

## 2. Use NodePort for client and server
```
change your services default cluster ip to nodePort
```
*Example [services](codes/manifests/server-service.yaml)*

## 3. Install helm + kustomize → install ArgoCD via helm
*https://helm.sh/docs/intro/install/*
*https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/*

## 4. Port forward ArgoCD
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

```

## 5. Install ArgoCD CLI and login
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Verify installation
argocd version --client

argocd login <instance_public_ip>:8080 --username admin --password <initial_password> --insecure
```

## 6. Install image updater v0.18.0

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/v0.18.0/manifests/install.yaml
```

## 7. Add GitHub repo to ArgoCD (optional use only if your repo is private if public skip this step) 
```
argocd repo add https://github.com/<username>/<repo>.git \
  --username <username> \
  --password <github-token>
```

## 8. Push images to DockerHub with semver FIRST
```
docker push <docker-username>/<image-client>:0.0.0
docker push <docker-username>/<image-server>:0.0.0
```

## 9. Create kustomization.yaml
```
- list all resources
- add images with newTag
- DO NOT include secrets.yaml
- DO NOT include application.yaml
```
*Example [services](codes/manifests/kustomization.yaml)*


## 10. Create git-creds.yaml and apply manually
```
kubectl apply -f git-creds.yaml
- add to .gitignore  
- never push to GitHub
```
*Example [secrets](codes/manifests/secrets.yaml)*

## 11. Create and apply application.yaml
```
- make sure aliases match in image-list and update-strategy
  image-list: server=... client=...
  server.update-strategy: semver
  client.update-strategy: semver
```
*Example [application](codes/argocd-app/application.yaml)*


## 12. Verify everything is working
```
argocd app get todo
kubectl get pods -n todo
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f
```

## 13. Push new image and see result
```
docker push <docker-username>/image-client:0.0.1
docker push <docker-username>/image-client:0.0.1
```

---

## Things to Always Remember

| Thing | Why |
|---|---|
| `git fetch` before `git push` | image updater commits to GitHub too |
| Never edit `newTag` manually | image updater owns kustomization.yaml |
| Recreating cluster wipes everything | reapply secrets + repo creds after |
| git-creds in `.gitignore` | never expose tokens in GitHub |
| semver tag before applying app | image updater needs existing tag to compare against |