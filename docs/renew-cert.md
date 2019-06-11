# Renew certificates

As of the moment of writing this we have 3 tls enabled websites.

1. stryker-mutator.io
1. dashboard.stryker-mutator.io
1. badge.stryker-mutator.io

All 3 are protected using [let's encrypt](https://letsencrypt.org/). 
The root domain (stryker-mutator.io) is automatically renewed. Others are hosted on azure and need our attention from time to time.

## One time setup

*Note: these instructions only work on linux or mac. For windows users, you can use the [Windows subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

Install the `certbot`(https://certbot.eff.org/).
Install the `certbot-cloudflare-plugin` (https://certbot-dns-cloudflare.readthedocs.io/en/stable/)
Install the azure-cli tooling: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest

Create your `~/.secrets/certbot/cloudflare.ini` file:

```
# Cloudflare API credentials used by Certbot
dns_cloudflare_email = cloudflare@example.com
dns_cloudflare_api_key = 0123456789abcdef0123456789abcdef01234567
```

Run: 

```
az login
az account set -s pdc_stryker_prod01
```

## Renew

Use this script to renew:

```
set -e
PASSWORD=`openssl rand -base64 16`

certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini -d dashboard.stryker-mutator.io -d badge.stryker-mutator.io
cd /etc/letsencrypt/archive/dashboard.stryker-mutator.io/
openssl pkcs12 -export -out stryker.pfx -inkey privkey1.pem -in cert1.pem -certfile chain1.pem -password pass:$PASSWORD
az webapp config ssl upload --certificate-file ./stryker.pfx --certificate-password $PASSWORD --name stryker-mutator-badge --resource-group stryker-mutator
az webapp config ssl upload --certificate-file ./stryker.pfx --certificate-password $PASSWORD --name stryker-badge --resource-group strykermutator-badge-website

THUMBPRINT=`az webapp config ssl list --resource-group stryker-mutator --query [0].thumbprint | tr -d '"'`

az webapp config ssl bind --certificate-thumbprint $THUMBPRINT --ssl-type SNI --name stryker-mutator-badge --resource-group stryker-mutator
az webapp config ssl bind --certificate-thumbprint $THUMBPRINT --ssl-type SNI --name stryker-badge --resource-group strykermutator-badge-website
```