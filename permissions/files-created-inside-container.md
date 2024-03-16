## Avoiding Permission Issues

### Without docker compose
It's recommended that you create a non-root user inside the Dockerfile. Besides creating it, we also need to explicitly set the user ID and group ID.

Example:
```
FROM ubuntu

ARG USER_ID
ARG GROUP_ID

RUN addgroup --gid $GROUP_ID nameOfTheUser
RUN adduser --disabled-password --gecos '' --uid $USER_ID --gid $GROUP_ID nameOfTheUser

USER nameOfTheUser
```

To ensure the host user has permission to edit files created by the container, we need to build a fresh image with the host UID and GID.

During the image build process, we must specify the host user ID and group ID.

```
$ docker build -t myimage \
  --build-arg USER_ID=$(id -u) \
  --build-arg GROUP_ID=$(id -g) .
```
After the image is built, the host can edit the files created by the container.