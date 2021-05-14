# Creating a new container image

## Dependencies

If you would like to help create new container images, there's a few tools
you will need:

- [make](https://www.gnu.org/software/make/)
- [Docker](https://www.docker.com/get-started)
- [buildx](https://docs.docker.com/buildx/working-with-buildx/)
- any other compiler tools used to build and test your code

## Creating a new project

Create a new GitHub project using
[this template](https://github.com/k8s-at-home/template-container-image)
as reference.

If you create this project under [k8s-at-home](https://github.com/k8s-at-home)
then you are ready to start coding. If this is not the case you will need to
create required GitHub secrets as described in the
[template README](https://github.com/k8s-at-home/template-container-image/blob/main/README.md).

The template includes are required workflows we use to build multi-arch containers.
It also contains a
[sync workflow](https://github.com/k8s-at-home/template-container-image/blob/main/.github/workflows/sync_template.yaml)
that will generate pull requests when the template `.github folder` changes.

### Adding your code

The project contains an example Golang program and a Makefile to test it. If your
project uses Golang you can then start extending it. If you use a different
programming languaje you will need to remove the go files and adjust the
Makefile.

### Building and testing

1. `make` will build and test the code
2. `make docker-build` will build the container

### Testing the CI pipeline in GitHub

After you upload your changes to GitHub the CI workflow action should be invoqued
automatically. If it runs sucesfully your container should not be uploaded to
[GHCR](https://github.com/orgs/k8s-at-home/packages?ecosystem=container).
It will be private so you will need to ask a community mainteiner to make it public.

## References

Check the following examples of code projects:

- [dnsmadeeasy-webhook](https://github.com/k8s-at-home/dnsmadeeasy-webhook)
- [dynamic-ip-controller](https://github.com/k8s-at-home/dynamic-ip-controller)
