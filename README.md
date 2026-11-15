# devopsy-template-registry

A private Docker registry ([CNCF
distribution](https://distribution.github.io/distribution/)) on a
[devopsy](https://github.com/hanoii/devopsy-cli) server, behind
[devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik).
It has two users: one that pushes, for CI or a build server, and a
read-only one, for the servers that pull the images. Use it for devopsy's
image mode, where images are built and pushed and each environment
pulls its commit's image.

## Using this template

This template is meant to be copied and changed. Start a project from it
on GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Nothing from this
repository reaches your copy or your servers until you pull its changes
and release them.

## Try it

Point the target at your server in `.devopsy/.env` (gitignored), or in
the environment:

```sh
echo DEVOPSY_SERVER=devopsy@203.0.113.10 >> .devopsy/.env
```

The registry gets `<project>-<environment>.<server's base domain>`, named
after its compose project (`registry-prod`). Each release imports the
base domain from the server's proxy (devopsy-template-traefik) through
the `devopsy.import` label in `compose.yaml`, and fails while no proxy
runs. The release prints what it imported, and `devopsy @prod --debug
imports` shows it later and whether the proxy has changed it since.
Override it per target, or set it empty for no wildcard URL, with
`devopsy @prod --vars set --show PROXY_BASE_DOMAIN`.

```sh
devopsy @prod --release          # runs deploy: https://registry-prod.<server's base domain>
devopsy @prod credentials      # URL, users and passwords for docker login
devopsy @prod credentials pull # the read-only user only
devopsy @prod rotate pull      # a new password for it
devopsy @prod --status         # URLs, users, htpasswd, storage used
devopsy @prod logs -f registry
```

Then, where you push:

```sh
docker login registry-prod.example.com -u devopsy
docker tag myapp registry-prod.example.com/myapp:abc123
docker push registry-prod.example.com/myapp:abc123
```

On the servers that run the images, log in as the `pull` user. It can
pull and list, but not push or delete, so a server holds no credential
that could replace an image.

```sh
docker login registry-prod.example.com -u pull
```

## Secrets

On the first release, the `secrets` prepare step generates
`REGISTRY_PASSWORD`, `REGISTRY_PULL_PASSWORD` and `REGISTRY_HTTP_SECRET`
into the server's `shared/.env`, before the release goes live.

Every `deploy` compares `mnt/auth/htpasswd` with the two users and their
passwords. The users are `REGISTRY_USER` (default `devopsy`) and
`REGISTRY_PULL_USER` (default `pull`). When they differ, `deploy` rewrites
the file (bcrypt, through `httpd`'s `htpasswd`).

`devopsy @prod rotate pull` (or `push`) writes a new random password into
`shared/.env`, deploys, and prints the new login. The old password stops
working at once, so update it wherever that user logs in. To choose a
password yourself, run `devopsy @prod --vars set REGISTRY_PULL_PASSWORD`,
then `devopsy @prod --reload` (`reload: deploy`).

## The read-only user

The registry's htpasswd has no permissions, and every user in it can
push. Traefik makes `pull` read-only. Pulls are `GET` and `HEAD` requests
(manifests, blobs, the catalog). A second router sends every other method
(uploads, tags, deletes) through a basic auth of the push user alone,
before the registry checks the same credentials again.

`deploy` writes that user's htpasswd line into `.env` as
`REGISTRY_PUSH_USERS`, for the router's label. The value is in single
quotes because compose expands `$` in unquoted values. The `pull` user
can still list every repository and tag.

## Through a stop

Both routers name `hold@file`, the hold in the server's Traefik
([Stopped
sites](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/hold.md)).
A pull during a release or a restart waits for the registry instead of
getting a 502 or a 404, and is sent on to the same URL with a 307, which
Docker follows. A browser sends a POST's body again after that 307, but
Docker cannot send an upload's body again, so an upload fails as it
would without the hold. Release the server's Traefik first, since
an older one has no hold and does not join this project's network. To
opt out, drop `hold@file` from the routers' `middlewares` labels.

## Storage

Images live in `shared/mnt/registry`. Back it up, or accept rebuilding
the images. It is the service's data (`devopsy.data=true`). Its
`devopsy.data.version` is the image's major, `REGISTRY_VERSION` (3 by
default), and the image's tag reads the same variable. devopsy never
runs the store on a lower major, so a rollback across a major is
refused. `devopsy @prod --export registry > registry.tar` is a tar of
it, taken with the registry stopped, and `--import registry` puts one
back.

The registry runs as its environment's own uid (`DEVOPSY_UID`, which
devopsy gives each environment on a server and no other environment's
containers share) with a read-only root filesystem and no capabilities.
devopsy gives it the storage directory (the service's
`devopsy.mnt.owner=user` label, from its `user:`), which the deploy user
cannot do. The tools (`htpasswd`, `crane`) run as that uid too, without
capabilities.

## Uploads

Traefik limits how long a request, including its body, may take: 60
seconds by default, which can cut off a large layer pushed over a slow
link.
Raise the limit in the server's Traefik settings (`600s`, or `0` for no
limit) with `devopsy @<server>-traefik --vars set --show
PROXY_READ_TIMEOUT`, then `devopsy @<server>-traefik --reload`.

Cloudflare's Free and Pro plans take a request body of at most 100 MB,
through its proxy or a tunnel, so a larger layer fails to push (413).
Keep the registry on a direct host (DNS only), or its layers small. On a
server with both tunneled and direct sites, the registry names its
entrypoint like any site, `PROXY_ENTRYPOINTS: tunnel` in its target's
`env` with `PROXY_CERTRESOLVER` unset (devopsy-template-traefik's
[Tunneled and direct sites on one
server](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/tunnel.md#tunneled-and-direct-sites-on-one-server)).

## Behind an HTTP proxy

On a server behind an HTTP proxy, Docker gives every container the proxy
settings of the deploy user's `~/.docker/config.json`
([devopsy-cli's servers doc](https://github.com/hanoii/devopsy-cli/blob/main/docs/servers.md#behind-an-http-proxy)).
The registry needs nothing listed there: its healthcheck never uses the
proxy, and `crane` reaches the registry directly while other registries
stay behind the proxy.

## Destroying

`devopsy @prod --destroy` is built into devopsy; the project has no
destroy step. It takes down the containers and volumes, empties `mnt/`
as root, then removes the environment's directory.

## Cleaning up

Pushing a tag again, or deleting it, leaves its old layers behind. List
the tags, delete the ones you no longer need, then collect:

```sh
devopsy @prod images          # every tag: digest, compressed size
devopsy @prod images myapp    # one repository's
devopsy @prod delete myapp:abc123 myapp:def456   # type 'delete' to go on
devopsy @prod delete myapp    # every tag of it: type 'delete all'
devopsy @prod gc --dry-run    # only lists what would go
devopsy @prod gc              # stops the registry, collects, starts it again
devopsy @prod --status        # the storage it uses now
```

These commands run
[crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane)
as the push user, in the `tools` profile's `crane` service. It reaches the
registry on the project's network (`registry:5000`), without going
through Traefik.
`devopsy @prod crane ...` runs any other crane command there, like
`devopsy @prod crane ls registry:5000/myapp`.

`delete` lists the tags first and asks you to type `delete`, or `delete
all` for a whole repository; `--yes` skips the prompt, and is needed
without a terminal. It removes only the tags. An image another tag
points to stays (`images` shows which tags share a digest), and the
repository is still listed, with no tags. Sizes count each tag's layers,
shared or not, so they add up to more than `--status` shows. A server
that ran a deleted tag keeps its copy for rollbacks, but a new release
of that tag fails.

`gc` removes untagged manifests and the layers nothing else uses. An
image pushed only by digest counts as untagged. Older
registry versions deleted the platform images of multi-platform tags this
way; on 3.1.2 a kept tag still pulls on every platform (checked with
`crane export --platform`). The image is
`registry:${REGISTRY_VERSION:-3}`, and each release pulls its fixes, so
check again after an update.

## Without a target

With a local devopsy-template-traefik, the registry is at
`https://registry.localhost` (the config's project name):

```sh
devopsy --release
devopsy credentials
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
