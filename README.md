# linode-hook
`linode-hook` is a shell script to integrate the [dehydrated ACME client](https://github.com/lukas2511/dehydrated) with Linode DNS servers for dns-01 challenges. 

The purpose of this project is to automate the integration of the Let's Encrypt ACME dns-01 challenge with Linode DNS hosting, using the standard POSIX shell and minimal external dependencies.

In my search to accomplish dns-01 challenge automation on Linode DNS, I found only one solution online - a hook for dehydrated that was written entirely in GoLang. I found this difficult to set up and support on many systems, so I decided to write my own, with simplicity and compatibility in mind.

## Reqiurements
- [dehydrated ACME client script](https://github.com/lukas2511/dehydrated)
- POSIX-compatible shell (tested on bash and ash)
- grep
- dig
- openssl compatible binary (tested with openssl, should work with libressl)
- curl
- [jq](https://stedolan.github.io/jq/)
- ssh client
- rsync
- Linode account with DNS zones and an API key configured

## Limitations
- Challenges are performed serially and must be completed before the script can continue. Linode only updates its DNS servers every 15 minutes, so even with a 5 minute TTL on your zones, it takes a very long time to get through multiple challenges. Ideas are welcome for ways to parallelize this portion (doing so would reduce total challenge time to ~15 minutes instead of ~15 minutes per challenge).

## Configuration
This hook script is designed under the assumption that dehydrated will run as an unprivileged user (preferrably its own user), and that this same user will exist on all target servers for deployment. It requires you to configure SSH key authentication between the host that this script runs on and the deployment targets.

First, download the dehydrated script and [configure dehydrated per the documentation](https://github.com/lukas2511/dehydrated/blob/master/README.md#getting-started). Configure dehydrated with the following options in the config file:
- `CHALLENGETYPE="dns-01"`
- `HOOK="linode-hook"`

`HOOK_CHAIN="yes"` is now supported and recommended, as it's considerably faster with multiple domains in a certificate. Configure the other options as necessary to suit your environment. 

Create your `domains.txt` file and put your domain names in it.

Download `linode-hook` and place the script in the same location as dehydrated. All the configuration for the `linode-hook` script is done at the top of the script, and is mostly self-explanatory. Here's a quick summary.

- `MY_API_KEY` - this should point at a file which contains your Linode API key. By default it's `.apikey` in the script's working directory.
- `HOSTS` - this is a list of hostnames where you want your certificates to be deployed upon successful issue.
- `DEPLOYPATH` - this is the base location where certificates will be deployed on the destination servers.

Next, create the deploy path directory (defaults to `/etc/ssl/letsencrypt`) on all target servers, and set the dehydrated user as the owner of it. Then, create subdirectories on each server matching the common name of each certificate you want deployed to that server. For example, if you want `site1.com` on `server1`, and `site2.com` on `server2`, you would create a `/etc/ssl/letsencrypt/site1.com` directory on `server1`, and a `/etc/ssl/letsencrypt/site2.com` directory on `server2`.

Create an ssh keypair for the user running the dehydrated script, and add the public key to the `authorized_keys` file for the same user on all the destination hosts.

Run dehydrated for the first time and register with Let's Encrypt.
- `./dehydrated --register --accept-license`

Run dehydrated for the second time to start requesting certificates. You will have to accept SSH host keys during the first deploy step if you haven't already done so.
- `./dehydrated --cron`

Configure your services to use the deployed certificates, and implement a strategy to reload the configuration when new certificates are deployed. The simplest way is to restart on a regular interval. A more exact approach can be seen in the `certcheck` script (requires an openssl compatible binary on the destination hosts).

Finally, add `dehydrated --cron` to the dehydrated user's crontab on a regular basis. Daily is probably best, but the interval is up to you. Keep in mind that it can take a long time to execute if you add a significant number of domains to the domains.txt file.

## Certificate Checker
I have included certcheck, which is a sample script you can add to the crontabs of the servers where the certificates will be deployed. 
As written, it will look up the names of certificates deployed to the server from linode-hook, check if there are newer certificates than the ones currently being used by the web server at each given URL, and if so, it will reload the webserver (in the example, apache2) to apply the new certificate. 

This will need to be modified slightly to match the web server or other services you are running, but it's the last piece needed to have the central deployment fully automated. Without it, or an equivalent tool, the service will need to be reloaded manually whenever the certificates expire.

## Development Goals
- Replace jq with a pure shell json interpreter (to reduce external dependencies)
