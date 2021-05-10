# CertDeploy – A deploy hook script for Certbot
CertDeploy is a "deploy hook" script for the [Certbot ACME client](https://certbot.eff.org/) written in Bash.

It can be used with the `--deploy-hook` option of Certbot to easily deploy (or better: "install/move") your previously obtained X.509 certificates from Certbot's default location to a **desired directory structure with your custom UNIX file and directory permissions and custom user/group ownership**.

CertDeploy has been developed for people like me who want their certificate files residing in a **different and controllable directory structure** instead of the default directory (`/etc/letsencrypt/live/foo`) provided by Certbot. Many services do not run with `root` privileges, so they need to have read permissions for the X.509 certificate file(s) and the corresponding private key file, which means that one need to adjust the UNIX file permissions and user/group ownership anyway.

CertDeploy is a **different approach** than just changing the UNIX permission modes of the files residing in the directory provided by Certbot and gives you the opportunity to create your own directory structure for your X.509 certificates. If you don't like this approach, it's okay.

## How to install CertDeploy?
Since CertDeploy is just a Bash script, you don't really have to "install" something here. Just place the script in the `/usr/local/sbin` directory and it should by fine. If `/usr/local/sbin` is already in your `$PATH`, you can call it from the command-line by typing `certdeploy`, otherwise you need to use the full path (or you have to add `/usr/local/sbin` to your `$PATH`).

Since you will be using an absolute path in Certbot's `--deploy-hook` option anyway, you can put CertDeploy to any location you want. I like it when it resides in `/usr/local/sbin`, because that's the place where I put all my custom scripts which are **intended** to get executed by `root` only (therefore `sbin`, not `bin`).

Make sure that no unprivileged user has write permissions on `/usr/local/sbin` or the `certdeploy` script, because CertDeploy is usually executed with `root` privileges by the Certbot ACME client!

## How to use CertDeploy from the command-line?
The only two required command-line arguments are the source and the target directory path:

~~~bash
certdeploy [OPTIONS] source_directory target_directory
certdeploy [OPTIONS] /etc/letsencrypt/live/foo_bar.example.org /etc/certdeploy/foo/bar.example.org
~~~

The **source directory** is usually the `/etc/letsencrypt/live/foo_bar.example.org` directory provided by Certbot in which your newly issued (or renewed) certificate files reside. The **target directory** is the path to the custom directory in which the certificate files shall be copied into by CertDeploy.

The following options let you change the UNIX file permission modes of the target directory and/or the certificate files within your **target directory**. You also can change the name of the individual certificate files **for the target directory** (the private key, the intermediate, the certificate or the certificate chain file):

* `[-m mode]` **(default: `0600`)**:  
Mode for target certificate files (octal notation, 3-4 digits)

* `[-o owner]` **(default: `$(id -u)`)**:  
User ownership for certificate files in target directory

* `[-g group]` **(default: `$(id -g)`)**:  
Group ownership for certificate files in target directory

* `[-M mode]` **(default: `0755`)**:  
Mode for target directory (octal notation, 3-4 digits)

* `[-O owner]` **(default: `$(id -u)`)**:  
User ownership for target directory

* `[-G group]` **(default: `$(id -g)`)**:  
Group ownership for target directory

* `[-K filename]` **(default: `confidential.pem`)**:  
Filename for private key in target directory

* `[-I filename]` **(default: `intermediate.pem`)**:  
Filename for intermediate in target directory

* `[-C filename]` **(default: `certificate_only.pem`)**:  
Filename for certificate in target directory

* `[-F filename]` **(default: `certificate_full.pem`)**:  
Filename for certificate chain in target directory

**Notice:** CertDeploy assumes that the certificate files **in the source directory** are named by the following, which is the default for the files provided by Certbot. Those filenames are hardcoded into CertDeploy:

* `privkey.pem` for the RSA/ECDSA private key
* `chain.pem` for the X.509 intermediate certificate
* `cert.pem` for the X.509 certificate (without intermediate)
* `fullchain.pem` for the X.509 certificate (with intermediate)

But since CertDeploy is just a simple Bash script, you can edit it if you want to use this script not in conjunction with certificate files provided by Certbot. *Yes, you can!*

## How to use CertDeploy in conjunction with Certbot?
In this example, we will request a new X.509 certificate with Certbot intended to use on a Mumble server installed on a Debian Buster GNU/Linux machine. The Mumble server daemon runs as user `mumble-server` and needs to have at least read permissions for the certificate file(s) and their corresponding private key:

~~~ini
; /etc/mumble-server.ini
sslCert=/etc/certdeploy/mumble/voip.example.org/certificate_full.pem
sslKey=/etc/certdeploy/mumble/voip.example.org/confidential.pem
~~~

It is sufficient to use UNIX permissions `0600` (default) and user ownership `mumble-server` to achieve this. Since Certbot is running as `root` and because we omit the `-g` option of CertDeploy, the group ownership of the certificate files will become the default `$(id -g)` (which will be substituted to the primary group of `root` in this case).

OK, just request a new staging (test) certificate from Certbot with the `certonly` subcommand and provide the `--deploy-hook` option as follows. (You may need to adjust your `--webroot-path` in which the `.well-known/acme-challenge` directory for your domains is located. I have this directory globally located at `/var/www/.well-known/acme-challenge` for **all** my hostnames to make things easier.)

**Notice:** If I have not mentioned it yet:  
*You need to have a dedicated webserver installed for this step!*

~~~bash
certbot --staging certonly --cert-name mumble_voip.example.org -d voip.example.org --webroot --webroot-path /var/www --deploy-hook '/usr/local/sbin/certdeploy -o mumble-server $RENEWED_LINEAGE /etc/certdeploy/mumble/voip.example.org'
~~~

The first thing you might notice here is the use of the `$RENEWED_LINEAGE` variable. This is beside of `$RENEWED_DOMAINS` one of two variables you can use in conjunction with Certbot's `--deploy-hook` option (see [Certbot docs for more information](https://certbot.eff.org/docs/using.html?highlight=renewed_lineage)).

So instead of typing the absolute path to the source directory in `/etc/letsencrypt/live`, you can just use the `$RENEWED_LINEAGE` variable which is substituted **by Certbot** and points to the `live` directory of your new (or renewed) certificate files. After you issued the command above and got the certificate, you should see some information printed by the `stat` command of CertDeploy inside the output of the Certbot command:

~~~
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator webroot, Installer None
Obtaining a new certificate
Running deploy-hook command: /usr/local/sbin/certdeploy -o mumble-server $RENEWED_LINEAGE /etc/certdeploy/mumble/voip.example.org
Output from certdeploy:
[755:drwxr-xr-x] [root:root] => /etc/certdeploy/mumble/voip.example.org
[600:-rw-------] [mumble-server:root] => /etc/certdeploy/mumble/voip.example.org/certificate_full.pem
[600:-rw-------] [mumble-server:root] => /etc/certdeploy/mumble/voip.example.org/certificate_only.pem
[600:-rw-------] [mumble-server:root] => /etc/certdeploy/mumble/voip.example.org/confidential.pem
[600:-rw-------] [mumble-server:root] => /etc/certdeploy/mumble/voip.example.org/intermediate.pem


IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/mumble_voip.example.org/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/mumble_voip.example.org/privkey.pem
   Your cert will expire on 2021-06-09. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
~~~

Since your new certificate has been successfully issued, you will find a new file called `mumble_voip.example.org.conf` residing in `/etc/letsencrypt/renewal`. This is Certbot's directory in which the information for the automated renewal process is kept. If you `cat` the file, you will see your deploy hook command which will be executed again if Certbot automatically renews your expiring certificate(s) after (usually) 60 days:

~~~
# renew_before_expiry = 30 days
version = 0.31.0
archive_dir = /etc/letsencrypt/archive/mumble_voip.example.org
cert = /etc/letsencrypt/live/mumble_voip.example.org/cert.pem
privkey = /etc/letsencrypt/live/mumble_voip.example.org/privkey.pem
chain = /etc/letsencrypt/live/mumble_voip.example.org/chain.pem
fullchain = /etc/letsencrypt/live/mumble_voip.example.org/fullchain.pem

# Options used in the renewal process
[renewalparams]
account = <hash>
renew_hook = /usr/local/sbin/certdeploy -o mumble-server $RENEWED_LINEAGE /etc/certdeploy/mumble/voip.example.org
server = https://acme-staging-v02.api.letsencrypt.org/directory
authenticator = webroot
webroot_path = /var/www,
[[webroot_map]]
~~~

That's it! But there is one essential thing **we forgot to mention here:** *You may need to reload or restart your service/daemon so that he knows about the new certificate files!*

No problem! You can **combine multiple commands** with `&&` in Certbot's `--deploy-hook` option like in a shell. We check (for perfectionism) if the service is even running right now with `systemctl is-active mumble-server` and then restart the Mumble server daemon with `systemctl restart mumble-server`:

~~~bash
certbot --staging certonly --cert-name mumble_voip.example.org -d voip.example.org --webroot --webroot-path /var/www --deploy-hook '/usr/local/sbin/certdeploy -o mumble-server $RENEWED_LINEAGE /etc/certdeploy/mumble/voip.example.org && systemctl is-active mumble-server && systemctl restart mumble-server'
~~~

OK. Since (hopefully) everything works as expected right now, you can delete your staging (test) certificate with all the related configuration files by executing Certbot with the `delete` subcommand and the `--cert-name` option:

~~~bash
certbot delete --cert-name mumble_voip.example.org
~~~

You can now repeat the steps in this section **without using the `--staging` flag** of Certbot so that you get a real/valid X.509 certificate issued by the Let's Encrypt Certification Authority (CA). Have fun!