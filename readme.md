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
  --namespace default \
  --create-namespace \
  --set letsencrypt.email=my.email@gmail.com \
  --set hostname=my.dynamicdnshostname.ch \
  --set hostnamepantry=my.dynamicdnshostname.ch \
  --set hostnamepantryapi=api.my.dynamicdnshostname.ch
```

Cloudflare DDNS cronjob (optional):

```
helm upgrade \
  home helm/ \
  --install \
  --namespace default \
  --create-namespace \
  --set cloudflare.enabled=true \
  --set cloudflare.secretName=cloudflare-ddns \
  --set cloudflare.secretKey=apiToken \
  --set cloudflare.apiToken=<CLOUDFLARE_API_TOKEN> \
  --set cloudflare.zoneId=<CLOUDFLARE_ZONE_ID> \
  --set cloudflare.recordId=<CLOUDFLARE_DNS_RECORD_ID> \
  --set cloudflare.recordName=my.dynamicdnshostname.ch \
  --set cloudflare.proxied=false \
  --set cloudflare.schedule='*/10 * * * *'
```
