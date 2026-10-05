traefik routes /broken to a wrong port.

>root@ubuntu:~$ cat /etc//traefik/dynamic.yml
>http:
>
> routers:
>
>    root:
>      rule: "Path(`/`)"
>      entryPoints:
>        - web
>      service: backend
>
>    api:
>      rule: "Path(`/api`)"
>      entryPoints:
>        - web
>      service: backend
>
>    broken:
>      rule: "Path(`/broken`)"
>      entryPoints:
>        - web
>      service: broken-service
>
>  services:
>
>    backend:
>      loadBalancer:
>        servers:
>          - url: "http://127.0.0.1:8080"
>
>    broken-service:
>      loadBalancer:
>        servers:
>          - url: "http://127.0.0.1:9999"
