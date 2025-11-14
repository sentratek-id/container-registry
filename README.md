# A simple registry container

This is a simple registry container configuration meant to self-host your docker images.
Useful if your docker image is bigger than what is available on your registry's free plan, and you have a spare server to use instead.
Configuration for local use and internet use (via a nginx reverse proxy) is available.

## Use behind an nginx reverse proxy

You would have to configure your nginx to use reverse proxy.

First you should have a TLS certificate solution.
Secondly, you would need to set a `htpasswd` file to hold your credentials.
To obtain a htpasswd file, you can run:[^basicauth]

```shell
docker run \
  --entrypoint htpasswd \
  httpd:2 -Bbn testuser testpassword > datadir/htpasswd
```

Replace the `testuser` and `testpassword` to your desired username and password.

[^basicauth]: Read [distribution documentation](https://distribution.github.io/distribution/about/deploying/#native-basic-auth)

Then your nginx would have to be configured into something like this:

```nginx
# Redirect http traffics to https
server {
    listen 80;
    listen [::]:80;
    server_name docker.domain.tld; # replace with your actual (sub)domain
    return 301 https://$server_name$request_uri;
}

# Handle https connections, terminate TLS and pass requests to our
# registry
server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;
    server_name docker.domain.tld; # replace with your actual (sub)domain

    # Certificate parameters, eg. let's encrypt
    ssl_certificate         /etc/letsencrypt/live/docker.domain.tld/cert.pem;
    ssl_certificate_key     /etc/letsencrypt/live/docker.domain.tld/key.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    
    # docker images can be quite big
    client_max_body_size 5G;

    ssl_verify_client on;
    location / {
        # Our registry docker container runs on port 5000
        proxy_pass http://localhost:5000;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

This goes without saying that you should replace `domain.tld` with any domain you own.
It also follows that your server uses nginx as its web server.
For other web servers, please adjust accordingly, consult to their respective documentations as needed.

Then you can fire up the container:

```shell
docker compose up -d registry
```

If everything works as intended, you should be able to login:

```shell
docker login docker.domain.tld
```

Enter your username and password, and then you can use the registry!

To push image to your registry, you would have to tag it:
```shell
docker tag <image_name> docker.domain.tld/<repository_name>
docker push docker.domain.tld/<repository_name>
```

To pull from your registry:

```shell
docker pull docker.domain.tld/<repository_name>
```

This container will store data inside `datadir/data`, and authentication in `datadir/htpasswd`.
Therefore any images pushed within sould persist after the container is turned off.

## Local usage

This simple registry container is meant to be run locally.
It is also meant to be accessed locally through `localhost:5000`.
To use it, simply run:

```shell
docker compose -f docker-compose.local.yml up -d registry
```

Then you can tag your image and push it to this registry:

```shell
docker tag <image_name> localhost:5000/<repository_name>
docker push localhost:5000/<repository_name>
```

> Replace `<image_name>` and `<repository_name>` according to your project requirements.
> Note that you can also use the same name for `<image_name>` and `<repository_name>`.

After pushing the docker image, you can use it by pulling from this repository:

```shell
docker pull localhost:5000/<repository_name>
```

This container will store data inside `datadir/data`, so that images pushed within
can persist after the container is turned off.

## Use case

For example, you have the following docker image: `project_docker`
and you want it to be pushed to your own registry.
Then you can delete the local image, and later pull again from the registry.

> This setup works for both local deployment and via a reverse proxy.
> For local deployment, simply replace docker.domain.tld with localhost:5000
> and for deployment via a reverse proxy, change docker.domain.tld to your domain.

You can run:

```shell
docker compose up -d registry
docker tag project_docker docker.domain.tld/project_docker
docker push docker.domain.tld/project_docker
```

Then you delete the local image:

```shell
# deletes the local image and the retagged image
docker rmi project_docker docker.domain.tld/project_docker
```

Later you can pull it again from the registry:

```shell
docker pull docker.domain.tld/project_docker
docker images
```

That's all!

**Reading materials:**

- [Deploy a registry server](https://distribution.github.io/distribution/about/deploying/)
- [How to Use Your Own Registry](https://www.docker.com/blog/how-to-use-your-own-registry-2/)
- [Self hosted Docker registry](https://blog.aawadia.dev/2022/11/02/docker-registry-ci/)

