

```
FROM registry.example.com:8443/ubi8/nodejs-16:1

LABEL org.opencontainers.image.authors="Your Name"
LABEL com.example.environment="production"
LABEL com.example.version="0.0.1"

ENV SERVER_PORT=3000
ENV NODE_ENV="production"

EXPOSE $SERVER_PORT

WORKDIR /opt/app-root/src

COPY . .

RUN npm install

CMD npm start
```

-------------------------

# ENV 

The following example declares a DB_HOST variable with the database hostname.

`ENV DB_HOST="database.example.com"`

Then you can retrieve the environment variable in your application. The following example is a Python script that retrieves the DB_HOST environment variable.

```
from os import environ

DB_HOST = environ.get('DB_HOST')

# Connect to the database at DB_HOST...
```

# ARG

Use the ARG instruction to define build-time variables, typically to make a customizable container build.

`podman build --build-arg key=example-value`

Developers commonly configure the `ENV` instructions by using the ARG instruction. This is useful for preserving the build-time variables for runtime.

If you do not configure the `ARG` default values but you must configure the ENV default values, then use the ${VARIABLE:-DEFAULT_VALUE} syntax, such as:

```
ARG VERSION \
    BIN_DIR

ENV VERSION=${VERSION:-1.16.8} \
    BIN_DIR=${BIN_DIR:-/usr/local/bin/}

RUN curl "https://dl.example.io/${VERSION}/example-linux-amd64" \
        -o ${BIN_DIR}/example
```

# VOLUME

Use the `VOLUME` instruction to persistently store data. 

```
FROM registry.redhat.io/rhel9/postgresql-13:1

VOLUME /var/lib/pgsql/data
```

`podman inspect VOLUME_ID`

To remove unused volumes, use the podman volume prune command.

`podman volume prune`

`podman volume ls --format="{{.Name}}\t{{.Mountpoint}}"`

# ENTRYPOINT and CMD 

## ENTRYPOINT 

- defines an executable, or command, that is always part of the container execution. This means that additional arguments are passed to the provided command.

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
ENTRYPOINT ["echo", "Hello"]
```

```
podman run my-image
Hello
```
```
podman run my-image Red Hat
Hello Red Hat
```

## CMD

- passing arguments to the container overrides the command provided in the CMD instruction.

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
CMD ["echo", "Hello", "Red Hat"]
```
```
podman run my-image
Hello Red Hat
```

- Running a container with the argument whoami overrides the echo command.

```
podman run my-image whoami
root
```

- When a `Containerfile` specifies both `ENTRYPOINT` and `CMD` then `CMD` changes its behavior. In this case the values provided to CMD are passed as default arguments to the ENTRYPOINT.

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
ENTRYPOINT ["echo", "Hello"]
CMD ["Red", "Hat"]
```

```
podman run my-image
Hello Red Hat
```
```
podman run my-image Podman
Hello Podman
```

- The `ENTRYPOINT` and `CMD` instructions have two formats for executing commands:

`Text array`

`ENTRYPOINT ["executable", "param1", ... "paramN"]`

- In this form you must provide the full path to the executable.

`String form`

The command and parameters are written in a text form, such as:

`CMD executable param1 ... paramN`

- The string form wraps the executable in a shell command such as `sh -c "executable param1 …​ paramN"`. 
- This is useful when you require shell processing, for example for variable substitution.

# Multistage Builds

- uses multiple `FROM` instructions to create multiple independent container build processes, also called `stages`. 
- Every stage can use a different base image and you can copy files between stages. The resulting container image consists of the last `stage`.
- Multistage builds can reduce the image size by only keeping the necessary runtime dependencies. 

Example: 

```
FROM registry.redhat.io/ubi8/nodejs-14:1

WORKDIR /app
COPY . .

RUN npm install
RUN npm run build

RUN serve build
```

- `npm install` command installs the required NPM packages, which includes packages that are only needed at build-time. 
- `npm run build` command uses the packages to create an optimized production-ready application build. 

- the image contains both the build-time and runtime dependencies, which increases the image size. - The resulting image also contains the Node.js runtime, which is not used at container runtime but might increase the attack surface of the container.

To avoid this issue, you can define two stages:

1. First stage: Build the application.

2. Second stage: Copy and serve the static files by using an HTTP server, such as NGINX or Apache Server.

The following example implements the two-stage build process.

```
# First stage
FROM registry.access.redhat.com/ubi8/nodejs-14:1 as builder 
#Define the first stage with an alias. The second stage uses the builder alias to reference this stage.

COPY ./ /opt/app-root/src/
RUN npm install
RUN npm run build  
#Build the application.

# Second stage
FROM registry.access.redhat.com/ubi8/nginx-120 
#Define the second stage without an alias. It uses the ubi8/nginx-120 base image to serve the production-ready version of the application.

COPY --from=builder /opt/app-root/src/ /usr/share/nginx/html 
#Copy the application files to a directory in the final image. The --from flag indicates that Podman copies the files from the builder stage.
```
