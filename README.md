## rXg Certbot Tutorial

Do you want free SSL certificates forever? Me, too.

We are going to use *Let's Encrypt* with Certbot and DNS-01 validation to get certificates, and a custom script to write these to the rXg database. We are using DNS-01 rather than HTTP-01 so that this process works if the rXg is in a private network for a lab, and also so we don't have to interrupt the web server or have it serve weird files.

One consequence of using DNS-01 is that your DNS provider needs an API that is known to the Certbot community so there is a plugin for it. Or you can write a plugin for your DNS provider. I am going to use CloudFlare with `rxg.schmit.io` in this tutorial.

### Step 1 - Get API credentials

Get your API key from your DNS provider.

I am using CloudFlare, and they have a community-supported API. This is what my API key looks like: `df5*******************************cee`

### Step 2 - Write the API credentials to disk on the rXg

1. SSH into your rXg and `su -` to root and use the first alpha-only line of the IUI as the password. We will do everything as `root` and from the `/root` directory from now on.
2. You should now be in the directory `/root`; check using `pwd`.
3. Using `vi` (or `nano`, or whatever text editor you choose), create the file `certbot.cred`:

```
# Cloudflare API credentials used by Certbot
dns_cloudflare_email = schmitgreg@gmail.com
dns_cloudflare_api_key = df5*******************************cee
```

4. Restrict the permissions of this file since it contains your API key:

```
$ chmod 600 certbot.cred
```

### Step 3 - Install the `certbot-dns-cloudflare` package

We need to install the `certbot-dns-cloudflare` plugin, and we're using Python2/pip to do so, which is already installed on the rXg. This operation will also have the side effect of updating the already installed Certbot package. The rXg upgrade process could install older versions or remove packages altogether. If you upgrade your rXg later and have issues with Certbot, re-executing this command will likely fix those issues.

1. Type the command `pip install certbot-dns-cloudflare`.

### Step 4 - Run `certbot` for the first time

1. Run `certbot`:

```
$ certbot certonly --dns-cloudflare \
    --dns-cloudflare-credentials ~/certbot.cred \
    --dns-cloudflare-propagation-seconds 60 \
    -d rxg.schmit.io
```

2. You will need to enter an admin email address (I recommend your CloudFlare email), agree to their Terms of Service, and you may see additional prompts; follow them and provide sane responses.
3. Then you should see something like:

```
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for rxg.schmit.io
Waiting 60 seconds for DNS changes to propagate
Waiting for verification...
Cleaning up challenges
```

4. Your private key should be located at `/etc/letsencrypt/live/<FQDN>/privkey.pem` and your host/intermediate certs should be slapped together in `/etc/letsencrypt/live/<FQDN>/fullchain.pem`.

### Step 5 - Write your rXg scripts

If we copied the files to wherever the rXg looks for certificates, that might work, except the rXg is likely to overwrite those files and we don't know when that might happen. Attempting that would be hacky. We need a ruby script that writes the keys to the rXg database. This will ensure stability because the rXg will know to properly reboot the web server when the database is updated.

1. Using your text editor, open `push_certs.rb`, and use this template, but change the `domain` variable to your domain:

```
#!/usr/bin/env ruby

# This script pushes Certbot-generated certificates to the rXg via the API.

# get rails env
require '/space/rxg/console/config/boot_script_environment'

# definitions
domain = 'rxg.schmit.io'
cert_directory = "/etc/letsencrypt/live/#{domain}"
key_file = "#{cert_directory}/privkey.pem"
fullchain_file = "#{cert_directory}/fullchain.pem"
cert_regex = '-----[A-Z ]+-----[\S\s]*?-----[A-Z ]+-----'

# read the files
key = File.read(key_file).strip!
fullchain = File.read(fullchain_file).strip!

# parse out the host/intermediate certs (first cert is host)
host_cert = fullchain[/#{cert_regex}/]
intermediate_cert = fullchain.sub(/#{cert_regex}/, '').strip!

# push to rXg
name = "certbot for #{domain}"
s = SslKeyChain.find_by(name: name)
unless s
  s = SslKeyChain.new(name: name)
end
s.server_key = key
s.intermediate_certificate = intermediate_cert
s.certificate = host_cert
s.note = "Written by Certbot post-hook /root/push_certs.rb"
s.save!
```

2. Adjust the permissions by running `chmod 700 push_certs.rb`.

Now all we need is for the system to regularly run `certbot renew` (and `push_certs.rb` if that was successful). We cannot edit `/etc/crontab` as that file is managed by the rXg software. However, we could try adding a periodic daily script in `/etc/periodic/daily` and hoping that RGNets doesn't sanitize that directory.

3. Edit `/etc/periodic/daily/certbot`:

```
#!/bin/sh

/usr/local/bin/certbot renew --post-hook "/root/push_certs.rb"

exit 0
```

2. Adjust the permissions to ensure root can execute by running `chmod 700 /etc/periodic/daily/certbot`.

### That's it

So this tutorial was written while I am testing it in a live lab environment, and I'll update this if anything goes wrong and I find a solution.
