# Golang Backend with Docker

This repository contains a Docker setup for building and deploying a Golang backend application using Ubuntu.

## Getting Started

Follow these instructions to build and run the application inside a Docker container.

### Prerequisites

Ensure you have the following installed on your machine:
- [Docker](https://www.docker.com/get-started)

## Building the Docker Image

Run the following command to build the backend Docker image:

```sh
docker build -t my-go-app .
```

This command will:
1. Use Golang to install dependencies and build the backend application.
2. Copy the built binary into a minimal Ubuntu container.

## Running the Container

Once the image is built, you can run the container using:

```sh
docker run -d -p 8080:8080 --name go-container my-go-app
```

This command:
- Runs the container in detached mode (`-d`).
- Maps port 8080 on the host to port 8080 in the container.
- Names the running container `go-container`.

## Stopping and Removing the Container

To stop the running container, use:

```sh
docker stop go-container
```

To remove the container after stopping it:

```sh
docker rm go-container
```

## Accessing the API

Once the container is running, you can access the API at:

```
http://localhost:8080
```

## Cleaning Up

To remove the Docker image, first, remove any running containers, then run:

```sh
docker rmi my-go-app
```

## License

This project is open-source and available
