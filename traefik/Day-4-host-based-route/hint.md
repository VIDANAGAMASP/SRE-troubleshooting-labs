# backend also check host header and only accept web.example..com and api.example.com
# traefik routes to backend based on www.example.com and api.example.com
# web.exmple.com gets rejected by traefik and www.example.com gets rejected by the application
