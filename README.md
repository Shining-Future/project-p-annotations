# Project P annotations

## Create dataset locally

Assume data in a directory `/path/to/project-p/data`:

```
.
├── 2020-05-24
├── 2021-04-13
├── 2022-05-04
├── 2023-04-09
├── 2023-06-30
├── 2023-07-26
├── 2023-07-26-mini-3-pro
├── 2023-07-27-panorama
├── 2023-07-27-phantom-4-pro
```

---

## Deploy CVAT in a cloud

Any cloud provider is ok, but *Project P* starts with [*VK Cloud*](https://mcs.mail.ru/app/en).

### Create a virtual machine

Go to `Cloud computing` -> `Virtual machines` -> `+ Add`, perform step-by-step VM creation, save SSH key (`/path/to/key`), then log into the machine:

```shell
ssh -i /path/to/key ubuntu@new-ip-address
```

Install Docker:

```shell
sudo apt install docker-ce
```

### Set up DNS

It's preferably to set up DNS for the `new-ip-address` in order to connect like a god: `ssh -i /path/to/key ubuntu@domain-name` and use HTTPS (otherwise one needs to tunnel via SSH).

### Set up CVAT

First, get it:

```shell
git clone https://github.com/opencv/cvat.git
cd cvat
TAG=$(git tag -l | sort | tail -1)
git checkout "$TAG"
```

Then run:

```shell
CVAT_HOST=domain-name docker compose -f docker-compose.yml -f components/serverless/docker-compose.serverless.yml up -d
```

> Configure HTTPS later

---

## Generate CVAT manifest

In order to attach data from a [cloud storage](https://opencv.github.io/cvat/docs/manual/basics/attach-cloud-storage), CVAT requires so-called [manifest files](https://opencv.github.io/cvat/docs/manual/advanced/dataset_manifest), which can be generated (from `/path/to/project-p/data` directory) with:

```shell
docker run -ti --rm -u "$(id -u)":"django" -v "$PWD":"/local" --entrypoint python3 cvat/server:"$TAG" utils/dataset_manifest/create.py --output-dir /local /local
```

Where `TAG=$(git tag -l | sort | tail -1)` from `/path/to/cvat` directory on the server where CVAT is being deployed (or just the same version of CVAT Docker image which is being deployed).

---

## Upload dataset to the cloud

*Project P* uses S3-like object storage, so some steps for GCP or Azure may differ.

### Create a bucket

Go to `Object storage` -> `Buckets` -> `+ Add`, provide the new bucket name, select storage class (`Icebox` - cold data storage, 'cause the data are not going to be overwritten/accessed frequently), select default ACL (`authenticated-read` - recomended, anyway permissions may be changed later manually).

### Create access credentials

For S3-like object storage one needs only two types of credentials: **access key id** and **secret access key**.

Go to the newly created bucket -> `Keys` -> `+ Add key`, provide a name (no need to limit access for now), then copy and save somewhere `Access key ID` and `Secret Key`.

### Upload dataset to the bucket

It's possible to do via:

1. `aws` command-line utility ([CLI](https://mcs.mail.ru/docs/en/base/s3/storage-connecting/s3-cli))
2. [SDK](https://mcs.mail.ru/docs/en/base/s3/storage-connecting/s3-sdk) like `boto3` Python package
3. File managers [that support S3](https://mcs.mail.ru/docs/en/base/s3/storage-connecting/s3-file-managers)
4. Web interface (yeah, here we go)

---

## Connect the cloud storage in CVAT

Following the official CVAT guide, go to `Cloud storages` -> `+` (add), provide a name for the connection, provider (`AWS S3` in this case), bucket name, authorization type (`Key id and secret access key pair` and fill in the saved access keys), endpoint url (e. g. https://hb.ru-msk.vkcs.cloud or blank for official AWS), region (`ru-msk`, may be added), and manifest (name of the manifest file created earlier `manifest.jsonl` - just put it into the bucket root).

---

## Create a GitHub repository for annotations

Just like [**this**](.) repository.

---

## Create a new task in CVAT

In CVAT ui go to `Tasks` -> `+` -> `+ Create a new task` (or `Projects` -> select a project -> `+` -> `+ Create a new task`) - task creation form.

First of all, provide the new task name, project name or set of labels (if the task is not within a project), subset (optionally), select files...

### Add files from the cloud storage

In `Select files` tab -> `Cloud storage` -> `Select cloud storage` (name of the connected cloud storage) -> `Select manifest file` (no option if it is the only one upon creation) -> `Files` (unfold and select files).

### Atatch Git for annotations

In `Advanced` section set `Dataset repository url` to an ssh form of git-url, plus annotations file with parent prefix, e. g.:

```
git@github.com:Shining-Future/project-p-annotations.git [task-name/annotations.xml]
```

> Annotations may be saved as either `*.zip` or `*.xml` files.

For `Choose format` select `CVAT for images 1.1`.

Field `Segment size` may be useful to set how many images may be in a job.

### Add SSH key for annotations repository

Finally hitting `Submit & Open` or `Submit & Continue` will fail and show a pop-up window with SSH key, that must be added account-wide e. g. in *GitHub*. After public key is added to the *GitHub* account, task submission will proceed (click one of the submission buttons).

> At this point newly created task should be up with cloud data source and Git-versioned annotations.

---

## Set up HTTPS for CVAT

*CVAT* comes with optional HTTPS support ([docker-compose.https.yml](https://github.com/opencv/cvat/blob/develop/docker-compose.https.yml)), that requires Let's Encypt set up.

### Let's Encrypt

Just follow *CVAT* official [installation guide](https://opencv.github.io/cvat/docs/administration/basics/installation/#deploy-secure-cvat-instance-with-https).

### Self-signed HTTPS

This is the most interesting option. Following the [*Ubuntu* security reference](https://ubuntu.com/server/docs/security-certificates) and some experience with *Traefik* and *Docker Compose* one needs the following steps:

1. Generate a CSR (certificate signing request) with `server.key` and `server.csr` as output files
2. Generate a self-signed certificate with `server.crt` as output (X.509 format puplic key)
3. Install `server.key` and `server.crt` to `/path/to/private` and `/path/to/public` respectively
4. Create a drop-in [*Traefik*](https://doc.traefik.io/traefik/getting-started/configuration-overview) [config](https://doc.traefik.io/traefik/reference/dynamic-configuration/file), say, `self-signed.yml` with contents:

```yaml
tls:
  certificates:
    - certFile: /path/to/public/server.crt
      keyFile: /path/to/private/server.key
```
5. Create a Docker Compose file, say, `docker-compose.self-signed.yml` for *CVAT*:

```yaml
services:
  cvat_server:
    labels:
      - traefik.http.routers.cvat.entrypoints=websecure
      - traefik.http.routers.cvat.tls=true

  cvat_ui:
    labels:
      - traefik.http.routers.cvat-ui.entrypoints=websecure
      - traefik.http.routers.cvat-ui.tls=true

  traefik:
    image: traefik:v2.4
    container_name: traefik
    command:
      - '--providers.docker.exposedByDefault=false'
      - '--providers.docker.network=cvat'
      - '--entryPoints.web.address=:80'
      - '--entryPoints.web.http.redirections.entryPoint.to=websecure'
      - '--entryPoints.web.http.redirections.entryPoint.scheme=https'
      - '--entryPoints.websecure.address=:443'
      - '--providers.file.directory=/etc/traefik/rules'
      # Uncomment to get Traefik dashboard
      # - "--entryPoints.dashboard.address=:8090"
      # - "--api.dashboard=true"
    ports:
      - 80:80
      - 443:443
    volumes:
      - /path/to/public/server.crt:/path/to/public/server.crt:ro
      - /path/to/private/server.key:/path/to/private/server.key:ro
      - /path/to/sef-signed.yml:/etc/traefik/rules/self-signed.yml:ro
```
6. Start CVAT with:

```shell
docker compose -f docker-compose.yml -f docker-compose.self-signed.yml up -d
```

After these manipulations CVAT shall work over 80 port (redirecting it to 443 and using self-signed certificates).

> Yeah, that's it. Browsers will complain about non-valid certificate and insecure connection (but we know it's ok).

> Nevertheless protocol will be HTTPS and in certificate viewer one can see the details of the created self-signed certificate.

---
