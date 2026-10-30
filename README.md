# NGINX proxy for Huawei Cloud OBS

This repository contains configuration files for a NGINX reverse proxy to
access Huawei Cloud Object Storage Service (OBS) using path-style URLs.

## Context

AWS S3 supports two different addressing models for buckets and objects:
**path-style** and **virtual-hosted-style** addressing.

In path-style addressing, the bucket name is included in the URL path:

```plain
https://S3_DOMAIN/BUCKET_NAME/OBJECT_KEY

# Example: Huawei Cloud OBS in the LA-Sao Paulo1 region
https://obs.sa-brazil-1.myhuaweicloud.com/backup-bucket/folder01/file01.txt
```

In virtual-hosted–style addressing, the bucket name is part of the hostname:

```plain
https://BUCKET_NAME.S3_DOMAIN/OBJECT_KEY

# Example: Huawei Cloud OBS in the LA-Sao Paulo1 region
https://backup-bucket.obs.sa-brazil-1.myhuaweicloud.com/folder01/file01.txt
```

In May 2019, AWS announced plans to deprecate path-style addressing, initially
scheduling its end of support for September 30, 2020. However, following
customer feedback, AWS postponed the deprecation, and no new end-of-support
date has been established. For more details, see
[Amazon S3 Path Deprecation Plan – The Rest of the Story](https://aws.amazon.com/blogs/aws/amazon-s3-path-deprecation-plan-the-rest-of-the-story/).

Huawei Cloud OBS, however, has a different policy. For security and compliance
reasons, OBS has prohibited path-style URL access to buckets since
December 30, 2023. This restriction was announced in a public notice published
on November 6, 2023: <https://www.huaweicloud.com/intl/en-us/notice/20231106133906404.html>.

This restriction causes compatibility issues with applications that use
S3-compatible storage but support only path-style addressing. Such
applications cannot access Huawei Cloud OBS directly because OBS requires
virtual-hosted-style addressing.

When an application attempts to access an OBS bucket using a path-style URL,
OBS returns an HTTP 403 error with the error code VirtualHostDomainRequired.
For more details, see the [OBS error code documentation](https://support.huaweicloud.com/intl/en-us/api-obs/obs_errorcode_0014.html).

## Proposed solution

NGINX acts as a reverse proxy, accepting requests that use path-style
addressing and rewriting them to use virtual-hosted-style addressing when
forwarding them to Huawei Cloud OBS.

Note that this solution works only with Signature Version 2 (AuthV2). With
AuthV2, the HTTP Host header is not included in the signature calculation,
allowing NGINX to replace it with the appropriate OBS hostname without
invalidating the request signature.

Huawei Cloud OBS uses an authentication algorithm similar to AuthV2, as
described in the [OBS API Reference](https://support.huaweicloud.com/intl/en-us/api-obs/obs_04_0010.html).
However, AuthV4 authentication algorithm is also supported.

## Procedure

1. Obtain valid TLS certificates for the proxy domain to be used. Below is an
   example using Let's Encrypt and DNS challenge (manual validation, single
   time, no renewal):

    ```sh
    sudo apt install -y certbot

    sudo certbot certonly --manual --preferred-challenges dns -d "obs-proxy.example.com"

    # Certbot generates:
    # /etc/letsencrypt/live/obs-proxy.example.com/
    # ├── fullchain.pem
    # └── privkey.pem
    ```

2. Save the certificate and private key files to the `certs` folder (optional);

3. Make a copy of `.env.example` named `.env` and update the values accordingly;

4. Validate the configuration and run the container:

    ```sh
    docker compose config
    docker compose up -d
    docker compose logs -f nginx
    ```

## Results

Solution validated with:

- Acronis Cyber Protect Cloud Version 26.10 + Acronis Cyber Protect Agent
  for Windows (64-bit) v26.10.43456 (at 2026-10-30)
