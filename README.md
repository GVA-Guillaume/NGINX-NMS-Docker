# NGINX Management Suite - Docker image builder

## Description

This repo creates a docker image for:

- [NGINX Instance Manager](https://docs.nginx.com/nginx-instance-manager/) 2.4.0+
- [NGINX Management Suite API Connectivity Manager](https://docs.nginx.com/nginx-management-suite/acm/) 1.0.0+
- [Security Monitoring](https://docs.nginx.com/nginx-management-suite/admin-guides/installation/install-guide/#install-nms-modules) 1.0.0+

The image can optionally be built with Second Sight support (see https://github.com/F5Networks/SecondSight)

## Tested releases

This repository has been tested with:

- NGINX Instance Manager 2.4.0, 2.5.0, 2.5.1, 2.6.0
- NGINX Management Suite API Connectivity Manager 1.0.0, 1.1.0, 1.1.1, 1.2.0
- Security Monitoring 1.0.0

## Prerequisites

- Docker 20.10+ to build the image
- Private registry to push the target Docker image
- Kubernetes/Openshift cluster with dynamic storage provisioner enabled: see the [example](/contrib/pvc-provisioner)
- NGINX Ingress Controller with `VirtualServer` CRD support (see https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/)
- Access to F5/NGINX downloads to fetch NGINX Instance Manager 2.4.0+ installation .deb file and API Connectivity Manager 1.0+ installation .deb file
- Linux host running Docker to build the image

## How to build

1. Clone this repo
2. Download NGINX Instance Manager 2.4.0+ .deb installation file for Ubuntu 22.04 "jammy_amd64" (ie. `nms-api-connectivity-manager_1.2.0.668430332~jammy_amd64.deb`) and copy it into `nim-files/`
3. Optional: download API Connectivity Manager 1.0+ .deb installation file for Ubuntu 22.04 "jammy_amd64" (ie. `nms-api-connectivity-manager_1.2.0.668430332~jammy_amd64.deb`) and copy it into `nim-files/`
4. Optional: download Security Monitoring .deb installation file for Ubuntu 22.04 "jammy_amd64" (ie. `nms-sm_1.0.0-697204659~jammy_amd64.deb`) and copy it into `nim-files/`
5. Build NGINX Instance Manager Docker image using:

```
$ ./scripts/buildNIM.sh 
NGINX Management Suite Docker image builder

 This tool builds a Docker image to run NGINX Management Suite

 === Usage:

 ./scripts/buildNIM.sh [options]

 === Options:

 -h                     - This help
 -n [filename]          - The NGINX Instance Manager .deb package filename
 -a [filename]          - The API Connectivity Manager .deb package filename - optional
 -w [filename]          - The Security Monitoring .deb package filename - optional
 -t [target image]      - The Docker image name to be created
 -s                     - Enable Second Sight (https://github.com/F5Networks/SecondSight/) - optional

 === Examples:

 ./scripts/buildNIM.sh -n nim-files/nms-instance-manager_2.6.0-698150575~jammy_amd64.deb \
        -a nim-files/nms-api-connectivity-manager_1.2.0.668430332~jammy_amd64.deb \
        -w nim-files/nms-sm_1.0.0-697204659~jammy_amd64.deb \
        -t my.registry.tld/nginx-nms:2.6.0
```

6. Edit `manifests/1.nginx-nim.yaml` and specify the correct image by modifying the "image" line and configure NGINX Instance Manager username, password and the base64-encoded license file for automated license activation. In order to use API Connectivity Manager an ACM license is required

```
image: your.registry.tld/nginx-nim2:tag
[...]
env:
  ### NGINX Instance Manager environment
  - name: NIM_USERNAME
    value: admin
  - name: NIM_PASSWORD
    value: nimadmin
  - name: NIM_LICENSE
    value: "<BASE64_ENCODED_LICENSE_FILE>"
```

To base64-encode the license file the following command can be used:

```
base64 -w0 NIM_LICENSE_FILENAME.lic
```

Additionally, parameters user by NGINX Instance Manager to connect to ClickHouse can be configured:

```
env:
  [...]
  - name: NIM_CLICKHOUSE_ADDRESS
    value: clickhouse
  - name: NIM_CLICKHOUSE_PORT
    value: "9000"
  ### If username is not set to "default", the clickhouse-users ConfigMap in 0.clickhouse.yaml shall be updated accordingly
  - name: NIM_CLICKHOUSE_USERNAME
    value: "default"
  ### If password is not set to "NGINXr0cks", the clickhouse-users ConfigMap in 0.clickhouse.yaml shall be updated accordingly
  - name: NIM_CLICKHOUSE_PASSWORD
    value: "NGINXr0cks"
```

7. If Second Sight was built in the image, configure the relevant environment variables. See the documentation at https://github.com/F5Networks/SecondSight/#on-kubernetesopenshift

```
env:
  ### Second Sight Push mode
  - name: STATS_PUSH_ENABLE
    #value: "true"
    value: "false"
  - name: STATS_PUSH_MODE
    value: CUSTOM
    #value: PUSHGATEWAY
  - name: STATS_PUSH_URL
    value: "http://192.168.1.5/callHome"
    #value: "http://pushgateway.nginx.ff.lan"
  ### Push interval in seconds
  - name: STATS_PUSH_INTERVAL
    value: "10"
```

8. Check / modify files in `/manifests/certs` to customize the TLS certificate and key used for TLS offload

9. Start and stop using

```
./scripts/nimDockerStart.sh start
./scripts/nimDockerStart.sh stop
```

10. After starting NGINX Instance Manager it will be accessible from outside the cluster at:

NGINX Instance Manager GUI: `https://nim2.f5.ff.lan`
NGINX Instance Manager gRPC port: `nim2.f5.ff.lan:30443`

and from inside the cluster at:

NGINX Instance Manager GUI: `https://nginx-nim2.nginx-nim2`
NGINX Instance Manager gRPC port: `nginx-nim2.nginx-nim2:443`


Second Sight REST API (if enabled at build time - see the documentation at `https://github.com/F5Networks/SecondSight`):
- `https://nim2.f5.ff.lan/f5tt/instances`
- `https://nim2.f5.ff.lan/f5tt/metrics`
- Push mode (configured through env variables in `manifests/1.nginx-nim.yaml`)

Grafana dashboard: `https://grafana.nim2.f5.ff.lan` - see [configuration details](/contrib/grafana)

Running pods are:

```
$ kubectl get pods -n nginx-nim2 -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
clickhouse-7bc96d6d56-jthtf   1/1     Running   0          5m8s   10.244.1.65   f5-node1   <none>           <none>
grafana-6f58d455c7-8lk64      1/1     Running   0          5m8s   10.244.2.80   f5-node2   <none>           <none>
nginx-nim2-679987c54d-7rl6b   1/1     Running   0          5m8s   10.244.1.64   f5-node1   <none>           <none>
```

11. For NGINX Instances running on VM/bare metal only: after installing the nginx-agent on NGINX Instances to be managed with NGINX Instance Manager 2, update the file `/etc/nginx-agent/nginx-agent.conf` and modify the line:

```
grpcPort: 443
```

into:

```
grpcPort: 30443
```

and then restart nginx-agent


## Additional features

- [Grafana dashboard for telemetry](/contrib/grafana)


# Starting NGINX Management Suite

```
$ ./scripts/nimDockerStart.sh start
namespace/nginx-nim2 created
Generating a RSA private key
...................+++++
...............................+++++
writing new private key to 'nim2.f5.ff.lan.key'
-----
secret/nim2.f5.ff.lan created
deployment.apps/nginx-nim2 created
service/nginx-nim2 created
service/nginx-nim2-grpc created 
virtualserver.k8s.nginx.org/vs-nim2 created

$ kubectl get pods -n nginx-nim2 -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
clickhouse-7bc96d6d56-jthtf   1/1     Running   0          5m8s   10.244.1.65   f5-node1   <none>           <none>
grafana-6f58d455c7-8lk64      1/1     Running   0          5m8s   10.244.2.80   f5-node2   <none>           <none>
nginx-nim2-679987c54d-7rl6b   1/1     Running   0          5m8s   10.244.1.64   f5-node1   <none>           <none>
```

NGINX Management Suite GUI is now reachable from outside the cluster at:
- Web GUI: `https://nim2.f5.ff.lan`
- gRPC: `nim2.f5.ff.lan:30443`
- Second Sight: see [usage](https://github.com/F5Networks/SecondSight/blob/main/USAGE.md)

# Stopping NGINX Management Suite

```
$ ./scripts/nimDockerStart.sh stop
namespace "nginx-nim2" deleted
```
