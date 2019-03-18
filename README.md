# Docker Swarm 建立

1. docker swarm init (swarm manager)
```cmd
>>get manager token 
  sudo docker swarm join-token manager
>>get worker token 
  sudo docker swarm join-token worker
```

2. add node (swarm worker)
```cmd
docker swarm join \
    --token SWMTKN-1-25528wedero5nw877tarq69i1h4c5rkuyqs6vkq5h4147hea1t-8tu2bjuasicrzkq0v0vrbyfkb \
    10.142.0.12:2377
```
```cmd
>>(swarm manager)
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

