# kubernetes-aws-kops.terraform

 Project to test deployemnt of Kubernetes in AWS VPC using Kops and Terraform


 ## tldr

First step is to create base infrastructure for k8s cluster - vpc, subnets and routing components

```bash
 terraform apply -var name=yourdomain.com

```

```bash
export NAME=$(terraform output cluster_name)
export KOPS_STATE_STORE=$(terraform output state_store)
export ZONES=$(terraform output -json availability_zones | jq -r '.value|join(",")')

kops create cluster \
    --master-zones $ZONES \
    --zones $ZONES \
    --topology private \
    --dns-zone $(terraform output public_zone_id) \
    --networking calico \
    --vpc $(terraform output vpc_id) \
    --target=terraform \
    --out=. \
    ${NAME}
```

Take vpc and subnets description output from TF and pass it to `gensubnets` module which runs in Docker container

```bash
terraform output -json | docker run --rm -i OlegGorJ/gensubnets:0.1 | pbcopy

kops edit cluster ${NAME}

```

```bash
# replace *subnets* section with your paste buffer (be careful to indent properly)
# save and quit editor

kops update cluster \
  --out=. \
  --target=terraform \
  ${NAME}

terraform apply -var name=yourdomain.com
```

---

## using a subdomain

If you want all of your dns records to live under a subdomain in its own hosted zone, you need to setup route delegation to the new zone. After running
`terraform apply -var name=k8s.yourdomain.com` , you can run the following commands to setup the delegation:

```bash
cat update-zone.json \
 | jq ".Changes[].ResourceRecordSet.Name=\"$(terraform output name).\"" \
 | jq ".Changes[].ResourceRecordSet.ResourceRecords=$(terraform output -json name_servers | jq '.value|[{"Value": .[]}]')" \
 > update-zone.json

aws --profile=default route53 change-resource-record-sets \
 --hosted-zone-id $(aws --profile=default route53 list-hosted-zones | jq -r '.HostedZones[] | select(.Name=="yourdomain.com.") | .Id' | sed 's/\/hostedzone\///') \
 --change-batch file://update-zone.json
```

Wait until your changes propagate before continuing. You are good to go when command

```bash
host -a k8s.yourdomain.com

```

returns the correct NS records.



---
