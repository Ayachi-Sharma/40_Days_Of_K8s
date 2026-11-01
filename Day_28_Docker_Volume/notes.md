# Docker Layered Architecture

## 1. Introduction

Docker uses a layered architecture to build and manage container images. Each Docker image consists of multiple layers stacked on top of one another.

These layers help Docker to:
- Reuse existing layers.
- Reduce image build time.
- Save disk space.
- Manage container files efficiently.
- Support persistent data storage.

## 2. Understanding Docker Layers

Docker image layers are created from instructions in a Dockerfile.

For example:

```dockerfile
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y nginx
COPY index.html /var/www/html/
CMD ["nginx", "-g", "daemon off;"]
```

Each instruction that changes the image filesystem generally creates a new image layer.

Docker image layers are **read-only**, which means they cannot be modified directly after the image is built.

Multiple containers created from the same image can share these read-only layers.

## 3. Read-Only Layers

- These layers belong to the Docker image.
- They contain the application code, dependencies, libraries, and other required files.
- They cannot be modified by running containers.
- Multiple containers can share the same image layers.

For example, if we create three containers using the same Nginx image, all three containers can share the underlying read-only image layers.

## 4. Container Layer (Writable Layer)

When a container is created from an image, Docker adds a writable layer on top of the read-only image layers.

This is called the **container layer**.

Characteristics:
- It is specific to each container.
- Any changes made inside the container are stored in this layer.
- It allows applications to create, modify, and delete files.
- Changes made in one container do not affect other containers using the same image.

**Example:**

```bash
docker run -it ubuntu bash
```

Inside the container:

```bash
touch demo.txt
```

The `demo.txt` file is created in the container's writable layer.

If the container is removed, this data is lost unless it was stored using persistent storage.

## 5. Copy-on-Write (CoW) Mechanism

Docker uses a mechanism called **Copy-on-Write**.

When a container wants to modify a file that already exists in a read-only image layer:

1. Docker locates the original file in the image layer.
2. It copies that file into the writable container layer.
3. The modification is performed on the copied file.
4. The original image layer remains unchanged.

This allows multiple containers to share the same image layers without affecting one another.

## 6. Storage Drivers

Storage drivers are responsible for managing Docker image layers and the container's writable layer.

Their responsibilities include:
- Managing read-only image layers.
- Managing writable container layers.
- Implementing the Copy-on-Write mechanism.
- Handling filesystem changes between layers.

Examples of Docker storage drivers:
- Overlay2
- AUFS
- Btrfs
- ZFS
- Device Mapper

**Note:** Overlay2 is a commonly used storage driver on Linux systems.

## 7. Volume Drivers and Persistent Data

The writable container layer is temporary. When a container is deleted, its writable layer is also deleted.

To preserve data even after removing a container, Docker provides volumes.

Volumes allow data to exist independently of the container lifecycle.

Characteristics:
- Data persists even after container deletion.
- Multiple containers can share the same volume.
- Volumes are useful for databases and application data.
- Docker manages volumes separately from the container's writable layer.

Example:

```bash
docker volume create mydata
```

Run a container with the volume:

```bash
docker run -d \
  --name mycontainer \
  -v mydata:/app/data \
  nginx
```

Here, `/app/data` inside the container is mounted to the Docker volume `mydata`.

Even if the container is removed, the volume and its data remain available.

## 8. Difference Between Storage Drivers and Volume Drivers

| Storage Drivers | Volume Drivers |
|---|---|
| Manage image and container filesystem layers. | Manage persistent storage volumes. |
| Handle Copy-on-Write operations. | Provide storage independent of container lifecycle. |
| Manage the writable container layer. | Store application data persistently. |
| Example: Overlay2. | Example: Local volume driver. |

## 9. Important Commands

```bash
# List Docker images
docker images

# Inspect image layers and metadata
docker image inspect nginx

# Check Docker storage driver
docker info

# List running containers
docker ps

# Create a Docker volume
docker volume create mydata

# List Docker volumes
docker volume ls

# Inspect a volume
docker volume inspect mydata

# Remove a volume
docker volume rm mydata
```

## 10. Conclusion

Docker uses a layered filesystem architecture in which image layers are read-only and the container layer is writable.

Storage drivers manage these layers and implement Copy-on-Write, while volume drivers help manage persistent data storage.

**Key takeaway:**
- Image layers = Read-only.
- Container layer = Writable and temporary.
- Storage driver = Manages filesystem layers.
- Volume driver = Manages persistent data.