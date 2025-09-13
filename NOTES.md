Traefic IngressRoute not responding to calls from configured host is seems (devops-demo.macmini.home).
I reconfigured the catch all Ingress controller of other apps.
---
I'm not using the containers from my local registry I think.
It says docker.io/sidpalas/... in the describe of the go-lang pod.
---
Preparing to setup CI/CD. Looks like I have to change the image-ci.yml a bit.
No need to use buildx and the `build-container-image-multi-arch` task.
Instead I'll need a build and push to registry step that is equal for all services.
I'll have to fork this course on GitHub in order to get my own GitHub Actions.

Because I'm using my local container registry, running the pipeline from GitHub wouldn't work.
I'm testing out `act`, to run the `image-ci.yml` locally.
Failing to push to registry, with known error:
```
Get "https://registry.macmini.home/v2/": tls: failed to verify certificate: x509: certificate is valid for 2dd6172ca5a0b91bb83ab139d79ad9a6.56b7a60df24855024af65cb1c350f3d0.traefik.default, not registry.macmini.home
```
This is not a `act` issue, also occurs when I run the build+push task manually.
I have to configure my docker locally to tell Docker daemon to skip https on this registry:
Edit `/etc/docker/daemon.json`:
```
{
  "insecure-registries": ["registry.macmini.home"]
}
```
Restart docker: `systemctl restart docker`

This also works for `act` appearently, that means it's using the systems Docker daemon.
---
To create PR from pipeline a token is needed.
I opted for a fine-graned token, which limits access to the repo.
I order to pass it, created a `secrets.env` with the token, and pass it to act using `--secret-file`
