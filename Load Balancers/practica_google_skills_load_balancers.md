# Practica Google Skills - Load Balancers

## Cambiar zona y region
```bash
export REGION=asia-south1
export ZONE=asia-south1-a
export DEBIAN_FAMILIY=debian-12
```

```bash
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```

## Crear VM

```bash
gcloud compute instances create web3 \
    --zone=$ZONE \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "
<h3>Web Server: web3</h3>" | tee /var/www/html/index.html'
```

## Add a firewall

```bash
gcloud compute firewall-rules create www-firewall-network-lb --target-tags network-lb-tag --allow tcp:80
```

# Task 2. Configure the load balancing service (PassTrough Load Balancer)

## Create a Static IPV4

```bash
gcloud compute addresses create network-lb-ip-1 \
  --region $REGION
```

## Create a health check

```bash
gcloud compute http-health-checks create basic-check
```

## Create a target POOL

```bash
gcloud compute target-pools create www-pool \
  --region $REGION --http-health-check basic-check
```

## Add instances to pool

```bash
gcloud compute target-pools add-instances www-pool \
    --instances web1,web2,web3
```

## Create a forwarding rule

```bash
gcloud compute forwarding-rules create www-rule \
    --region $REGION \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

## Check the traffic

```bash
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region $REGION --format="json" | jq -r .IPAddress)
while true; do curl -m1 $IPADDRESS; done
```

# Task 3. Create HTTP Load Balancer

## Create a instance template

```bash
gcloud compute instance-templates create lb-backend-template \
   --region=$REGION \
   --tags=allow-health-check \
   --machine-type=e2-medium \
   --image-family=$DEBIAN_FAMILIY \
   --image-project=debian-cloud \
   --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
```

## Create a managed instances group

```bash
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=$ZONE
```

## Create firewall rule

```bash
gcloud compute firewall-rules create fw-allow-health-check \
  --network=default \
  --action=allow \
  --direction=ingress \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --target-tags=allow-health-check \
  --rules=tcp:80
```

## Crear la IP Publica

```bash
gcloud compute addresses create lb-ipv4-1 \
  --ip-version=IPV4 \
  --global
```

## Create a health check

```bash
gcloud compute health-checks create http http-basic-check
```

## Create a backend

```bash
gcloud compute backend-services create web-backend-service \
  --protocol=HTTP \
  --port-name=http \
  --health-checks=http-basic-check \
  --global
```

## Asociate backend to instance group

```bash
gcloud compute backend-services add-backend web-backend-service \
  --instance-group=lb-backend-group \
  --instance-group-zone=$ZONE \
  --global
```

## Config a url map

```bash
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
```

## Create a http proxy

```bash
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
```

## Create a forwading rule (Asociate the external ipv4 and destination proxy)

```bash
gcloud compute forwarding-rules create http-content-rule \
   --address=lb-ipv4-1\
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

## Obtener la IP

```bash
gcloud compute addresses describe lb-ipv4-1 \
  --format="get(address)" \
  --global
```
