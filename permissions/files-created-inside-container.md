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

### With docker compose
There are two primary methods to bring data into a container: ADD/COPY and VOLUMES.

#### ADD/COPY
ADD/COPY takes the contents of a file or folder and copies them to a specified path in the container. This operation occurs during the build time of the container.

#### VOLUMES
Volumes act as a sort of symlink between a local file or folder on the host machine and a file or folder inside the container.

#### Why occurs errors using VOLUMES
Example: When we are running a docker instance on the host machine, and the app files are owned by the host user (hostMachineName) with an ID X. The container is running the php on a usar called www-data that has an id of Y. And when trying to modify files under the application directory, the difference between those two ownerships causes conflict in the permissions.

We could only change the file permissions with the chown command, but if the containers went down and after that bring back again, the permissions are resetted, and we'll have to do the same chown thing again.

We could add the command to the Dockefile to change the permission for us in the building proccess, but wont work, because the volumes are mounted after the container build process runs.

#### Way to fix
The code bellow creates a group called laravel, with the same group id as the host machine's group that owns the app's files. The same thing is made for the user.

-g: pass in a group ID that we want to attach to this new group
${GID}: an environment variable for the group ID passed in through Docker Compose
--system: it's a system-wide group
groupLaravel: the name of the group we're creating

```
RUN addgroup -g ${GID} --system groupLaravel
RUN adduser -G groupLaravel --system -D -s /bin/sh -u ${UID} userLaravel
```
-G: the name of the group we want to assign this user to, an in our case it's the group we just created
-D: don't create a password for this user
-s /bin/sh: give it the Alpine Linux shell
-u ${UID}: pass in a user ID that we want to attach to this new user, and like the group ID it's coming through an environment variable
userLaravel: the name of the user we're creating

But, we still have to modify the user that PHP is running in the container, because by default uses www-data. This could be done copying over a php.ini file, or we can do in the dockerfile, with the code below.
```
RUN sed -i "s/user = www-data/user = userLaravel/g" /usr/local/etc/php-fpm.d/www.conf
RUN sed -i "s/group = www-data/group = groupLaravel/g" /usr/local/etc/php-fpm.d/www.conf
```

The code above, uses sed to change the user and group that PHP runs as.

But we still have to set the environment variables. The code bellow defines the env variable that will be filled by the docker-compose.yml, to get the host user and group id.
```
ARG UID
ARG GID
 
ENV UID=${UID}
ENV GID=${GID}
```


The arguments UID and GID in the code bellow, are environment variables pulled from the terminal. If a UID or GID isn't found, value for each is set to 1000 (meaning the :- operator)
```
php:
    build:
      context: .
      dockerfile: php.dockerfile
      args:
        - UID=${UID:-1000}
        - GID=${GID:-1000}
```

So, before build the containers, we have to check if the UID and GID has value. We do that with the command echo
```
echo $UID
echo $GID
```

If the values are filled, we don't need to export them, otherwise we have to export those with the command bellow.
```
export UID=$(id -u) && export GID=$(id -g)
```

After that, we can build the containers, and in the building time the laravel group and user will be created with the host user ID and group ID. The php process will then use the new laravel user to run as, meaning that any write access to the app filesystem should be granted since the defining user attributes for the permissions now match.
