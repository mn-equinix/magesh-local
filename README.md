# <center>Cloud Run POC

## Create a Service Account

Google <a href="https://cloud.google.com/run/docs/securing/service-identity#per-service-identity">recommends</a> giving every Cloud Run service a dedicated identity by assigning it a user-managed service account instead of using the default service account.

```
gcloud iam service-accounts create cloud-run-poc \
    --description="User-managed SA for Cloud Run" \
    --display-name="Cloud_Run_POC"
```

## [Optional] Permissions

<p style="color:blue">Skip this step if you are using your user account that has Editor access and not trying to impersonate this service account to deploy the application.</p>

Setting the <a href="https://cloud.google.com/run/docs/deploying-source-code#permissions_required_to_deploy">permissions</a> required to deploy the app directly from source code.

```
gcloud projects add-iam-policy-binding arched-autumn-379510 \
    --member="serviceAccount:cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com" \
    --role="roles/cloudbuild.builds.editor"
```

`gcloud projects add-iam-policy-binding arched-autumn-379510 \
    --member="serviceAccount:cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.admin"`

`gcloud projects add-iam-policy-binding arched-autumn-379510 \
    --member="serviceAccount:cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com" \
    --role="roles/storage.admin"`

`gcloud projects add-iam-policy-binding arched-autumn-379510 \
    --member="serviceAccount:cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com" \
    --role="roles/run.admin"`

`gcloud projects add-iam-policy-binding arched-autumn-379510 \
    --member="serviceAccount:cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"`

## [Optional] Access to Impersonate the Service Account

<p style="color:blue">If you wish to impersonate the above created service account to deploy, below roles must be added to the user account.</p>

`gcloud iam service-accounts add-iam-policy-binding \
    cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com \
    --member="user:magesh.nagu@gmail.com" \
    --role="roles/iam.serviceAccountTokenCreator"`

`gcloud iam service-accounts add-iam-policy-binding \
    cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com \
    --member="user:magesh.nagu@gmail.com" \
    --role="roles/iap.tunnelResourceAccessor"`

## Copy the Source Code

Cloning the Google Cloud's public repository that contains sample source codes. We will utilize the <a href="https://github.com/GoogleCloudPlatform/python-docs-samples/tree/main/run/helloworld">Hello World</a> python program for our Cloud Run application.

`git clone https://github.com/GoogleCloudPlatform/python-docs-samples`

## Deploy the app

<p style="color:blue">Use the --impersonate-service-account flag as required.</p>

Deploying the app using our user-managed service account and <a href="https://cloud.google.com/run/docs/securing/ingress?_ga=2.117147078.-194576007.1677837727#settings">setting the ingress</a> to allow internal traffic only.

`gcloud run deploy helloworld \
    --region us-central1 \
    --source python-docs-samples/run/helloworld/ \
    --platform managed \
    --ingress internal \
    --service-account cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com \
    --impersonate-service-account=cloud-run-poc@arched-autumn-379510.iam.gserviceaccount.com`

## Attach a VPC

Serverless VPC Access connector is required only if we wish to control the <a href="https://cloud.google.com/run/docs/configuring/connecting-vpc#manage">egress setting</a> of our Cloud Run application to route requests through the VPC network. We must provide necessary <a href="https://www.googlecloudcommunity.com/gc/Infrastructure-Compute-Storage/Error-use-vpc-with-cloud-run/m-p/463760">permisisons</a> to the Cloud Run <a href="https://cloud.google.com/iam/docs/service-agents">Service Agent</a>.

## Internal Load Balancer

**export NETWORK=equinix-l2-vpc \
export REGION=us-central1 \
export RESERVED_SUBNET=chews-lb-reserved-subnet-dev \
export RESERVED_SUBNET_RANGE=10.190.0.0/26 \
export NETWORK_ENDPOINT_GROUP=chews-neg-cloudrun \
export CLOUDRUN_SERVICE=helloworld \
export BACKEND_SERVICE=chews-backend-service \
export URL_MAP=chews-load-balancer-cloudrun \
export HTTP_TARGET_PROXY=chews-http-proxy-cloudrun \
export FORWARDING_RULE=chews-lb-forwarding-rule-cloudrun \
export SUBNET=gcp-admin-connecter \
export LOAD_BALANCER_IP_ADDRESS=10.192.101.44**

We can <a href="https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-internal-serverless">set up</a> Internal HTTP(S) Load Balancer is used to <a href="https://cloud.google.com/run/docs/securing/private-networking#from-private-network">receive requests</a> from private network.

1. Create a <a href="https://cloud.google.com/load-balancing/docs/proxy-only-subnets?_gl=1*2jjkl2*_ga*MTk0NTc2MDA3LjE2Nzc4Mzc3Mjc.*_ga_WH2QY8WWF5*MTY3ODI1MTI5MS4xMC4xLjE2NzgyNTM3OTIuMC4wLjA.&_ga=2.45215076.-194576007.1677837727">proxy-only subnet</a> within our VPC network and reserve it only for this Load Balancer. The subnet must be in the same region as the Load Balancer. Subnetwork of size at least /26 is required for Internal HTTP(S) Load Balancing.

> The proxy servers implementing your regional Envoy-based load balancer need IP addresses, which will be allocated automatically from a subnet that you reserve for this purpose (and this purpose only) in region "us-central1". Each proxy will use its assigned IP address when connecting to the servers implementing your backend services.

* <a href="https://www.freecodecamp.org/news/subnet-cheat-sheet-24-subnet-mask-30-26-27-29-and-other-ip-address-cidr-network-references/">Subnet Cheat Sheet</a>

`gcloud compute networks subnets create $RESERVED_SUBNET \
    --network=$NETWORK \
    --range=$RESERVED_SUBNET_RANGE \
    --enable-flow-logs \
    --purpose=INTERNAL_HTTPS_LOAD_BALANCER \
    --region=$REGION \
    --role=ACTIVE`

2. Create a Netork Endpoint Group.

> The Cloud Run service must be in the same project and the same region as the Serverless NEG.

`gcloud compute network-endpoint-groups create $NETWORK_ENDPOINT_GROUP \
    --network=$NETWORK \
    --network-endpoint-type=serverless \
    --cloud-run-service=$CLOUDRUN_SERVICE \
    --region=$REGION`

3. Create a Backend Service

> --enable-logging cannot be specified for SSL Proxy and TCP Proxy Load Balancing. \

> --network can only be set if the load-balancing-scheme is INTERNAL.

`gcloud compute backend-services create $BACKEND_SERVICE \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --protocol=HTTP \
    --region=$REGION \
    --enable-logging`

`gcloud compute backend-services add-backend $BACKEND_SERVICE \
    --region=$REGION \
    --network-endpoint-group=$NETWORK_ENDPOINT_GROUP \
    --network-endpoint-group-region=$REGION`

4. Create Routing Rules for Incoming Requests. 

> Host rules or path matchers are not set as it only targets one backend service representing a single serverless app.

<p style="color:blue">This step creates the Load Balancer with the name given for URL map but with no frontend configured.</p>

`gcloud compute url-maps create $URL_MAP \
    --default-service=$BACKEND_SERVICE \
    --region=$REGION`

5. Create Target Proxy.

> A target HTTP proxy is referenced by one or more forwarding rules which specify the network traffic that the proxy is responsible for routing. The target HTTP proxy points to a URL map that defines the rules for routing the requests. The URL map's job is to map URLs to backend services which handle the actual requests.

`gcloud compute target-http-proxies create $HTTP_TARGET_PROXY \
    --url-map=$URL_MAP \
    --region=$REGION`

6. Create Forwarding Rule.

> A forwarding rule directs traffic that matches a destination IP address (and possibly a TCP or UDP port) to a forwarding target (load balancer, VPN gateway or VM instance). \

> You can specify a reserved static external or internal IP address with the --address=ADDRESS flag for the forwarding rule. Otherwise, if the flag is unspecified, an ephemeral IP address is automatically assigned from the subnet. \

> Do not use the proxy-only subnet for the forwarding rule IP address. You can configure any valid IP address from the (unreserved) subnet in the region.

`gcloud compute forwarding-rules create $FORWARDING_RULE \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --network=$NETWORK \
    --subnet=$SUBNET \
    --address=$LOAD_BALANCER_IP_ADDRESS \
    --target-http-proxy=$HTTP_TARGET_PROXY \
    --target-http-proxy-region=$REGION \
    --region=$REGION \
    --ports=80`


