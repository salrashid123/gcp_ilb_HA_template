
# GCP Internal LoadBalancer with Regional Instance Groups for High Availability


## Introduction

Sample Google Cloud [Deployment Manager](https://cloud.google.com/deployment-manager/docs/) template to create
a high-availability VM cluster behind a single IP address.

It is basically an implementation of

- [https://cloud.google.com/compute/docs/load-balancing/internal/#high-availability](https://cloud.google.com/compute/docs/load-balancing/internal/#high-availability)
- [https://cloud.google.com/compute/docs/instance-groups/distributing-instances-with-regional-instance-groups](https://cloud.google.com/compute/docs/instance-groups/distributing-instances-with-regional-instance-groups)

as a DM Template similar to the published sample [here](https://github.com/salrashid123/deploymentmanager-samples/tree/master/examples/v2/internal_lb) with several differences and features:

- [Regional Managed Instance Group](https://cloud.google.com/compute/docs/instance-groups/distributing-instances-with-regional-instance-groups)
- Instance group running in its own private network.
- Firewall rules controlling traffic inbound to private network
- [VPC Peering with Internal LoadBalancers](https://cloud.google.com/vpc/docs/vpc-peering#internal_load_balancing)
- [Autoscaling](https://cloud.google.com/compute/docs/reference/rest/v1/regionAutoscalers)
- [Internal Load Balancer with default affinity](https://cloud.google.com/compute/docs/load-balancing/internal/)
- Post startup can remove external IP addresses for each VM

## Architecture

![ILB](/images/ILB.png)


The default in this repo creates an apache2 server on port :80 on each VM.

### Prerequsites

Enable APIs on project

```
gcloud services enable compute.googleapis.com
gcloud services enable logging.googleapis.com
gcloud services enable monitoring.googleapis.com
gcloud services enable deploymentmanager.googleapis.com
gcloud services enable runtimeconfig.googleapis.com
```


### Install ILB+Instance Group

Run:

```
$ gcloud deployment-manager deployments create apache --config cluster.yaml

NAME                                       TYPE                                                 STATE      ERRORS  INTENT
apache-addPeering-from-custom-subnet       gcp-types/compute-v1:compute.networks.addPeering     COMPLETED  []
apache-addPeering-to-custom-subnet         gcp-types/compute-v1:compute.networks.addPeering     COMPLETED  []
apache-as                                  compute.v1.regionAutoscaler                          COMPLETED  []
apache-bes                                 compute.v1.regionBackendService                      COMPLETED  []
apache-config                              runtimeconfig.v1beta1.config                         COMPLETED  []
apache-custom-network                      compute.v1.network                                   COMPLETED  []
apache-custom-subnet                       compute.v1.subnetwork                                COMPLETED  []
apache-firewall-allow-default-us-central1  compute.v1.firewall                                  COMPLETED  []
apache-firewall-allow-health-check         compute.v1.firewall                                  COMPLETED  []
apache-firewall-allow-internal-lb          compute.v1.firewall                                  COMPLETED  []
apache-firewall-allow-tcp22-icmp           compute.v1.firewall                                  COMPLETED  []
apache-fwdrule                             compute.v1.forwardingRule                            COMPLETED  []
apache-hc-tcp                              compute.v1.healthCheck                               COMPLETED  []
apache-igm                                 compute.v1.regionInstanceGroupManager                COMPLETED  []
apache-it                                  compute.v1.instanceTemplate                          COMPLETED  []
apache-removePeering-from-custom-subnet    gcp-types/compute-v1:compute.networks.removePeering  COMPLETED  []
apache-waiter                              runtimeconfig.v1beta1.waiter                         COMPLETED  []
default                                    gcp-types/compute-v1:compute.networks.get            COMPLETED  []
default-subnet-us-central1                 gcp-types/compute-v1:compute.subnetworks.get         COMPLETED  []
```

### Acquire LB IP

Now acquire the LB IP address.  You can get this from the DM's ```output``` variable or via the LB configuration on
the Cloud Console as shown below.

![ILB](/images/lb_ip.png)

In this example, the LB host:port is:
```192.168.0.8:80```

You can also see the instance groups are in its own subnet spread across the region:

![ILB](/images/ilb_vm.png)


### Verify

- Create an instance in ```us-central1-a``` (since Internal LB can't span cross-regions)

```bash
gcloud compute  instances create "instance-1" --zone "us-central1-a" --no-service-account --no-scopes --min-cpu-platform "Automatic" --image "debian-9-stretch-v20180206" --image-project "debian-cloud" --boot-disk-size "10" 
```

Then SSH to the instance and access the ILB's IP address.  You should see requests round-robin to various backends:

```bash

$ for i in {1..100}; do curl -s  http://192.168.0.8; done | sort| uniq -c |sort -nr
     34 apache-instance-drbt.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-f
     27 apache-instance-q72p.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-b
     20 apache-instance-b9q2.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-c
     19 apache-instance-zpxk.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-f

$ for i in {1..100}; do curl -s  http://192.168.0.8; done | sort| uniq -c |sort -nr
     27 apache-instance-zpxk.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-f
     27 apache-instance-b9q2.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-c
     24 apache-instance-drbt.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-f
     22 apache-instance-q72p.c.dmproject-189820.internal  projects/420215049367/zones/us-central1-b

```

## Logging and Monitoring

You are free to addin 
- Stackdriver Logging
- Stackdriver Monitoring

to the VM's start script

The following screenshots describe monitoring and logging on the cluster for the specific service (in this case apache)

* Logging
![ILB](/images/logging.png)

* Monitoring 
![ILB](/images/monitoring.png)


## Conclusion

You can use this template to create any number of high-availability systems within GCP using GCE instances, COS instances and so on.
This setup works best with stateless systems as the default template and LB strategy doesn't specifically call out affinity.

---

### CleanUP
You can simply delete the deployment to clean up almost all artifacts.  At the moment of writing (19/Feb/18), the DM
template cannot concurrently delete VPC peering setup so one of them is omitted.  TO clean the VPC peering setup, goto
[VPC Peering](https://console.cloud.google.com/networking/peering/list) and delete the residual one after deleting the deployment

