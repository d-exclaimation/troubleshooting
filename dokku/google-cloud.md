# Dokku on Google Cloud Platform (Compute Engine)

## Installing

```bash
wget https://raw.githubusercontent.com/dokku/dokku/v0.25.7/bootstrap.sh
sudo DOKKU_TAG=v0.25.7 bash bootstrap.sh
```

**Info**

- Bootstrapping using this with any Ubuntu / Debian flavored linux works fine.

**Problems**

- GCP's **shell** timeout in 30 minutes of idle time.

**Solutions**

- Either keep interacting with the shell while installing or leave it be and check later if dokku was successfully installed.

**Helpful**

- Check all running process on Linux to see if installing is still on going.
- Ran the `dokku` command line to see if set.
- Open the **external** ip on a browser to see if `nginx` has been setup.

## SSH Configuration

[Dokku documentation on SSH](https://dokku.com/docs/deployment/user-management/)

1. Add ssh to virtual machine

_Clean ssh way (require ssh for cloud provider)_

```bash
cat ~/.ssh/id_rsa.pub | ssh root@dokku.me dokku ssh-keys:add KEY_NAME
```

_Manual_

```bash
sudo mkdir ~/.ssh; sudo vim ~/.ssh/id_rsa.pub
```

and copy paste the public key.

2. Add ssh key to dokku

```bash
dokku ssh-keys:add admin ~/.ssh/id_rsa.pub
```

**Info**

- No need to configure ssh on the GCP part as the only thing bad is the ephemeral IP.
- Dokku will handle pushing code to its container orchestration.

## Deploy the app

1. Create app

```bash
# Virtual Machine
dokku apps:create $app
```

2. Install all necessary plugin

```bash
# Virtual Machine
sudo dokku plugin:install $plugin_link
```

3. Set remote on local machine and push to dokku

```bash
# Locally in $app root folder
git remote add dokku dokku@$external_ip:$app
git push dokku main:master
```

**Info**

- Reserving static IP would be best here but not entirely necessary given some situations.
- App deployed are not yet live and ready

## Configuring dokku's docker network

Check app report

```bash
dokku network:report $app
```

Output should match

```bash
Network attach post create:
       Network attach post deploy:
       Network bind all interfaces:          false
       Network computed attach post create:
       Network computed attach post deploy:
       Network computed bind all interfaces: false
       Network computed initial network:
       Network computed tld:
       Network global attach post create:
       Network global attach post deploy:
       Network global bind all interfaces:   false
       Network global initial network:
       Network global tld:
       Network initial network:
       Network static web listener:
       Network tld:
       Network web listeners:                172.17.0.3:80
       #                                         |     |
       # $docker_image_ip -----------------------^     |
       # $port            -----------------------------^
```

**Info**

- Static web listener must be off
- Bind all interfaces must be `false`.
- Web listeners must point to `$docker_image_ip:$port`

**Problems**

- Static web listener are set
- Web listeners point to the incorrect ip and port combination.

**Solutions**

1. Check running Docker images

```bash
sudo docker ps
```

2. Use the container ID and inspect $docker_image_ip

```bash
docker inspect $container_id
```

Should be in `json["NetworkSettings"]["IPAddress"]`.

3. Set the static web listener to ip and port combo

```bash
dokku network:set $app static-web-listener $docker_image_ip:$port
```

4. Remove the static web listener

```bash
dokku network:set $app static-web-listener
```

## Configuring Nginx

1. Change redirect internal `IP.web.1`

```bash
sudo vim /home/dokku/$app/IP.web.1 # 127.0.0.1 -> $docker_image_ip
```

2. Rebuild `$app` nginx.conf

```bash
dokku proxy:build-config $echo
```

3. Remove any domains global and local but keep vhost

```bash
dokku domains:clear $app; dokku domains:clear-global; dokku domains:enable $app;
```

4. Modify nginx.conf to use the `$external_ip` as `server_name`

```bash
sudo vim /home/dokku/$app/nginx.conf
```

```conf
server {
  listen      [::]:80;
  listen      80;

  server_name $external_ip;

  ...
```

5. Restart Nginx

```bash
sudo service nginx restart
```

6. Using domain (Alternative)

Replace the variables

```bash
$external_ip -> $domain
```

or use Dokku CLI

```bash
dokku domains:set $app $domain

# Remove globals
dokku domains:clear-global
```

**Info**

- Will be different for configuration with a domain (Further testing is needing).
- nginx.conf might be resetted using nginx.conf.sigil or other method might be better.

**Helpful**

- Restarting instance might be good

```bash
sudo dokku ps:restart --all
```

## Test server

Go to `$external_ip/$path_name`.

