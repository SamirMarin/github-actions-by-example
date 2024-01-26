# The build and push workflow job
The build and push workflow job plays a pivotal role in our CI/CD pipeline by automating the creation and distribution of our service's Docker image. This job ensures that the latest version of our service is containerized and ready for deployment.

## Docker and Its Components
Before diving into the workflow job, it's crucial to comprehend the foundational concepts of Docker and the role of containers in modern software deployment:

1. Docker Image: This is essentially the blueprint or recipe for your application. It is a lightweight, standalone, and executable package that bundles everything necessary to run your software. This includes not just your code, but also the runtime, system tools, libraries, and settings.
2. Docker Container: When a Docker image is executed, it becomes a container. Think of the container as the live, running instance of the image. Containers encapsulate and run your application, isolating it from the underlying system and other containers. This ensures consistency across different environments, as the container runs the same regardless of where it's deployed.

## Dockerfile
To implement the build and push workflow job, we need to have a Dockerfile in our repository.

The Dockerfile is a crucial component for containerizing your service. It defines the steps to assemble the Docker image. Let's make a Dockerfile and break down the contents of the file

create a file called `Dockerfile` at the root of the repo

```yaml
touch Dockerfile
```

```Dockerfile
FROM --platform=$BUILDPLATFORM golang:1.21 as builder

WORKDIR /app

COPY go.mod ./
COPY go.sum ./
RUN go mod download

COPY . .

ARG TARGETOS
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o app


FROM alpine

WORKDIR /app

# add binary
COPY --from=builder /app/app app

ENTRYPOINT ["/app/app"]
```

Stage 1: Building the application
1. Base Image: FROM --platform=$BUILDPLATFORM golang:1.21 as builder
   This line sets the base image for the build stage. It uses a Golang image, version 1.21, and names this stage 'builder'. The --platform=$BUILDPLATFORM part ensures that the build process uses the appropriate platform (OS/architecture).
2. Setting the Working Directory: WORKDIR /app
   This command sets /app as the working directory for any subsequent instructions.
3. Copying Go Module Files and Downloading Dependencies:
   ```Dockerfile
   COPY go.mod ./
   COPY go.sum ./
   RUN go mod download
   ```
   These lines copy the go.mod and go.sum files to the container and download the project dependencies. This ensures that your Go project's dependencies are correctly managed and isolated within the Docker container.
4. Copying the Source Code: COPY . .
   This command copies your project's source code into the container.
5. Compiling the Application:
   ```Dockerfile
   ARG TARGETOS
   ARG TARGETARCH
   RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o app
   ```
   These lines introduce build arguments for the target OS and architecture, allowing for a more flexible build process. The application is compiled with these parameters, and CGO is disabled to ensure that the binary is statically linked and can run in a minimal environment like Alpine Linux.

Stage 2: Preparing the Runtime Image
1. Base Image: FROM alpine
   This line sets the base image for the final stage to Alpine Linux, a minimal and lightweight Linux distribution, ideal for running applications in a containerized environment.
2. Setting the Working Directory: WORKDIR /app
   Similar to the build stage, it sets the working directory to /app for any subsequent instructions.
3. Copying the Compiled Binary: COPY --from=builder /app/app app
   This command copies the compiled application binary from the 'builder' stage into this stage. It ensures that only the necessary binary and files are included in the final image, keeping it lightweight.
4. Setting the Entry Point: ENTRYPOINT ["/app/app"]
   This line defines the default executable for the container. When the container starts, it will execute the application binary.

By using multiple stages, separating the build environment from the runtime environment. We ensure a cleaner, more secure final image, as it contains only what's necessary to run the application.

