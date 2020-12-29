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

# adding docker networks for efficient cross-container communication

create docker network

`docker network create goals-net`

now instead of running the mongodb container with port connected to the host, run the container with network we created

`docker run --rm -d --network goals-net --name mongodb mongo`

we still need to publish the port 80 with the backend because the fronend will be running on the browser of the host machine and will be sending api requests to backend via localhost:80

`docker run -d --rm --network goals-net --name goals-backend-c goals-backend-i`

we do not need to run the frontend container connected with the network we created because the frontend app will be running on the browser
`docker run -it --rm --name goals-frontend-c goals-frontend-i`