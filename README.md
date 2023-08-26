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

---