---
title: docker 安装v2ray
date: 2019-10-09 10:28:02
tags: [v2ray,caddy, docker]
categories: 网络
---

## before you begin

- Have a vps
- Have a domain
- Solve your domain to your vps's IP
## Start

### run v2ray
```golang
docker run -d --name v2ray -v /etc/v2ray:/etc/v2ray v2ray/official  v2ray -config=/etc/v2ray/config.json
```


### test v2ray container IP and port 
```golang
telnet 172.17.0.2 8888
```

### run caddy
docker run -d --restart=always -p 80:80 -p 443:443 -v /data/caddy:/caddy --name=caddy blob/caddy -conf="/caddy/conf/Caddyfile"



### Caddyfile (/data/caddy/Caddyfile)
```golang
blog.luohua13.cn {
    gzip
    root /caddy/html/
    index index.html index.htm
    proxy /app00741314 172.17.0.2:8888 {
        websocket
        header_upstream -Origin
    }
}
```


### server端config.json (/etc/v2ray/config.json)
```golang
{
  "inbounds": [
    {
      "port": 8888,
      "listen":"0.0.0.0"
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "b831381d-3434-4d53-ad4f-8cda48b30811",
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/app"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

### client端config.json
```golang
{
  "log": {
    "error": "",
    "loglevel": "info",
    "access": ""
  },
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": {
        "ip": "",
        "userLevel": 0,
        "timeout": 360,
        "udp": false,
        "auth": "noauth"
      },
      "port": "1080"
    }
  ],
  "outbounds": [
    {
      "mux": {
        "enabled": false,
        "concurrency": 8
      },
      "protocol": "vmess",
      "streamSettings": {
        "wsSettings": {
          "path": "/app",
          "headers": {
            "host": ""
          }
        },
        "tlsSettings": {
          "serverName": "your domain",
          "allowInsecure": true
        },
        "security": "tls",
        "network": "ws"
      },
      "tag": "agentout",
      "settings": {
        "vnext": [
          {
            "address": "your domain",
            "users": [
              {
                "id": "b831381d-3434-4d53-ad4f-8cda48b30811",
                "alterId": 64,
                "level": 0,
                "security": "none"
              }
            ],
            "port": 443
          }
        ]
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {
        "domainStrategy": "AsIs",
        "redirect": "",
        "userLevel": 0
      }
    },
    {
      "tag": "blockout",
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "none"
        }
      }
    }
  ],
  "dns": {
    "servers": [
      ""
    ]
  },
  "routing": {
    "strategy": "rules",
    "settings": {
      "domainStrategy": "IPIfNonMatch",
      "rules": [
        {
          "outboundTag": "direct",
          "type": "field",
          "ip": [
            "geoip:cn",
            "geoip:private"
          ],
          "domain": [
            "geosite:cn",
            "geosite:speedtest"
          ]
        }
      ]
    }
  },
  "transport": {}
}
```