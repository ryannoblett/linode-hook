# linode-hook
linode-hook is a shell script to integrate the dehydrated ACME client (https://github.com/lukas2511/dehydrated) with Linode DNS servers for dns-01 challenges. 

The purpose of this project is to automate the integration of the Let's Encrypt ACME dns-01 challenge with Linode DNS hosting, using the standard POSIX shell and minimal external dependencies.

In my search to accomplish dns-01 challenge automation on Linode DNS, I found only one solution online - a hook for dehydrated that was written entirely in GoLang. I found this difficult to set up and support on many systems, so I decided to write my own, with simplicity and compatibility in mind.

# Reqiurements
- dehydrated ACME client script (https://github.com/lukas2511/dehydrated)
- POSIX-compatible shell (tested on bash and ash)
- standard utilities (tested on GNU utilities)
- openssl compatible binary (tested with openssl, should work with libressl)
- jq (https://stedolan.github.io/jq/)
- ssh client
- rsync
- Linode account with DNS zones and an API key configured

# Limitations
- Challenges are performed serially and must be completed before the script can continue. Linode only updates its DNS servers every 15 minutes, so even with a 5 minute TTL on your zones, it takes a very long time to get through multiple challenges. Ideas are welcome for ways to parallelize this portion (doing so would reduce total challenge time to ~15 minutes instead of ~15 minutes per challenge), but I'm not sure it's possible due to the design of dehydrated.

# Configuration
First, configure dehydrated per the documentation. This hook script is designed under the assumption that dehydrated will run as an unprivileged user (preferrably its own user), and that this user will exist on all target servers for deployment. It requires you to configure SSH key authentication between the host that this script runs on and the targets.

Next, create the /etc/ssl/letsencrypt directory on all target servers, and set the dehydrated user as the owner of it. Then, create subdirectories on each server matching the common name of each certificate you want deployed to that server. For example, if you want site1.com on server1, and site2.com on server2, you would create a /etc/ssl/letsencrypt/site1.com directory on server1, and a /etc/ssl/letsencrypt/site2.com directory on server2.

Once you have that set up, download the dehydrated script and configure it with the following options in the config file:
- CHALLENGETYPE="dns-01"
- HOOK="linode-hook"
You must leave HOOK_CHAIN="no" (this is the default and does not need to be defined explicitly). Configure the other options as necessary to suit your environment. 

Create your domains.txt file and put your domain names in it per the documentation.

Download linode-hook and place the script in the same location as dehydrated. All the configuration for the linode-hook script is done at the top of the script, and is mostly self-explanatory. Here's a quick summary.

- API_KEY - this should point at a file which contains your Linode API key. By default it's ".apikey" in the current working directory.
- HOSTS - this is a list of hostnames where you want your certificates to be deployed upon successful issue.

Finally, configure your services to use the deployed certificates, and implement a strategy to reload the configuration when new certificates are deployed. The simplest way is to restart on a regular interval. A more exact approach can be seen in the certcheck script (requires an openssl compatible binary).
