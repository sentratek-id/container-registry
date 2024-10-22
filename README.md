# A simple registry container

This simple registry container is meant to be run locally.
It is also meant to be accessed locally through `localhost:5000`.
To use it, simply run:

```shell
docker compose up -d
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
and you want it to be pushed to the registry.
Then you can delete the local image, and later pull again from the registry.
You can run:

```shell
docker compose up -d
docker tag project_docker localhost:5000/project_docker
docker push localhost:5000/project_docker
```

Then you delete the local image:

```shell
# deletes the local image and the retagged image
docker rmi project_docker localhost:5000/project_docker
```

Later you can pull it again from the registry:

```shell
docker pull localhost:5000/project_docker
docker images
```

That's all!

