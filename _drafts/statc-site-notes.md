# Part 1 - Create Site

Make new repo on GitHub called static-site
  Initialize with readme
git clone https://github.com/benc-uk/static-site.git

```bash
cd static-site
```

```bash
HUGO_VER="0.68.3"
wget "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VER}/hugo_${HUGO_VER}_Linux-64bit.tar.gz"
tar -xzf "hugo_${HUGO_VER}_Linux-64bit.tar.gz" hugo && rm *.gz
echo "hugo" > ".gitignore"
```

Create site
```
./hugo new site . --force
```

---

Add a theme, create example site
```bash
git submodule add https://github.com/digitalcraftsman/hugo-creative-theme themes/hugo-creative-theme
cp -r themes/hugo-creative-theme/exampleSite/* .
```

---

Test
```
./hugo serve
```
http://localhost:1313/

---

```
git add .
git commit -m "first commit"
git push
```

# Part 2 - Azure
```
LOCATION="northeurope"
RES_GRP="temp.staticsite"
STORAGE_NAME="staticbc1"
```

### deploy
```bash
az group create --name $RES_GRP --location $LOCATION -o table
az storage account create -g $RES_GRP --name $STORAGE_NAME --location $LOCATION --sku Standard_LRS --kind StorageV2 -o table
az storage blob service-properties update --account-name $STORAGE_NAME --static-website --index-document index.html
```

### upload
```bash
echo "public/**" >> ".gitignore"
./hugo -b ''
az storage blob upload-batch --source ./public --destination \$web --account-name $STORAGE_NAME
```

### get url
```
az storage account show --name $STORAGE_NAME --query "primaryEndpoints.web" -o tsv
```

# Part 3 - CI

### get key
```
az storage account keys list --account-name $STORAGE_NAME --query "[0].value" -o tsv
```
Add secret **STORAGE_KEY** with value

In GitHub add action 'Simple workflow'
Name build.yaml

add this file ...

Click save and commit

---

---
# Azure Front Door

The account being accessed does not support http.
### enable http on storage
```
az storage account update --name $STORAGE_NAME --https-only false
```

### create fd
```
FD_NAME="staticbc2"
BACKEND_HOST=$(az storage account show --name $STORAGE_NAME --query "primaryEndpoints.web" -o tsv | sed -e 's/.$//' -e 's/https:\/\///')
echo Backend host is: $BACKEND_HOST
az network front-door create -g $RES_GRP --name $FD_NAME --backend-address $BACKEND_HOST -o table
```

### get url
```
FD_HOST=$(az network front-door show -g $RES_GRP --name $FD_NAME --query "frontendEndpoints[0].hostName" -o tsv)
echo "DNS record(s) should point at: $FD_HOST"
```

# Custom domain


### create CNAME
```
DNS_RES_GRP="live.misc"
DNS_ZONE="benco.io"
CNAME_NAME="demo"
az network dns record-set cname create -g $DNS_RES_GRP --zone-name $DNS_ZONE --name $CNAME_NAME -o table
az network dns record-set cname set-record -g $DNS_RES_GRP --zone-name $DNS_ZONE --record-set-name $CNAME_NAME --cname $FD_HOST -o table
```

### add frontend
```
FD_FRONT_NAME="static-site-bencoiodemo"
az network front-door frontend-endpoint create -g $RES_GRP --front-door-name $FD_NAME --name $FD_FRONT_NAME --host-name "${CNAME_NAME}.${DNS_ZONE}" -o table
```

### point route between front and back
```
az network front-door routing-rule update -g $RES_GRP --front-door-name $FD_NAME --name DefaultRoutingRule --frontend-endpoints $FD_FRONT_NAME -o table
```

TEST!

---
# Enabling HTTPS

### enable TLS
```
az network front-door frontend-endpoint enable-https -g $RES_GRP --front-door-name $FD_NAME --name $FD_FRONT_NAME --certificate-source FrontDoor
```

### make traffic HTTPS only
```
az network front-door routing-rule update -g $RES_GRP --front-door-name $FD_NAME --name DefaultRoutingRule --accepted-protocols Https
```




# apex domain

in azure 
create A record,use alias and point a frontdoor 







### TEMP
```
az storage blob delete-batch -s \$web --account-name $STORAGE_NAME
```
```
az network front-door routing-rule update -g $RES_GRP --front-door-name $FD_NAME --name DefaultRoutingRule --accepted-protocols "Https" --forwarding-protocol "HttpsOnly"
``