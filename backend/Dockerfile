FROM node

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

EXPOSE 80

ENV MONGODB_USER=root
ENV MONGODB_PASSWORD=password
# these user and pasword env varialbes are default values
# we can supply actual values by -e command when running a container

CMD [ "npm", "start" ]