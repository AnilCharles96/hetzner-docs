# Deploying sentry to kubeadm cluster


I have a hetzner dedicated server where I use proxmox to create virtual machines and I am running kubeadm cluster in those VMs. 

Nginx will proxy to 10.0.0.105:32496 (virtual machine where kubeadm worker node is running) this is the Load balancer service with NodePort exposed by gateway api using cilium gatewayclass.

The following is the Nginx configuration for sentry in host machine. 

```
upstream backend_pool {
    server 10.0.0.105:32496; # worker node -> gateway api
    keepalive 32;
}

server {

    server_name sentry.example.com;

    #modsecurity on;
    #modsecurity_rules_file /etc/nginx/modsec/modsecurity.conf;

    location ~ ^/api/[1-9]+/(envelope|minidump|security|store|unreal)/ {
     proxy_pass http://backend_pool;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header X-Forwarded-Proto $scheme;
     #proxy_set_header Cookie $http_cookie;
     proxy_http_version 1.1;
     proxy_set_header Connection "upgrade";
     proxy_set_header Upgrade $http_upgrade;

     add_header Access-Control-Allow-Origin * always;
     add_header Access-Control-Allow-Credentials false always;
     add_header Access-Control-Allow-Methods GET,POST,PUT always;
     add_header Access-Control-Allow-Headers sentry-trace,baggage always;
     add_header Access-Control-Expose-Headers sentry-trace,headers always;
    }
  
 
    location / {
        proxy_pass http://backend_pool;

        # Use proxy params
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Referer $http_referer;
    }


    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/sentry.example.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/sentry.example.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

server {
    if ($host = sentry.example.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    server_name sentry.example.com;
    listen 80;
    return 404; # managed by Certbot


}

```


# Gateway and httproute for sentry
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cilium-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: sentry-route
  namespace: sentry
spec:
  parentRefs:
    - name: cilium-gateway
      namespace: default
  hostnames:
    - sentry.example.com 
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api/1/envelope
        - path:
            type: PathPrefix
            value: /api/1/store
        - path:
            type: PathPrefix
            value: /api/1/unreal
        - path:
            type: PathPrefix
            value: /api/1/minidump
        - path:
            type: PathPrefix
            value: /api/1/security
        - path:
            type: PathPrefix
            value: /api/2/envelope
        - path:
            type: PathPrefix
            value: /api/2/store
        - path:
            type: PathPrefix
            value: /api/2/unreal
        - path:
            type: PathPrefix
            value: /api/2/minidump
        - path:
            type: PathPrefix
            value: /api/2/security
      backendRefs:
        - name: sentry-relay
          port: 3000  
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: sentry-web
          port: 9000
```