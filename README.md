## Provison a jumpbox

```
git clone https://github.com/making/terraforming-jumpbox
cd terraforming-jumpbox
cp terraform.tfvars.sample terraform.tfvars
terraform plan -out plan
terraform apply plan
./ssh-jumpbox.sh
```

## Install CLIs on jumpbox

```
chmod +x ./provision.sh 
./provision.sh
```

## Pave AWS env

```
mkdir ~/workspace
cd ~/workspace
git clone https://github.com/pivotal/paving.git
cd paving
git checkout -b custom
```

### Apply patches

```
git remote add voor https://github.com/voor/paving.git
git remote add making https://github.com/making/paving.git
git fetch voor add-kubernetes-tags
git fetch making add-s3-permissions
git fetch making force-destroy-buckets
git cherry-pick -x ef6d4965a19c489bfe3dbbb0a9c5faa167f70475
git cherry-pick -x 1ad99f8224d9e7662f2a2474ed437e907c2f4d56
git cherry-pick -x 85a8832a797ac2e15fb607d59733c6ed56188887
```

### Run terraform

```
export TF_VAR_region=ap-south-1
export TF_VAR_access_key=AKIA**********
export TF_VAR_secret_key=******
export TF_VAR_availability_zones='["ap-south-1a","ap-south-1b","ap-south-1c"]'
export TF_VAR_environment_name=dev1
export TF_VAR_hosted_zone=tanzu.example.com

cd aws
terraform init
terraform plan -out plan
terraform apply plan
```

## Download Pivnet files

```
pivnet login --api-token=****
mkdir ~/workspace/pivnet
cd ~/workspace/pivnet
pivnet download-product-files --product-slug='ops-manager' --release-version='2.9.5' --product-file-id=713247
pivnet download-product-files --product-slug='elastic-runtime' --release-version='2.9.5' --product-file-id=709121
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.7.0' --product-file-id=649104
pivnet download-product-files --product-slug='pivotal-container-service' --release-version='1.7.0' --product-file-id=646536
sudo install pks-linux-amd64-1.7.0-build.483 /usr/local/bin/pks
pivnet download-product-files --product-slug='platform-automation' --release-version='4.4.3' --product-file-id=709887
tar xvf platform-automation-image-4.4.3.tgz ./rootfs/usr/bin/p-automator
sudo install rootfs/usr/bin/p-automator /usr/local/bin/
```

## Create ops manager vm

```
mkdir -p ~/workspace/config/${TF_VAR_environment_name}/ops-manager
cd ~/workspace/config/${TF_VAR_environment_name}/ops-manager

terraform output -state=${HOME}/workspace/paving/aws/terraform.tfstate -json \
   | bosh int - --path /stable_config/value \
   | bosh int - > vars.yml

wget https://raw.githubusercontent.com/making/platform-automation/master/config/aws/sandbox/ops-manager/config.yml

p-automator create-vm \
  --config ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/config.yml \
  --image-file ${HOME}/workspace/pivnet/ops-manager-aws-2.9.5-build.144.yml \
  --state-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/state.yml \
  --vars-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml
```

## Configure authentication

```
cd ~/workspace/config/${TF_VAR_environment_name}/ops-manager

export OM_USERNAME=admin
export OM_PASSWORD=$(uuidgen)
export OM_DECRYPTION_PASSPHRASE=$(uuidgen)

cat <<EOF > auth.yml
username: ${OM_USERNAME}
password: ${OM_PASSWORD}
decryption-passphrase: ${OM_DECRYPTION_PASSPHRASE}
EOF

cat <<EOF > env.yml
target: https://opsmanager.${TF_VAR_environment_name}.${TF_VAR_hosted_zone}
connect-timeout: 30
request-timeout: 1800
skip-ssl-validation: true
username: ${OM_USERNAME}
password: ${OM_PASSWORD}
decryption-passphrase: ${OM_DECRYPTION_PASSPHRASE}
EOF

om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  --skip-ssl-validation \
  configure-authentication \
  --config ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/auth.yml
```

## Generate and configure let's encrypt

```
mkdir -p ~/workspace/letsencrypt
cd ~/workspace/letsencrypt

export AWS_REGION=${TF_VAR_region}
export AWS_ACCESS_KEY_ID=${TF_VAR_access_key}
export AWS_SECRET_ACCESS_KEY=${TF_VAR_secret_key}
export AWS_HOSTED_ZONE_ID=....... (not hosted zone name but ID)
export SUBDOMAIN=${TF_VAR_environment_name}.${TF_VAR_hosted_zone}
export EMAIL=.......

lego --accept-tos \
  --key-type=rsa4096 \
  --domains="*.${SUBDOMAIN}" \
  --domains="*.apps.${SUBDOMAIN}" \
  --domains="*.sys.${SUBDOMAIN}" \
  --domains="*.uaa.sys.${SUBDOMAIN}" \
  --domains="*.login.sys.${SUBDOMAIN}" \
  --domains="*.pks.${SUBDOMAIN}" \
  --domains="*.tkgi.${SUBDOMAIN}" \
  --email=${EMAIL} \
  --dns=route53 \
  run

cat <<EOF > ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/ssl-certificate.yml
ssl-certificate:
  certificate: ((ssl_certificate))
  private_key: ((ssl_private_key))
EOF

om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  --skip-ssl-validation \
  configure-opsman \
  --config <(bosh int ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/ssl-certificate.yml \
    --var-file ssl_certificate=${HOME}/workspace/letsencrypt/.lego/certificates/_.${SUBDOMAIN}.crt \
    --var-file ssl_private_key=${HOME}/workspace/letsencrypt/.lego/certificates/_.${SUBDOMAIN}.key)
```

## Configrue director

```
mkdir -p ~/workspace/config/${TF_VAR_environment_name}/director
cd ~/workspace/config/${TF_VAR_environment_name}/director

wget https://raw.githubusercontent.com/making/platform-automation/master/config/aws/sandbox/director/config.yml

om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  configure-director \
  --config ${HOME}/workspace/config/${TF_VAR_environment_name}/director/config.yml \
  --vars-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml
```

## Configrue TAS

```
mkdir -p ~/workspace/config/${TF_VAR_environment_name}/tas
cd ~/workspace/config/${TF_VAR_environment_name}/tas
wget https://raw.githubusercontent.com/making/platform-automation/master/config/aws/sandbox/tas/config.yml

om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  upload-product \
  --product ${HOME}/workspace/pivnet/srt-2.9.5-build.10.pivotal
om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  stage-product \
  --product-name cf \
  --product-version 2.9.5
om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml  \
  configure-product \
  --config <(bosh int ${HOME}/workspace/config/${TF_VAR_environment_name}/tas/config.yml \
    --var-file ssl_certificate=${HOME}/workspace/letsencrypt/.lego/certificates/_.${SUBDOMAIN}.crt \
    --var-file ssl_private_key=${HOME}/workspace/letsencrypt/.lego/certificates/_.${SUBDOMAIN}.key) \
  --vars-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml
```

## Configure TKGI

```
mkdir -p ~/workspace/config/${TF_VAR_environment_name}/tkgi
cd ~/workspace/config/${TF_VAR_environment_name}/tkgi
wget https://raw.githubusercontent.com/making/platform-automation/master/config/aws/sandbox/tkgi/config.yml

om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  upload-product \
  --product ${HOME}/workspace/pivnet/pivotal-container-service-1.7.0-build.26.pivotal
om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  stage-product \
  --product-name pivotal-container-service \
  --product-version 1.7.0-build.26
om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml  \
  configure-product \
  --config <(bosh int ${HOME}/workspace/config/${TF_VAR_environment_name}/tkgi/config.yml \
    --var-file ssl_certificate=${HOME}/workspace/letsencrypt/.lego/certificates/_.${SUBDOMAIN}.crt \
    --var-file ssl_private_key=${HOME}/workspace/letsencrypt/.lego/certificates/_.${SUBDOMAIN}.key) \
  --vars-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml
```

## Apply changes

```
om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  apply-changes
```

## Verify TAS

```
TAS_SYS_DOMAIN=$(bosh int ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml --path /sys_dns_domain)
TAS_ADMIN_SECRET=$(om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml credentials -p cf -c .uaa.admin_client_credentials -f password)
TAS_ADMIN_PASSWORD=$(om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml credentials -p cf -c .uaa.admin_credentials -f password)
```

### Login to CF as admin

```
cf login -a api.${TAS_SYS_DOMAIN} -u admin -p ${TAS_ADMIN_PASSWORD} -o system -s system

cf create-org demo
cf create-space demo -o demo
cf target -o demo -s demo
mkdir -p /tmp/hello
echo '<?php echo "Hello World!";' > /tmp/hello/index.php
cf push hello -m 32m -p /tmp/hello -b php_buildpack
cf logs hello --recent
cf ssh hello
```

### Loing to UAA as admin client

```
uaac target https://uaa.${TAS_SYS_DOMAIN}
uaac token client get admin -s ${TAS_ADMIN_SECRET}
```

## Verify TKGI

```
TKGI_API_DNS=$(bosh int ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml --path /pks_api_dns)
TKGI_ADMIN_SECRET=$(om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml  credentials -p pivotal-container-service -c .properties.pks_uaa_management_admin_client -f secret)
TKGI_ADMIN_PASSWORD=$(om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml  credentials -p pivotal-container-service -c .properties.uaa_admin_password -f secret)

pks login -a ${TKGI_API_DNS} -k -u admin -p ${TKGI_ADMIN_PASSWORD}

sudo wget -O /usr/local/bin/pks-aws https://raw.githubusercontent.com/making/pks-cli-aws/master/pks-aws
sudo chmod +x /usr/local/bin/pks-aws
```

### Create a cluster

```
ENV_NAME=${TF_VAR_environment_name}
CLUSTER_NAME=cluster01

export AWS_DEFAULT_REGION=${TF_VAR_region}
export AWS_ACCESS_KEY_ID=${TF_VAR_access_key}
export AWS_SECRET_ACCESS_KEY=${TF_VAR_secret_key}

MASTER_HOSTNAME=$(pks-aws create-lb ${CLUSTER_NAME} ${ENV_NAME})
pks create-cluster ${CLUSTER_NAME} -e ${MASTER_HOSTNAME} -p small -n 1 --wait
pks-aws attach-lb ${CLUSTER_NAME}
pks-aws create-tags ${CLUSTER_NAME} ${ENV_NAME}
pks get-credentials ${CLUSTER_NAME}

kubectl create deployment demo --image=making/hello-cnb --dry-run -o=yaml > /tmp/deployment.yaml
echo --- >> /tmp/deployment.yaml
kubectl create service loadbalancer demo --tcp=8080:8080 --dry-run -o=yaml >> /tmp/deployment.yaml
kubectl apply -f /tmp/deployment.yaml
```

## Delete a cluster

```
kubectl delete -f /tmp/deployment.yaml
pks-aws delete-tags ${CLUSTER_NAME} ${ENV_NAME}
pks-aws delete-lb ${CLUSTER_NAME}
pks delete-cluster ${CLUSTER_NAME} --wait --non-interactive
```

## Delete installation

```
om --env ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/env.yml \
  delete-installation \
  --force
```

## Delete ops manager vm

```
p-automator delete-vm \
  --config ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/config.yml \
  --state-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/state.yml \
  --vars-file ${HOME}/workspace/config/${TF_VAR_environment_name}/ops-manager/vars.yml
```

## Wipe AWS env

```
cd ~/workspace/paving/aws
terraform destroy -auto-approve
```