---
title: Setting Up Everything on One Port
---

# One Port for Everything
Using this guide, you can handle all communications with your server (panel, TLS-enabled configs, and REALITY configs) through one (or two) ports.
The purpose of this setup is to make server communications more natural, bypass restrictions on a single port, or address similar needs.

::: tip Note
If you have changed your panel port over time and want previous subscription links to continue working, you can use this guide to make HAProxy listen on the old port and forward incoming traffic to the new local port so that both subscription links work. To do this, just add your previous port like port 443.
:::

In this guide, we'll use HAProxy to achieve our goal. Throughout the guide, we assume that the panel subdomain is panel.example.com,
the subdomain for TLS-enabled configs is sub.example.com, and the SNI address used in the REALITY config is reality.com.

So in this guide, we'll first install and configure HAProxy, then make necessary changes to the configs and panel to accept all traffic on one port. Finally, some additional notes are provided.

::: warning Attention
If you have previously used HAProxy to obtain SSL for your panel, you should use another method (we recommend UNIVCORN) to get SSL for the panel to avoid conflicts with these settings.
:::

## Installing and Configuring HAProxy

::: tip Note
In this guide, we install HAProxy directly on the server; if you prefer, you can install it in Docker yourself.

Also, if you plan to implement more complex rules in the future, remember to install HAProxy from its main repository and not from Linux repositories.
:::

First, run these commands for installation:

```bash
apt update
apt install -y haproxy
```

After installation, the HAProxy configuration file is located at `/etc/haproxy/haproxy.cfg`. Open this file with `nano` for editing.

Now, add the following configuration to the end of the configuration file after making changes according to the instructions, and save it.

::: code-group
```[haproxy.cfg]
listen front
 mode tcp
 bind *:443

 tcp-request inspect-delay 5s
 tcp-request content accept if { req_ssl_hello_type 1 }

 use_backend panel if { req.ssl_sni -m end panel.example.com }
 use_backend reality if { req.ssl_sni -m end reality.com }
 default_backend fallback

backend panel
 mode tcp
 server srv1 127.0.0.1:10000

backend fallback
 mode tcp
 server srv1 127.0.0.1:11000

backend reality
 mode tcp
 server srv1 127.0.0.1:12000 send-proxy
```
:::

HAProxy configurations consist of one or more frontends and one or more backends. Each frontend sends traffic to one of the backends based on rules defined in it. Understanding these two sections helps us better configure HAProxy.

Looking at this configuration, you can see that HAProxy listens on port 443 of the server and receives all traffic. Then, based on the SNI of the incoming traffic, it forwards it to a "local" port on the server, allowing us to differentiate between different types of traffic.

::: tip Note
In this configuration, a default backend is defined using default_backend, which handles any incoming traffic that doesn't match the two defined SNIs. You can remove this part of the code to block traffic that doesn't match the specified SNIs.
:::

After replacing your domains and placing this configuration at the end of the specified file, restart HAProxy with the following command to complete this stage.

```bash
systemctl restart haproxy
```

## Preparing Configurations
### Preparing REALITY Configuration
Let's say you want to have several different inbounds for each node or several different inbounds with different SNIs. If you simply place these inbounds one after another and set their ports to be the same, the connection will be disrupted and effectively impossible to establish.

Single-porting configs solves this problem. To do this, you need to change your config settings as follows (pay attention to lines 3, 4, and 13):

::: code-group
```json{3-4,13} [xray_config.json]
{
  "tag": "VLESS_TCP_REALITY",
  "listen": "127.0.0.1",
  "port": 12000,
  "protocol": "vless",
  "settings": {
    "clients": [],
    "decryption": "none"
  },
  "streamSettings": {
    "network": "tcp",
    "tcpSettings": {
      "acceptProxyProtocol": true
    },
    "security": "reality",
    "realitySettings": {
      "show": false,
      "dest": "x",
      "xver": 0,
      "serverNames": [
        "reality.com"
      ],
      "privateKey": "x",
      "shortIds": [
        ""
      ]
    }
  },
  "sniffing": {
    "enabled": true,
    "destOverride": [
      "http",
      "tls"
    ]
  }
}
```
:::

With these changes, your inbound listens on 127.0.0.1 (localhost) instead of 0.0.0.0, and you can create as many inbounds as you want in this way (with different local ports) and differentiate between them in HAProxy based on SNI.

### Preparing TLS-enabled Configurations
To have all types of TLS-enabled configs on one port, we use fallback (if you've previously used fallback for single-porting, skip this step and just match your fallback config port with HAProxy).

First, we need a fallback inbound. For this purpose, you can use the following inbound as an example:

::: code-group
```json{3-4,13} [xray_config.json]
{
    "tag": "TROJAN_FALLBACK_INBOUND",
    "port": 11000,
    "protocol": "trojan",
    "settings": {
        "clients": [],
        "decryption": "none",
        "fallbacks": [
            {
                "path": "/lw",
                "dest": "@vless-ws",
                "xver": 2
            },
            {
                "path": "/mw",
                "dest": "@vmess-ws",
                "xver": 2
            },
            {
                "path": "/tw",
                "dest": "@trojan-ws",
                "xver": 2
            }
        ]
    },
    "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
            "serverName": "SERVER_NAME",
            "certificates": [
                {
                    "ocspStapling": 3600,
                    "certificateFile": "/var/lib/marzban/certs/fullchain.pem",
                    "keyFile": "/var/lib/marzban/certs/key.pem"
                }
            ],
            "minVersion": "1.2",
            "cipherSuites": "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
            "alpn": [
                "http/1.1"
            ]
        }
    },
    "sniffing": {
        "enabled": true,
        "destOverride": [
            "http",
            "tls"
        ]
    }
},
```
:::

To better use this feature, it's good to understand its concept and operation. Fallback generally works in such a way that if the incoming traffic matches this inbound, it accepts it, and if not, it sends it to other inbounds based on the path. So, after placing the above inbound with fallback, we now define several other inbounds each with the path specified in the fallback inbound (if you already have such inbounds, just change their listen value to the defined values (@vless-ws, @vmess-ws, and @trojan-ws) and put their paths in the fallback inbound.

So the fallback inbound sends each incoming traffic to other inbounds based on path:

```
path = /lw     ->    listen: "@vless-ws"
path = /mw     ->    listen: "@vmess-ws"
path = /tw     ->    listen: "@trojan-ws"
```

So according to the above example, just match the listen and path sections of your inbounds with the fallback to run all configs on one port.

::: tip Note
Using fallback increases the server's processing load. You can assign a different subdomain for each of your configs and use HAProxy without the need for fallback to single-port TLS-enabled configs.
:::

::: warning Attention
Note that in inbounds where the listen value is in the form @xxx and used in fallback, remove the port line
:::

Now if you've single-ported the inbounds using this fallback method, enter the `.env` file and set the following variable equal to your fallback inbound tag:

```
XRAY_FALLBACKS_INBOUND_TAG = "TROJAN_FALLBACK_INBOUND"
```

## Preparing the Panel
As mentioned, our goal is to have all communications including the panel (subscription link) on one port. We previously entered the panel settings in the HAProxy configuration, and at this stage, we just need to match the port that the panel listens on with HAProxy. So for this, just edit the `.env` file and set the following variables equal to the defined value (or whatever you entered in HAProxy):

```
UVICORN_HOST = "127.0.0.1"
UVICORN_PORT = 10000
```

Now restart Marzban:

```bash
marzban restart
```

## Preparing Host Settings
Since the port you placed in the inbound was a local port and all traffic actually reaches your server from port 443, you need to manually change the port to 443 in the host settings of the configs you've created, otherwise the local ports will be set for the configs by default.

## Additional Notes:
::: warning Attention
HAProxy settings must also be applied to all node servers, or you can define separate inbounds for some node servers and listen directly on `0.0.0.0`.
:::

::: tip Note
In the provided HAProxy configuration, all traffic that doesn't match either panel.example.com or reality.com SNIs is forwarded to the fallback inbound, thus preventing the misuse of your IP as a clean Cloudflare IP
:::

::: warning Warning
If you use IP limiter, you must add the `send-proxy` phrase at the end of each server backend in HAProxy and also set the value `"acceptProxyProtocol": true` in your inbound config, as shown in the REALITY example above. If a config has `send-proxy` but doesn't have `"acceptProxyProtocol": true`, it won't connect.
:::
