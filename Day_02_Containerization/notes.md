# Dockerize a React Application Using Docker

## Prerequisites

**Disclaimer:** Docker must be installed, and the Docker CLI must be working. On Windows or macOS, you can use Docker Desktop. On Linux, you can install Docker Engine and the Docker Compose plugin.

Run the commands from the project directory containing your `package.json` file.

## Step 1: Choose an Application

Select an existing application that you want to containerize. For this task, I used my React + Vite frontend application.

**GitHub Repository:** [Tech.Care](https://github.com/Ayachi-Sharma/Tech.Care)

## Step 2: Initialize Docker Configuration

Open the project folder in your IDE and run the following command in the terminal:

```bash
docker init
```

This command generates four configuration and documentation files with sensible defaults to help you containerize your application.


`Dockerfile`        Contains instructions for building a Docker image of the application. 
`.dockerignore`     Files and directories to exclude from the Docker build context,ex: `node_modules`. 
`compose.yaml`      Defines how Docker Compose should build and run the application. 
`README.Docker.md`  Provides instructions for building and running the application using Docker. 


## Step 3: Build the Image and Start the Container

Run the following command from the project directory:

```bash
docker compose up --build
```

**Explanation:**

- `docker compose up` creates and starts the services defined in `compose.yaml`.
- `--build` builds the Docker image before starting the services.

## Step 4: Verify the Image and Container

To list the Docker images available on your machine, run:

```bash
docker images
```

To list running containers, run:

```bash
docker ps
```

These commands help verify that the image has been built and the container is running.

## Step 5: Access the Application

To run the built image and publish the application's port, use:

```bash
docker run --rm -p 4173:4173 coalitiontechnologies-server:latest
```

**Explanation:**

- `docker run` creates and starts a container from the specified image.
- `--rm` automatically removes the container when it stops.
- `-p 4173:4173` maps port `4173` on the host machine to port `4173` inside the container.
- `coalitiontechnologies-server:latest` is the image name and tag.

Once the container is running, open the following URL in your browser:

[http://localhost:4173]

You should now be able to access your React application.

**Note:** If you have already started the application using `docker compose up`, you do not need to run a separate `docker run` command at the same time. Stop the Compose-managed container first, or use the appropriate Compose workflow.

## Key Learnings

- `docker init` helps generate the initial Docker configuration files.
- A Dockerfile defines how an image is built.
- A Docker image is a template used to create containers.
- Docker Compose simplifies building and running an application using a configuration file.
- Port mapping makes an application running inside a container accessible from the host machine.
- `.dockerignore` and `.gitignore` serve different purposes: one controls the Docker build context, while the other controls which files Git ignores.

Through this exercise, I learned how to prepare a React application for containerization, build a Docker image, run a container, and access the application locally.

**Project Repository:** [https://github.com/Ayachi-Sharma/Tech.Care]