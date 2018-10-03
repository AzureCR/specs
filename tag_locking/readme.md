# Tag Locking

Customers desire the ability to lock updates to a given tag. 

As images are built and pushed to a registry, deployments depend on a given tag. Customers that follow [unique tagging practices](https://stevelasker.blog/2018/03/01/docker-tagging-best-practices-for-tagging-and-versioning-docker-images/) may want to assure tags aren't updated, increasing the stability of their deployments. 

However, some specially named tags may be require updating, such as the evil named `:latest` tag. 

ACR will support a Tag Locking Policy.

## Master Switch
To enable Tag Locking, users will first enable the policy on a registry.

## Opt In/Out

Administrators will be able to opt in/out given repositories and tags. 

Updatable Tags:

- `:latest`
- `:staging`

> It would be nice to support repo level configurations, where a repo can be opted in/out. 

