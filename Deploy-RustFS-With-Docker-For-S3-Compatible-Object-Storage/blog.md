# Deploy RustFS With Docker For S3 Compatible Object Storage

Hey there, fellow tech enthusiasts!
Are you looking for a robust, high-performance solution to connect your applications to S3-compatible storage? RustFS is an excellent choice - it's a lightweight, fast filesystem implementation written in Rust that provides seamless integration with S3-compatible object storage services. In this comprehensive guide, I'll walk you through deploying RustFS using Docker. It's straightforward, beginner-friendly, and perfect if you want a quick setup for S3-style bucket storage without the hassle of full-blown cloud providers. Let's get into it!

## What is RustFS?
RustFS is a virtual filesystem that bridges local file system operations with S3-compatible object storage. It allows you to mount S3 buckets as regular directories, making it easy to work with cloud storage as if it were local disk space. Built with performance and reliability in mind, RustFS leverages Rust's memory safety and speed advantages.

## Prerequisites:

Before we begin, ensure you have:

- **Docker Installed**: Head over to the official Docker site and grab the latest version for your OS. If you're on Linux, a quick `sudo apt install docker.io` (or equivalent) should do it.
- **Host Directories Ready**: You'll need folders for data and logs. RustFS runs as user ID 10001 inside the container, so ownership matters to avoid permission headaches.
- **Basic Command Line Comfort**: We'll be using terminal commands, but nothing too fancy.

That's it. No Rust compiler or complex builds required, thanks to pre-built Docker images.

## Step-by-Step Deployment Guide

Alright, let's deploy this bad boy. I'll assume you're on a Linux machine, but the steps translate easily to Mac or Windows with Docker Desktop.

### Step 1: Prepare Your Directories

First, create directories for persistent storage. RustFS stores your buckets and objects here, so they survive container restarts.

```
mkdir -p data logs
```

Now, change ownership to UID 10001 (the default user in the RustFS container):

```
chown -R 10001:10001 data logs
```

This prevents "permission denied" errors when the container tries to write files.

### Step 2: Pull and Run the Docker Container

RustFS has official images on Docker Hub. We'll use the latest tag for simplicity.

Run this command:

```
docker run -d --name rustfs --restart=always -p 9000:9000 -p 9001:9001 -v $(pwd)/data:/data -v $(pwd)/logs:/logs rustfs/rustfs:latest
```

Breaking it down:
- `-d`: Runs in detached mode (background).
- `--name rustfs`: Assigns a name to the container.
- `--restart=always`: Restarts the container if it crashes.
- `-p 9000:9000`: Exposes the S3 API port.
- `-p 9001:9001`: Exposes the web console.
- `-v $(pwd)/data:/data`: Mounts your local `data` folder to the container's data path.
- `-v $(pwd)/logs:/logs`: Same for logs.

If you prefer a specific version (check the GitHub releases for the latest stable), swap `latest` with something like `1.0.0-alpha.76`.

Pro tip: If you're using Podman instead of Docker (for rootless containers), just replace `docker` with `podman` - it works the same.

### Step 3: Verify It's Running

Check if the container is up:

```
docker ps
```

You should see `rustfs` in the list. If not, peek at logs with `docker logs rustfs` to troubleshoot.

### Step 4: Access the Web Console

Fire up your browser and head to `http://localhost:9001` (or your server's IP if remote).

Default credentials:
- Username: `rustfsadmin`
- Password: `rustfsadmin`

Change these ASAP for security! Once logged in, you can create buckets, upload files, and manage access keys.

### Step 5: Test S3 Compatibility

To confirm it's working as S3 storage, let's use the AWS CLI (install it if you haven't: `pip install awscli`).

Configure it:

```
aws configure
```

Set:
- Access Key ID: `rustfsadmin`
- Secret Access Key: `rustfsadmin`
- Default region: `us-east-1` (or whatever)
- Output format: `json`

Then, create a bucket:

```
aws --endpoint-url http://localhost:9000 s3 mb s3://my-test-bucket
```

Upload a file:

```
echo "Hello, RustFS!" > test.txt
aws --endpoint-url http://localhost:9000 s3 cp test.txt s3://my-test-bucket/
```

List it:

```
aws --endpoint-url http://localhost:9000 s3 ls s3://my-test-bucket/
```

Boom! you've got S3-compatible storage running locally!

### Bonus: Using Docker Compose for Observability

If you want monitoring baked in, grab the `docker-compose.yml` from the RustFS GitHub repo (https://github.com/rustfs/rustfs). It includes Prometheus, Grafana, and more.

Run:

```
docker compose --profile observability up -d
```

This spins up the whole stack. Access Grafana at `http://localhost:3000` (default creds: admin/admin) to monitor performance.

For Podman users: `podman compose --profile observability up -d`.

## Configuration Tips and Best Practices

- **Security First**: Enable HTTPS by generating certs and mounting them via volumes. Check the docs for TLS setup.
- **Scaling Up**: For production, deploy in a cluster mode. RustFS supports multi-node setups, start with the GitHub docs for that.
- **Backups and Persistence**: Your data is in the mounted `data` folder, so back that up regularly.
- **Performance Tweaks**: RustFS shines with small objects, but for large files, experiment with multipart uploads in your apps.
- **Integration Ideas**: Hook it up to apps like Nextcloud, ML pipelines, or even as a backend for Kubernetes persistent volumes.

If you run into issues, the RustFS GitHub is active for issues and PRs.

## Wrapping Up

There you have it. A quick, painless way to deploy RustFS with Docker for your own S3-compatible bucket storage. It's a breath of fresh air in the object storage world, blending Rust's safety and speed with the familiarity of S3. I've been playing with it for a side project, and the performance boost is noticeable compared to older alternatives.

~~If you've got questions or tweaks, drop a comment below.~~

 Happy storing, and may your buckets always be full (but not overflowing)!