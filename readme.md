# dockerizing the mongodb service

start the mongodb server

`docker run --rm -d -p 27017:27017 --name mongodb mongo`

# dockerizing the backend app

create the Dockerfile

change `localhost` in code to `host.docker.internal`

build backend app image

`docker build . -t goals-backend-i`

`docker run --rm -dp 80:80 --name goals-backend-c goals-backend-i`

# dockerizing the frontend app

create the Dockerfile

build frontend app image

`docker build . -t goals-frontend-i`

we should be able to run the frontend container in interactive mode with -it options

`docker run -it --rmp 3000:3000 --name goals-frontend-c goals-frontend-i`

# adding docker networks for efficient cross-container communication

create docker network

`docker network create goals-net`

now instead of running the mongodb container with port connected to the host, run the container with network we created

`docker run --rm -d --network goals-net --name mongodb mongo`

we still need to publish the port 80 with the backend because the fronend will be running on the browser of the host machine and will be sending api requests to backend via localhost:80

we also need to update the source code and replace `host.docker.internal` with `mongodb` and rebuild the image

`docker run -dp 80:80 --rm --network goals-net --name goals-backend-c goals-backend-i`

we do not need to run the frontend container connected with the network we created because the frontend app will be running on the browser

`docker run -it --rmp 3000:3000 --name goals-frontend-c goals-frontend-i`

# Adding Data persistance to mongodb with volumes

docker hub documentation shows us where does the mongodb stores data in the container. we can just attach a named volume to that location so that the container removal doesnot effect our data

`docker run -d --rm --network goals-net --name mongodb -v data:/data/db mongo`

# limiting access to mongodb container instance

from the official documentation on mongodb docker image here https://hub.docker.com/_/mongo we know that we can setup env variables to configure root username and password requirement.

start mongodb container with MONGO_INITDB_ROOT_USERNAME and MONGO_INITDB_ROOT_PASSWORD env variables

`docker run -d --rm --network goals-net --name mongodb -v data:/data/db -e MONGO_INITDB_ROOT_USERNAME=mongoadmin -e MONGO_INITDB_ROOT_PASSWORD=secret mongo`

```shell
docker run -d --rm --network goals-net --name mongodb-c \
-v data:/data/db \
-e MONGO_INITDB_ROOT_USERNAME=mongoadmin \
-e MONGO_INITDB_ROOT_PASSWORD=secret \
mongo
```
test with the following

```shell
docker run -it --rm --network goals-net mongo \
mongo --host mongodb-c \
    -u mongoadmin \
    -p secret \
    --authenticationDatabase admin \
    some-db
```
now according to doc here https://docs.mongodb.com/manual/reference/connection-string/ we need to update backend code mongodb connection string from

`mongodb://mongodb:27017/course-goals` to 

`mongodb://mongoadmin:secret@mongodb-c:27017/?authSource=admin`

and then rebuild image with command 

`docker build . -t goals-backend-i`

then start new container from backend image

`docker run -dp 80:80 --rm --network goals-net --name goals-backend-c goals-backend-i`

# improving the nodejs backend container to detect code changes, persist logs, persist npm libraries, use env variables for db connection string credentials

tobe able to make changes to the code and apply into container we need to bind-mount project folder to the working directory inside container. but we also need to save node_modules folder inside container from over-writing by the bind mount. for this we will create a anonymous volume attached to the node_modules folder. then finally we will create a named volume attached with logs folder to persist the logs folder.

```shell
docker run -dp 80:80 --rm --network goals-net \
-v logs:/app/logs \
-v /app/node_modules \
-v "$(pwd):/app" \
--name goals-backend-c goals-backend-i
 ```

 to be able to detect live changes and restart server inside container we need to add dev dependency inside the package.json file and configure start script for npm to start server with nodemon. then finally rebuild the image to apply the changes.

 ```js
 "devDependencies": {
    "nodemon": "2.0.4"
  }
```

```js
"scripts": {
    "start": "nodemon app.js"
  }
```

update Dockerfile `CMD` from `[ "node", "app.js"]` to `[ "npm", "start"]`

rebuild the image

`docker build . -t goals-backend-i`

rerun the container

```shell
docker run -dp 80:80 --rm --network goals-net \
-v logs:/app/logs \
-v /app/node_modules \
-v "$(pwd):/app" \
--name goals-backend-c goals-backend-i
 ```

 ## setting up default environment variables in dockerfile and using in nodejs code then passing different value when running the container with the run command

if we change our backend code to get db user and password from env varialbes like this

```js
let connection_str = "mongodb://${process.env.MONGODB_USER}:${process.env.MONGODB_PASSWORD}@mongodb-c:27017/?authSource=admin"
```

we can supply our container default env variable values like this in the docker file. we will have to build after this edit.

```shell
ENV MONGODB_USER=root
ENV MONGODB_PASSWORD=password
```

we will have to rebuild image after this step.

```shell
docker build . -t goals-backend-i
```

we can supply some other values by the -e option to the run command like this

```shell
docker run -dp 80:80 --rm --network goals-net \
-v logs:/app/logs \
-v /app/node_modules \
-v "$(pwd):/app" \
-e MONGODB_USER=mongoadmin \
-e MONGODB_PASSWORD=secret \
--name goals-backend-c goals-backend-i
```

 # Finally using .dockerignore file to ignore few files

 create a file named .dockerignore file and add the following lines

 ```
 node_modules
 .dockerignore
 .git
 ```