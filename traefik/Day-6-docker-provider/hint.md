# curl -H "Host: wrong.local" 127.0.0.1:80 you may reach with this

docker rm -f day6-webapp

docker run -d \
  --name day6-webapp \
  --label "traefik.enable=true" \
  --label 'traefik.http.routers.day6.rule=Host(`127.0.0.1`) || Host(`myapp.local`)' \
  --label "traefik.http.routers.day6.entrypoints=web" \
  --label "traefik.http.services.day6.loadbalancer.server.port=8080" \
  day6-webapp
