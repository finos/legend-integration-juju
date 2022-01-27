# Deploying GitLab locally
FINOS Legend charms can be used in conjunction with any private hosted Gitlab. Instructions on installing a self managed Gitlab are available from the [Gitlab site](https://docs.gitlab.com/). The following is an example of a docker-compose file that may be used to deploy Gitlab using a static IP address on your local machine.

An example of a `docker-compose.yml` file to deploy Gitlab is
``` yaml
version: '3.6'
services:
  gitlab:
    image: 'gitlab/gitlab-ee:latest'
    restart: always
    hostname: 'gitlab'
    domainname: 'gitlab.local'
    container_name: 'gitlab'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.local:443'
        gitlab_rails['initial_root_password'] = '${FINOS_GITLAB_PASSWORD:?no gitlab password set}'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
        gitlab_rails['lfs_enabled'] = true
        nginx['listen_port'] = 443
        nginx['ssl_certificate'] = '/etc/gitlab/ssl/gitlab.local.crt'
        nginx['ssl_certificate_key'] = '/etc/gitlab/ssl/gitlab.local.key'
        letsencrypt['enable'] = false
    ports:
      - '443:443'
      - '2224:2224'
    volumes:
      - '${GITLAB_HOME:?no gitlab home set}/config:/etc/gitlab'
      - '${GITLAB_HOME:?no gitlab home set}/logs:/var/log/gitlab'
      - '${GITLAB_HOME:?no gitlab home set}/data:/var/opt/gitlab'
      - './certs:/etc/gitlab/ssl'
    networks:
      legend:
        ipv4_address: 172.18.0.2
networks:
  legend:
    ipam:
      driver: default
      config:
        - subnet: 172.18.0.0/16

```

Prior to deploying Gitlab using this docker compose file two environment variables need to be set

1. `GITLAB_HOME` : This is the folder location under which Gitlab will
   store all its data.
   
2. `FINOS_GITLAB_PASSWORD` : This is the root password for Gitlab.

Also note that this docker compose file sets a static IP address (`172.18.0.2`) for Gitlab because the IP address is used in generating TLS certificates for Gitlab. This IP subnet and address, in the docker compose file, may be changed to suit your own deployment, if necessary. Generation of the TLS certificate can be done using a OpenSSL configuration file `cert.cnf` as shown below

``` 
[ req ]
default_bits                  = 2048
distinguished_name            = req_distinguished_name
req_extensions                = req_ext
[ req_distinguished_name ]
countryName                   = US
stateOrProvinceName           = NY
localityName                  = NY
organizationName              = XX
commonName                    = gitlab.local
[ req_ext ]
subjectAltName                = @alt_names
[alt_names]
DNS.1=gitlab.local
IP.1=172.18.0.2
```

Note that the OpenSSL config file uses the same IP and domain name as in the docker compose file to specify Subject Alternative Names. Using this config file a self-sigend TLS certificate may be generated using the following command line

```
$ mkdir certs
$ openssl req -newkey rsa:2048 -nodes -keyout "certs/gitlab.local.key" -x509 -days 365 -out "certs/gitlab.local.crt" \
	-config "cert.cnf" -extensions req_ext -subj "/C=US/ST=NY/L=NY/O=XX/CN=gitlab.local"
```

If your certificates were successfully generated you should see an ouput like this :

```bash
openssl req -newkey rsa:2048 -nodes -keyout "certs/gitlab.local.key" -x509 -days 365 -out "certs/gitlab.local.crt" \
	-config "cert.cnf" -extensions req_ext -subj "/C=US/ST=NY/L=NY/O=XX/CN=gitlab.local"
Generating a RSA private key
.............+++++
........+++++
writing new private key to 'certs/gitlab.local.key'
-----
openssl x509 -in "certs/gitlab.local.crt" -outform der -out "certs/gitlab.local.der"
```

Notice the generated self signed certificates are placed into the `certs` sub-directory.

Assuming the docker compose file (`docker-compose.yml`) show above is in your current directory and the certificates are in a sub-directory `certs` of the same directory, a Gitlab instance may be launched using the standard command line as in `docker-compose up -d`. Note that for a production deployment you may want to generate certificates using a valid certificate authority.

After exporting the environment variables and launching Gitlab using docker you may see output such as :

```bash
$ export GITLAB_HOME=~/gitlab
$ export FINOS_GITLAB_PASSWORD=mypassword
$ docker-compose up -d
Creating network "legend_legend" with the default driver
Creating gitlab ... done
```

It takes a while for Gitlab to start. You may monitor the status of Gitlab to see if it has started using the command `docker ps`.

If Gitlab is still in the process of starting you may see output such as :

```bash
$ docker ps
CONTAINER ID   IMAGE                     COMMAND             CREATED          STATUS                            PORTS                                                                                              NAMES
f3d3ac6ee794   gitlab/gitlab-ee:latest   "/assets/wrapper"   10 seconds ago   Up 9 seconds (health: starting)   22/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp, 80/tcp, 0.0.0.0:2224->2224/tcp, :::2224->2224/tcp   gitlab
```

Notice the `STATUS` is `health: starting`.

Once Gitlab has successfully started you will see ouput such as :

```bash
$ docker ps
CONTAINER ID   IMAGE                     COMMAND             CREATED         STATUS                   PORTS                                                                                              NAMES
f3d3ac6ee794   gitlab/gitlab-ee:latest   "/assets/wrapper"   4 minutes ago   Up 4 minutes (healthy)   22/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp, 80/tcp, 0.0.0.0:2224->2224/tcp, :::2224->2224/tcp   gitlab
```

Notice the `STATUS` is `healthy`.  Once Gitlab has started you may point your web browser to its local IP address, which is `172.18.0.2` in this tutorial, and you should see the Gitlab login screen. You may login using the password set by you in the `FINOS_GITLAB_PASSWORD` environment variable.

## Create GitLab Access Token
To integrate GitLab with Legend, you'll need to create a GitLab access to token; using the GitLab web UI, access `User Settings` > `Access Tokens` from the menu and fill in the following data:
- *Name:* "Legend Demo"
- *Scopes:* api
- *Expiration date:* Select any date in the future

![Gitlab Access Token](./images/gitlab-access-token.png)

Now click `Create access token` button and obtain your access token from the `Your new access token` textbox. **Save this Access Token to follow the local run tutorial**.
