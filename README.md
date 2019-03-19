# Docker Swarm 建立

1. docker swarm init (swarm manager)
```cmd
* get manager token 
  sudo docker swarm join-token manager
* get worker token 
  sudo docker swarm join-token worker
```

2. add node (swarm worker)
```cmd
docker swarm join \
    --token SWMTKN-1-25528wedero5nw877tarq69i1h4c5rkuyqs6vkq5h4147hea1t-8tu2bjuasicrzkq0v0vrbyfkb \
    10.142.0.12:2377
```
```cmd
* (swarm manager)
docker node ls
```

3. add label (swarm manager)
```cmd 
docker node update --label-add service=nodejs docker-node1
docker node update --label-add service=nodejs docker-node2
docker node update --label-add service2=springboot docker-backend1
docker node update --label-add service2=springboot docker-backend2
docker node update --label-add service=haproxy --label-add node=manager  docker-haproxy
```
```cmd 
docker node ls -q | xargs docker node inspect   -f '{{ .ID }} [{{ .Description.Hostname }}]: {{ .Spec.Labels }}'
```

# config / deploy haproxy
```cmd
defaults
  mode http
  option  httplog
  timeout connect 4000ms
  timeout client 50000ms
  timeout server 50000ms

  stats enable
  stats refresh 5s
  stats show-node
  stats uri  /stats/haproxy  # http://0.0.0.0:8080/stats/haproxy
  stats auth admin:123456

global
  log 0.0.0.0:1514 local0 debug
  user haproxy
  group haproxy

frontend http_front_api
    log global    
    bind *:8081  
    default_backend front_api_back
    
# Configure HAProxy to route requests to swarm nodes on port
backend front_api_back
    log global
    balance roundrobin
    server docker-node1 10.142.0.10:8300 check
    server docker-node2 10.142.0.11:8300 check
```
build image
```cmd
docker build -t swarm-haproxy:1.0.0 -f Dockerfile-haproxy .
docker tag  swarm-haproxy:1.0.0 asia.gcr.io/jovial-archive-216204/swarm-haproxy:gcp
gcloud docker -- push asia.gcr.io/jovial-archive-216204/swarm-haproxy:gcp
```

# config docker-stack.yml && Deploy

1.docker-stack.yml
```cmd
version: '3'
services:
    swarm-api:
        image: 'docker.io/richardkang/swarm-backend:1.0.0'
        ports:
            - '4000:4000'
        networks:
            - backend
        environment:
            - DEMO_ALLOWEDORIGINS=http://35.243.156.252:8081
            - SERVER_PORT=4000
        deploy:
            replicas: 3
            update_config: {parallelism: 1}
            restart_policy: {condition: on-failure}
            placement:
                constraints: ['node.labels.service2 == springboot']
    swarm-web:
        image: 'docker.io/richardkang/swarm-frontend:gcp'
        ports:
            - '8300:80'
        networks:
            - backend
        environment:
            - VUE_APP_BACKEND_HOST=http://35.243.156.252:4000
        depends_on:
            - swarm-api
        deploy:
            replicas: 3
            update_config: {parallelism: 1}
            restart_policy: {condition: on-failure}
            placement:
                constraints: ['node.labels.service == nodejs']
    swarm-haproxy:
        image: 'docker.io/richardkang/swarm-haproxy:gcp'
        ports:
            - '8081:8081'
            - '8082:8082'
        networks:
            - backend
        depends_on:
            - sgms-api
        deploy:
            replicas: 1
            update_config: {parallelism: 2}
            restart_policy: {condition: on-failure}
            placement:
                constraints: ['node.labels.service == haproxy']                
    visualizer:
        image: 'dockersamples/visualizer:stable'
        ports:
            - '8083:8080'
        stop_grace_period: 1m30s
        volumes:
            - '/var/run/docker.sock:/var/run/docker.sock'
        deploy:
            replicas: 1
            update_config: {parallelism: 2}
            restart_policy: {condition: on-failure}
            placement:
                constraints: ['node.labels.node == manager']                  
networks:
    backend:                       

```  

2. docker stack deploy -c docker-stack.yml my_stack

# Swarm command
```cmd
 docker stack ls
 docker stack ps $StackName
 docker stack services $StackName
 docker stack rm $StackName

 docker node ls

 docker service ls
 docker service logs -f $ID
 docker service create --name swarm-api-report -p 4100:4000 --replicas 2 --detach=false \
 --placement-pref 'spread=node.labels.service2.springboot' \
 --env DEMO_ALLOWEDORIGINS='http://35.243.156.252:8081' \
 --env SERVER_PORT=4000 \
 docker.io/richardkang/swarm-backend:1.0.0
 docker service rm $ID

 docker service scale my_stack_swarm-web=2
 docker service scale my_stack_swarm-api=2
```

# Swarm 
