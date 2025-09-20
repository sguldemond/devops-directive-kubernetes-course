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
I'm running into issues with `peter-evans/create-pull-request` locally using `act`.
I changed my token to a general use token with access to all my repo's.
But still getting this message:
```
[image-ci/update-tags       ]   ❓  ::group::Checking the base repository state
| [command]/usr/bin/git symbolic-ref HEAD --short
| changes
| Working base is branch 'changes'
| [command]/usr/bin/git remote prune origin
| git@github.com: Permission denied (publickey).
| fatal: Could not read from remote repository.
| 
| Please make sure you have the correct access rights
| and the repository exists.
```
The tokens I've been trying match the description of the `peter-evans/create-pull-request` tool,
but it's not working yet. I've tried setting the token directly in de pipeline, but same result.
Possible that this tool is not compatible with `act`, since it creates a GitHub PR from a GitHub pipeline usually.
---
Trying to deploy the demo-app using the `kluctl` GitOps feature.
Running the `kluctl deploy -t staging` works. But when looking at the web UI it looks different then the course video.
Also I'm getting warnings, which seems like one of the issues.
From IngressClass traefik I'm getting:
```
conflict with "helm" using networking.k8s.io/v1. Not updating field '.metadata.labels.app.kubernetes.io/instance' as we lost field ownership
```
and
```
conflict with "helm" using networking.k8s.io/v1. Not updating field '.metadata.labels.helm.sh/chart' as we lost field ownership
```
I should go back to chapter 12 and deploy using kluctl before the GitOps part of chapter 14.
I order to complete `kluctl delete -t staging`, after initializing the delete I have to run:
```
# Remove finalizers from the demo-app KluctlDeployment
kubectl patch kluctldeployment demo-app -n kluctl-gitops -p '{"metadata":{"finalizers":[]}}' --type=merge

# Remove finalizers from the gitops KluctlDeployment
kubectl patch kluctldeployment gitops -n kluctl-gitops -p '{"metadata":{"finalizers":[]}}' --type=merge
```
Then restart `kluctl delete -t staging`. 
---
Running kluctl without the GitOps part seems to work.
Only when running the kluctl delete command, it can't delete all resources.
Seems like its also trying to delete Traefic stuff, which may cause issues since they are owned by K3s.
I might be that Helm took over control of Traefic on the cluser, which is not good I think.
After some time I was able to run the `delete` command succesfully, some process must have ended or timed out.
---
Running the `kluctl deploy` does start a lot, but after running `kluctl validate -t staging`,
I can see not everything was deployed properly.
It looks like the same issues as in chapter 7, where the Middleware definition was not working nicely with existing Traefic setup.
I'll try to recreate the updated chapter 7 setup in the chapter 12 files. 
Deployment is working with Ingress manifests, and validation is succesfull.
Only when navigating to the URLs, I get a 404. Not only for the demo-app for for all Ingress I get 404s.
I'm getting to logs in the Traefic pod from kube-system namespace.
I can't find a pod which might be responsible to returning the 404 message.
Navigating to the IP address of the nodes also gives the same error, which might indicate it's a deeper issue.
After stopping k3s.service on macmini(0) it still returns the 404, same for k3s-agent.service on macmini1.
To check what is running on port 80 I can run `lsof -i :80 | grep LISTEN`, but nothing is showing, neither with `ss` or other simulair programs.
According to the analyses of `tcpdump` by GPT-5 the traffic gets picked up by the CNI (Container Network Interface).
Eventhough k3s is not running anymore, a containerized Loki stack still is.
The packets never touch the servers physical NIC bound to :80, but gets picked up by the CNI bridge.
Stopped and disabled the k3s service and rebooted the server, this should flush the iptables.
After rebooting and starting k3s, the same behavior happens, but now we know it the Loki service.
Maybe it's not the Loki service, the tcpdump program is just picking up on all kinds of traffic over port 80.
I tried to follow tcpdump instead of limiting is to 20 lines and focussed on the physical NIC: `sudo tcpdump -ni enp3s0f0 'tcp port 80' -f`.
My K3s+Debian setup used `nftables` not `iptables`, not sure if this is determined by Debian or K3s, I think Debian.
Flushing the the nftables ruleset (`sudo nft flush ruleset`) removes the 404 response.
I made a backup of the ruleset first and restored it after, which returned the 404 response:
```
sudo nft list ruleset > nft.backup
sudo nft -f nft.backup
```
I flushed the ruleset again, and rebooted the server, to check if K3s restored this "broken" ruleset.
After reboot of the server the 404 response is back, this is beginning to sound like a horror movie.
So some process in K3s repopulates the nftables ruleset.
The solution is probably quite straight forward, but the road there is teaching me a lot about the networking around k3s/k8s.
Issues like this are a gift, a oppurtunity to dive in and figure out what's really going on, instead of hammering around hoping you somehow hit the right switch. 
It might be an issue with Traefic after I installed Traefic again via Helm via the devops-course Task.
I want to try to reset Traefic to the default settings: https://docs.k3s.io/networking/networking-services#traefik-ingress-controller
Just calling `sudo kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik.yaml` wasn't enough.
It's most likely the Traefic pod, because when I delete it, before it's finished recreating, the 404 is not returned upon reload.
Could I make the Traefic pod more verbose, since I don't see the 404 responses logged?
Got macmini.home/whoami working! It was missing `spec.ingressClassName` with value `traefik`.
This was not it! But the other way around, the `spec.ingressClassName` breaks it actually.
I applied a catch all Ingress that would route everything to the whoami service.
Here is the documentation on the ingressclass annotation: https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/#providers-kubernetesIngress-ingressClass
For now I've removed the mention of `spec.ingressClassName` in my custom Ingress manifests.