# Nginx, Docker/multi-composers, Pm2, Cluster, Nodejs, Redis, Mongo, Babel and ES6

This is an essential example to build docker with multi composers, pm2, nodejs/es6, nginx, redis, mongo and cluster.

Step to run
1. Clone the [repo](https://github.com/diegothucao/multi-composers-pm2-cluster-nginx-nodejs-es6-redis-mongo)
2. Run development mode `docker-compose -f docker-compose.dev.yml up --build --remove-orphans` or Production mode `docker-compose -f docker-compose.prod.yml up --build --remove-orphans`. You you want to delete all old containers and images, try `docker rm $(docker ps -a -q)` and `docker rmi $(docker images -q)`
3. Open [localhost](http://localhost) to see server response data
4. Test Redis
	- Run [set data redis](http://localhost/store/diego)
	- Run [get data redis](http://localhost/diego)
5. Connect to database: Try username/ password/ database: `admin/ admin/ diego`. The port of development mode is `27017` and production mode is `27018`

create basic `Nodejs` code  
```javascript 

import express from 'express'
import cors from 'cors'
import { urlencoded, json } from 'body-parser'
import dotenv from 'dotenv'
import redisClient from './redis-client'

dotenv.load()

const app = express()
app.use(urlencoded({ extended: true, limit: '500mb'}))
app.use(json({ extended: true, limit: '500mb'}))
app.use(cors())

app.get('/', (_, res) => {
  res.send('Diego Cao: Hello')
})

// set data to Redis
app.get('/store/:key', async (req, res) => {
  const { key } = req.params;
  const value = req.query;
  await redisClient.setAsync(key, value)
  return res.send('Success')
})

// get data from Redis 
app.get('/:key', async (req, res) => {
  const { key } = req.params;
  const rawData = await redisClient.getAsync(key);
  return res.json(awData);
})

let server = app.listen(process.env.PORT || 8080)
server.setTimeout(500000)
```
Multi target 

```python
FROM node:10 as base
RUN mkdir -p /usr/diego
WORKDIR /usr/diego
COPY package*.json ./
RUN npm install -g pm2 yarn
RUN yarn install
COPY . .
RUN yarn run build

FROM base as diego-dev
EXPOSE 8080 80
CMD [ "pm2", "start", "ecosystem.config.js", "--env", "development", "--no-daemon" ]

FROM base as diego
EXPOSE 8081 80
CMD [ "pm2", "start", "ecosystem.config.js", "--env", "production", "--no-daemon" ]
```

Docker composer 

```
version: '3.4'
services:
  proxy:
    restart: always
    container_name: diego-dev-nginx
    build:
      dockerfile: Dockerfile
      target: diego-dev-nginx
      context: ./nginx
    ports:
      - "80:80"
    links:
      - diego-dev
  
  diego-dev-redis:
    restart: always
    image: redis
    container_name: diego-dev-redis
    expose:
      - 6379

  diego-dev-db:
    image: aashreys/mongo-auth:latest
    container_name: diego-dev-db
    restart: always
    volumes:
    - ./data/db:/var/mongo-dev-data/mongodb/data/db
    ports:
    - "27017:27017"
    command: mongod --port 27017
    expose:
      - 27017
    environment:
      AUTH: "yes"
      MONGODB_ADMIN_USER: root
      MONGODB_ADMIN_PASS: root
      MONGODB_APPLICATION_DATABASE: diego
      MONGODB_APPLICATION_USER: admin
      MONGODB_APPLICATION_PASS: admin

  diego-dev:
    build:
      context: .
      target: diego-dev
      dockerfile: ./Dockerfile
    container_name: diego-dev
    restart: always
    ports:
      - "8080:8080"
    links:
      - diego-dev-redis
      - diego-dev-db
    volumes:
      - .:/diego
    tty: true
    environment:
      PORT: 8080
      REDIS_URL: redis://diego-dev-redis
      MONGO_URL: mongodb://diego-dev-db/diego  
```
PM2 configuration 
```javascript
module.exports = {
  apps : [{
    name      : 'diego-service',
    script    : 'dist/app.js',
    exec_mode: 'cluster',
    instances: "max",
    env: {
      name : 'diego-dev',
      NODE_ENV: 'development'
    },
    env_production : {
      name : 'diego-pro',
      NODE_ENV: 'production'
    }
  }]
}
```

Nginx configuration 
```
upstream diego-dev {
  server diego-dev:8080;
}

server {
    listen              80;    
    # error_page 404 /404.html;
    #     location = /40x.html {}

    # error_page 500 502 503 504 /50x.html;
    #     location = /50x.html {}
    location / {
      proxy_pass http://diego-dev;
    }
}
```
	
If you see any issue, please do not hesitate to create an issue here or can contact me via email cao.trung.thu@gmail.com or [Linkedin](https://www.linkedin.com/in/diegothucao/)

Thanks
	
references
 1. https://docs.docker.com/install/	
 2. http://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/
 3. https://codefresh.io/docker-tutorial/node_docker_multistage/
 4. https://github.com/aashreys/docker-mongo-auth
