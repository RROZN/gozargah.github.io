---
title: Activating CloudFlare Warp
---

# Activating CloudFlare Warp

Using this guide, you can bypass some restrictions imposed on your IP by large companies like Google and Spotify, and use their services without any issues.

::: warning
Please note that Warp configurations have a connection limit of maximum 5 simultaneous devices. To solve this issue, you can use multiple configurations.
:::

## Step One: Creating Wireguard Configuration

### First Method: Using Windows

- First, you need to download the required `Asset` from the [releases](https://github.com/ViRb3/wgcf/releases) section. This file varies depending on your processor.
- Rename the `Asset` file to `wgcf`.
- Now in the File Explorer address bar, enter `cmd.exe`.

![image](https://github.com/Gozargah/gozargah.github.io/assets/50927468/fb9f3eae-8390-45a5-a7b3-c50db4aa82a1)

- In the opened terminal, enter `wgcf.exe`.
- Enter the command `wgcf.exe register` once and then `wgcf.exe generate`.
- A new file named `wgcf-profile.conf` will be created, and this is the Wireguard file we need.
- Your configuration is ready, and you can use it.

### Second Method: Using Linux

- First, you need to download the required `Asset` from the [releases](https://github.com/ViRb3/wgcf/releases) section. This file varies depending on your processor.
- You can do this using the `wget` command.

For AMD64 architecture processors:
```bash
wget https://github.com/ViRb3/wgcf/releases/download/v2.2.22/wgcf_2.2.22_linux_amd64
```
For ARM64 architecture processors:
```bash
wget https://github.com/ViRb3/wgcf/releases/download/v2.2.22/wgcf_2.2.22_linux_arm64
```
Change the file path to `/usr/bin/` and rename it to `wgcf`.

For AMD64 architecture processors:
```bash
mv wgcf_2.2.22_linux_amd64 /usr/bin/wgcf
chmod +x /usr/bin/wgcf
```
For ARM64 architecture processors:
```bash
mv wgcf_2.2.22_linux_arm64 /usr/bin/wgcf
chmod +x /usr/bin/wgcf
```
Then use these 2 commands to create the configuration:
```bash
wgcf register
wgcf generate
```
A file named `wgcf-profile.conf` will be created, and this is the configuration we need.

## Step Two: Using Warp+ (Optional)

- To receive a license and use Warp+, you can get a `license_key` through [this](https://t.me/generatewarpplusbot) Telegram bot.
- After receiving the `license_key`, you need to replace it in the `wgcf-account.toml` file.
::: tip Note
You can make this change using `nano` in Linux and `Notepad` or any other software in Windows.
:::
::: details Windows
To use commands on Windows, you need to use `wgcf.exe` instead of `wgcf`.
:::
Then you need to update the configuration information:
```bash
wgcf update
```
Then you need to create a new configuration file:
```bash
wgcf generate
```

## Step Three: Activating Warp on Marzban

### First Method: Using Xray Core

::: warning Note
- This method is only recommended for Xray version 1.8.3 or higher. In older versions, you might encounter Memory Leak issues.
- If your `Xray` version is lower than this version, you can upgrade your `Xray` version using the [Xray-core version change guide](/examples/change-xray-version).
:::

- Go to the Core Setting section in the Marzban panel.
- First, we add an outbound like the example and insert the information from the `wgcf-profile.conf` file into it.

```json
{
  "tag": "warp",
  "protocol": "wireguard",
  "settings": {
    "secretKey": "Your_Secret_Key",
    "DNS": "1.1.1.1",
    "address": ["172.16.0.2/32", "2606:4700:110:8756:9135:af04:3778:40d9/128"],
    "peers": [
      {
        "publicKey": "bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=",
        "endpoint": "engage.cloudflareclient.com:2408"
      }
    ],
    "kernelMode": false
  }
}
```

::: tip Note
If you want all traffic to go through Warp by default, place this Outbound first, and you don't need to do the next step.
:::

### Second Method: Using Wireguard Core

First, you need to install Wireguard prerequisites on your server.

```bash
sudo apt install wireguard-dkms wireguard-tools resolvconf
```
If you're using Ubuntu 24, use the following command to install wireguard:
```bash
sudo apt install wireguard
```
Then you need to add the `Table = off` statement to the Wireguard file like the example.

```conf
[Interface]
PrivateKey = Your_Private_Key
Address = 172.16.0.2/32
Address = 2606:4700:110:8a1a:85ef:da37:b891:8d01/128
DNS = 1.1.1.1
MTU = 1280
Table = off
[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
AllowedIPs = 0.0.0.0/0
AllowedIPs = ::/0
Endpoint = engage.cloudflareclient.com:2408
```

::: warning Note
If you don't add `Table = off`, your access to the server will be cut off, and you won't be able to connect to the server anymore. You'll need to access the server through your datacenter's website and disconnect from `Warp` to be able to establish normal connection again.
:::

- Then rename the file from `wgcf-profile.conf` to `warp.conf`.
- Place the file in the `/etc/wireguard` folder on the server.

```bash
sudo mv wgcf-profile.conf /etc/wireguard/warp.conf
```
- Enable Wireguard with the command below.

```bash
sudo systemctl enable --now wg-quick@warp
```

You can also disable `Warp` with this command:

```bash
sudo systemctl disable --now wg-quick@warp
```

- Go to the Core Setting section in the Marzban panel.
- First, add an outbound like the example.

```json
{
  "tag": "warp",
  "protocol": "freedom",
  "streamSettings": {
    "sockopt": {
      "tcpFastOpen": true,
      "interface": "warp"
    }
  }
}
```

::: tip Note
If you want all traffic to go through Warp by default, place this Outbound first, and you don't need to do the next step.
:::

## Step Four: Routing Settings

First, we add a `rule` in the `routing` section like the example.

```json
{
  "outboundTag": "warp",
  "domain": [],
  "type": "field"
}
```

Now you need to add your desired websites like the example.

```json
{
    "outboundTag": "warp",
    "domain": [
        "geosite:google",
        "openai.com",
        "ai.com",
        "ipinfo.io",
        "iplocation.net",
        "spotify.com"
    ],
    "type": "field"
}
```

Save the changes, and now you can use `Warp`.
::: details Marzban-Node

- If you're using `Warp` with xray core, you don't need to make changes in the node as it's automatically applied.

- If you're using the `Wireguard` core, you need to perform step three, second method on the node as well.
:::
