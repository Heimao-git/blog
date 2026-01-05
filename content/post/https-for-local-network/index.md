---
title: 用 Caddy 為內網的服務設定 HTTPS 驗證
description: 不再需要按 "接受風險並繼續"
slug: https-for-local-network
date: 2026-01-05 00:00:00+0800
categories:
    - 筆記
tags:
    - linux
    - 筆記
---

Caddy 是一個自動取得 HTTPS 的反向代理。簡單來說其利用 DNS 的 A/AAAA 紀錄判斷是否擁有該網域以取得憑證。  
但由於我的對內服務並沒有對外網開放，第三方無法判斷 IP 並取得憑證，因此需要另一種驗證方法。

> 本文使用到 Cloudflare DNS 服務，Caddy 為 lucaslorentz/caddy-docker-proxy 版本。

## DNS-01 challenge

DNS-01 驗證方式（challenge）是一種 [AMCE 驗證方式][1]，透過在 DNS 建立暫時的 TXT 紀錄，內容為 `_acme-challenge.<網域>`，並依此判斷是否擁有該網域。  
也因此 Caddy 在自動取得憑證時需要 DNS 的管理權限，[以 Cloudflare 為例][2]，需要具有 `Zone.Zone:Read` 和 `Zone.DNS:Edit` 權限的 API。

## Caddy 擴充

在開始設定前，需要[為 Caddy 安裝擴充][3]，我使用的 Dockerfile 如下：

```Dockerfile
ARG CADDY_VERSION=2.10.0
FROM caddy:${CADDY_VERSION}-builder AS builder

RUN xcaddy build \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2 \
    --with github.com/caddy-dns/cloudflare

FROM caddy:${CADDY_VERSION}-alpine

COPY --from=builder /usr/bin/caddy /usr/bin/caddy

CMD ["caddy", "docker-proxy"]
```

## Cloudflare

除了取得具有 `Zone.Zone:Read` 和 `Zone.DNS:Edit` 權限的 API 之外，也要在 DNS 紀錄中加入 A/AAAA 紀錄。

<!-- prettier-ignore -->
| 類型 | 名稱    | 內容        | Proxy 狀態 |
| ---- | ------- | ----------- | ---------- |
| A    | example | 192.168.x.x | 僅 DNS     |

## 網域設定（Caddyfile）

```Caddyfile
(dns-challenge) {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }
}

example.{env.DOMAIN} {
    import dns-challenge
    reverse_proxy 192.168.x.x:80
}
```

其中 `{env.CF_API_TOKEN}` 為 API Token，`{env.DOMAIN}` 為網域。

至於我使用的 lucaslorentz/caddy-docker-proxy 版本，則是在 docker compose 中 labels 加入：

```yaml
caddy: (dns-challenge)
caddy.tls.dns: cloudflare ${CF_API_TOKEN}
caddy_1: example.${DOMAIN}
caddy_1.reverse_proxy: 192.168.x.x:80
caddy_1.import: dns-challenge
```

## 除錯

https://github.com/caddy-dns/cloudflare?tab=readme-ov-file#troubleshooting  
我有遇到 `timed out waiting for record to fully propagate` 錯誤，但是在 Cloudflare 上有正常產生 TXT 紀錄，也照建議在 Caddyfile 的 tls 區塊加入 `resolvers 1.1.1.1`。  
最後是等過一陣子就成功驗證了，我猜是在 Cloudflare 那邊新增紀錄後 letsencrypt 那邊 DNS 還沒更新，因此要等一段時間。

## 參考

- [Setting Up a Secure Local Network with Caddy, Cloudflare DNS, and Let’s Encrypt](https://webenclave.com/2024/11/07/setting-up-a-secure-local-network-with-caddy-cloudflare-dns-and-lets-encrypt/)
- [How to use DNS provider modules in Caddy 2](https://caddy.community/t/how-to-use-dns-provider-modules-in-caddy-2/8148)
- [Challenge Types - Let's Encrypt][1]
- https://github.com/caddy-dns/acmedns?tab=readme-ov-file#using-acme-dns-to-obtain-https-certificates-with-caddy
- https://github.com/caddy-dns/cloudflare?tab=readme-ov-file#configuration
- https://github.com/lucaslorentz/caddy-docker-proxy?tab=readme-ov-file#custom-images

[1]: https://letsencrypt.org/docs/challenge-types#dns-01-challenge
[2]: https://github.com/caddy-dns/cloudflare?tab=readme-ov-file#configuration
[3]: https://github.com/lucaslorentz/caddy-docker-proxy?tab=readme-ov-file#custom-images
