On dev laptop:

```
helm repo add jetstack https://charts.jetstack.io
```

```
helm repo update
```

```
helm upgrade \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --install \
  --version v1.13.3 \
  --set installCRDs=true
```

```
helm upgrade \
  home helm/ \
  --install \
  --set hostname=my.dynamicdnshostname.ch \
  --set letsencrypt.email=my.email@gmail.com
```
