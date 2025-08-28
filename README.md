# Wallarm Node — Azure Container Instances (ACI) Deployment

> **Author:** Roman Yankin

> **Environment:** Azure Container Instances (ACI)

> **Date:** 2025-08-27

---

## Overview 

This repository documents the deployment of a Wallarm filtering node using **Azure Container Instances (ACI)**. I chose ACI because it provides a simple, yet scalable environment suitable for completion of the Wallarm Test Challenge. In addition evaluator can use the public IP (http://52.249.215.109) of the container with deployed Wallarm Node to execute tests and verify the validity of deployment. The deployment demonstrates a working Wallarm Node that registers with Wallarm Console, mounts a custom NGINX configuration from an Azure Storage Account and filters traffic proxied through NGINX.

---

## What I built (high-level)

- A single ACI container group running Wallarm Filtering node (`wallarm/node:6.4.1`).
- An **Azure File Share** (`wallarmdemo`) containing the `default.conf` NGINX configuration.


Traffic flow (summary):

```
Internet --> Wallarm Node (http://52.249.215.109) --> NGINX (proxy_pass) --> http://httpbin.org
```

High level diagram below depicts the architecture and traffic flow:
<img width="1460" height="928" alt="High Level arch and traffic diagram" src="https://github.com/user-attachments/assets/547a5c3e-a14e-4ffb-9b4b-c789adf9bdac" />


---

## Files included in this repo

- `wallarm-node.yaml` — ACI container group definition (YAML) used to deploy the container.
- `default.conf` — The NGINX configuration provided in [Wallarm Documentation](https://docs.wallarm.com/installation/cloud-platforms/azure/docker-container/#__code_9) (mounted from Azure File Share).
- `README.md` — (this file) deployment steps, issues and resolution.
- `screenshots` — various screenshots from Wallarm Web Console
- `GoTestWAF PDF report` - included for completeness 

> **Note:** Secrets and credentials are **NOT** checked into the repo.

---

## Why Azure Container Instances (ACI)?

I selected ACI for the evaluation for these reasons:

- **Fast setup & iteration:** ACI deploys containers in seconds without cluster setup.
- **Public IP capability:** Allows quick external testing (as opposed to local deployment).
- **Simple deployment:** Simple file share mounting, no hassle access to container logs for debug purposes.
- **Simplicity for the test:** The assignment required a working filtering node and logs; ACI minimised cloud-specific complexity.

---

## Deployment steps (commands)

Below is condensed version of the official Wallarm and Azure Documentation [deployment steps](https://docs.wallarm.com/installation/cloud-platforms/azure/docker-container/)

### 1) Create resource group

```bash
az group create --name WallarmDemoRcGroup --location eastus
```

### 2) Create storage account & file share (upload `default.conf`)

```shellx
# create file share
az storage share create --name wallarm-share --account-name <storageAccountName> --account-key <storageAccountKey>

# upload default.conf from current directory
az storage file upload --share-name wallarm-share --source ./default.conf --path default.conf --account-name <storageAccountName> --account-key <storageAccountKey>

# verify default.conf was properly uploaded
az storage file list --share-name wallarm-share --account-name <storageAccountName> --account-key <storageAccountKey> --output table
```

### 3) The ACI YAML (`wallarm-node.yaml`)

I used the `wallarm-node.yaml` file copied below to deploy the Wallarm Node in ACI (this is the only deviation from official documentation which suggests to use Github hosted configuration file):

```yaml
apiVersion: 2019-12-01
location: eastus
name: wallarm-node
properties:
  containers:
  - name: wallarm-node
    properties:
      image: wallarm/node:6.4.1
      resources:
        requests:
          cpu: 1.0
          memoryInGB: 1.5
      ports:
      - port: 80
      environmentVariables:
      - name: WALLARM_API_TOKEN
        secureValue: "<WALLARM_NODE_TOKEN>"
      - name: WALLARM_API_HOST
        value: "us1.api.wallarm.com"
      - name: WALLARM_LABELS
        value: "group=wallarm"
      volumeMounts:
      - name: wallarm-config-volume
        mountPath: /etc/nginx/http.d
  osType: Linux
  ipAddress:
    type: Public
    ports:
    - protocol: tcp
      port: 80
  volumes:
  - name: wallarm-config-volume
    azureFile:
      shareName: wallarm-share
      storageAccountName: <storageAccountName>
      storageAccountKey: <storageAccountKey>
  restartPolicy: Always
  imageRegistryCredentials:
  - server: index.docker.io
    username: <dockerhub-username>
    password: <dockerhub-access-token>
  type: Microsoft.ContainerInstance/containerGroups  
```


### 4) Deploy the container

```bash
az provider register --namespace Microsoft.ContainerInstance  # only once
az container create --resource-group WallarmDemoRcGroup --file wallarm-node.yaml
```

> **Note:** After container was deployed it received the following public IP `52.249.215.109`


### 5) Verify container & view logs

```bash
# show container state and public IP
az container show --resource-group WallarmDemoRcGroup --name wallarm-node --query ipAddress --output json

# tail logs
az container logs --resource-group WallarmDemoRcGroup --name wallarm-node

```

---

## Issues encountered 


### Issue 1 — Azure subscription not registered for Container Instances (registration error)

**Symptom:** `MissingSubscriptionRegistration` error when calling `az container create`.

**Cause:** The `Microsoft.ContainerInstance` resource provider was not registered on the subscription.

**Resolution:** Registered the provider via CLI or Portal:

```bash
az provider register --namespace Microsoft.ContainerInstance
```

**Result:** Subsequent `az container create` calls succeeded.


---

### Issue 2 — Image pull errors from Docker Hub (`RegistryErrorResponse`)

**Symptom:** `RegistryErrorResponse` from `index.docker.io` when ACI tried to pull the image.

**Cause:** Likely related to docker hub limits on unauthorised image pulls.

**Resolution:** Added `imageRegistryCredentials` with Docker Hub username and access token inside the ACI YAML.

**Result:** Image pulled successfully in subsequent attempts.

---

### Issue 3 — Wallarm node logs showed transient internal startup warnings

**Symptom:** Several log lines during startup:

- `CRIT Server 'unix_http_server' running without any HTTP authentication checking` (supervisord warning)
- `could not build optimal variables_hash, you should increase either variables_hash_max_size` (nginx warning)
- `wallarm: 127.0.0.1:3313 connect() failed 15` (nginx initially couldn't connect to local agent)

**Cause:** Not fully diagnosed, to be discussed. Likely related to some start-up procedures

**Resolution:** Verified the internal firewall process was running and listening.

**Key commands used to verify:**
```bash
az container exec --resource-group WallarmDemoRcGroup --name wallarm-node --exec-command "netstat -tlnp | grep 3313"
```

**Log excerpt:**

```
...
2025-08-26 23:57:03,650 CRIT Server 'unix_http_server' running without any HTTP authentication checking
2025/08/26 23:57:04 [error] 88#88: wallarm: 127.0.0.1:3313 connect() failed 15
... 
2025-08-26 23:57:05,892 INFO success: api-firewall entered RUNNING state
```


---

## Verification & testing

### 1) Confirm node registered

Confirm Wallarm Node shows up under the node page: https://us1.my.wallarm.com/nodes


### 2) Smoke test using Path Traversal attack as suggested by the Wallarm Docs

```bash
curl -v "http://52.249.215.109/etc/passwd"
```

Then check Wallarm node logs, looking specifically for the relevant GET request

```bash
az container logs --resource-group WallarmDemoRcGroup --name wallarm-node | grep 'etc/passwd'      
10.92.0.11 - - [27/Aug/2025:00:11:02 +0000] "GET /etc/passwd HTTP/1.1" 400 312 "-" "curl/8.7.1"
```

Finally finding this exact attack in Wallarm Console:
<img width="1049" height="271" alt="Path Traversal" src="https://github.com/user-attachments/assets/f0cdc0a6-eb0c-46be-a84f-4b5a205acdff" />




### 3) Final test with GoTestWAF

As required by the assignment I installed and configured GoTestWAF and generated test traffic to test Wallarm Filtering node in action

```bash
docker run --rm --network="host" -v ${PWD}/gotestwaf-reports:/app/reports wallarm/gotestwaf --url=http://52.249.215.109 --email romayankin@gmail.com 
```

Below screenshots capture Wallarm Console registering the attacks generated by GoTestWAF
<img width="1214" height="726" alt="Attacks" src="https://github.com/user-attachments/assets/f3e0ac6a-fcaa-43e7-b621-90012259131b" />
<img width="1045" height="663" alt="Threat Prevention" src="https://github.com/user-attachments/assets/5a61c375-db9d-4aae-bf28-81caf7f219e3" />
<img width="863" height="791" alt="Wallarm-filtering-node" src="https://github.com/user-attachments/assets/28104cfb-e9d3-44ce-9c52-5a2d839d5a37" />


---


## Excerpts from Relevant Logs 

### Wallarm container status details 

Below is the output produced by the `az container create` command cut-down to essential bits:

```json
{
  "name": "wallarm-node",
  "properties": {
    "instanceView": { "state": "Running" },
    "ipAddress": { "ip": "52.249.215.109", "type": "Public" },
    "volumes": [{ "azureFile": { "shareName": "wallarm-share", "storageAccountName": "wallarmdemo" }, "name": "wallarm-config-volume" }]
  }
}
```


### Wallarm Node logs (startup excerpt)

```
{"level":"info","component":"register","time":"2025-08-26T23:56:57Z","message":"node registration done"}
{"level":"info","component":"datasync","time":"2025-08-26T23:56:59Z","message":"node data synchronization done"}
```

---

## Contact / Support

If the evaluators want any additional logs, screenshots, or the final ACI YAML with placeholders filled, I can provide them in the forked repository.  

Overall it was a great test exercise which I enjoyed working on!

-Roman (romayankin@gmail.com)


---


