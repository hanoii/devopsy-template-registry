# devopsy-template-registry

A private Docker registry ([CNCF
distribution](https://distribution.github.io/distribution/)) on a
[devopsy](https://github.com/hanoii/devopsy-cli) server, behind
[devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik).
It has two users: one that pushes, for CI or a build server, and a
read-only one, for the servers that pull the images. Use it for devopsy's
image mode: build and push images from CI or a build server, and have
each environment pull its commit's image.

## Using this template

This template is meant to be copied and changed. Start a project from it
on GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Your copy and your
servers change only when you release.

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
runs. The release says what it imported, and `devopsy @prod --debug
imports` shows it later, with whether the proxy has changed it since. To
override it per target, or set it empty for no URL: `devopsy @prod --vars
set --show PROXY_BASE_DOMAIN`.

```sh
devopsy @prod --release          # runs deploy: https://registry-prod.<server's base domain>
devopsy @prod credentials      # URL, users and passwords for docker login
devopsy @prod credentials pull # the read-only user only
devopsy @prod rotate pull      # a new password for it
devopsy @prod --status         # URL, users, htpasswd, storage used
devopsy @prod logs -f registry
```

Then, where you push (CI, a build server):

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

The first release generates `REGISTRY_PASSWORD`, `REGISTRY_PULL_PASSWORD`
and `REGISTRY_HTTP_SECRET` into the server's `shared/.env` with
`secrets`, its prepare step, before it goes live. Every `deploy` checks
`mnt/auth/htpasswd` against the two users, `REGISTRY_USER` (default
`devopsy`) and `REGISTRY_PULL_USER` (default `pull`), and their
passwords, and rewrites it (bcrypt, through `httpd`'s `htpasswd`) when
they differ.

`devopsy @prod rotate pull` (or `push`) writes a new random password into
`shared/.env`, deploys, and prints the new login. The old password stops
working at once, so update it wherever that user logs in. To choose a
password yourself, run `devopsy @prod --vars set REGISTRY_PULL_PASSWORD`,
then `devopsy @prod --reload` (`reload: deploy`).

## The read-only user

The registry's htpasswd has no permissions: every user in it can push.
Traefik makes `pull` read-only. Pulls are `GET` and `HEAD` requests
(manifests, blobs, the catalog), and a second router sends every other
method (uploads, tags, deletes) through a basic auth of the push user
alone, before the registry checks the same credentials again. `deploy`
writes that user's htpasswd line into `.env` as `REGISTRY_PUSH_USERS`,
for the router's label, in single quotes, since compose expands `$` in
unquoted values. The `pull` user can still list every repository and tag.

## Through a stop

Both routers name `hold@file`, the server's Traefik's hold ([Stopped
sites](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/hold.md)).
A pull during a release or a restart waits for the registry instead of
getting a 502 or a 404, and is sent on to the same URL with a 307, which
Docker follows. An upload's body cannot be sent again, so an upload fails
as it would have. The hold needs a Traefik that has it, so release the
server's Traefik first. To opt out, drop `hold@file` from the routers'
`middlewares` labels.

## Storage

Images live in `shared/mnt/registry`: back it up, or accept rebuilding
the images. It is the service's data (`devopsy.data=true`), and its
`devopsy.data.version` is the image's major, `REGISTRY_VERSION` (3 by
default), which the image's tag reads too. devopsy never runs the store
on a lower major, so a rollback across a major is refused. `devopsy @prod
--export registry > registry.tar` is a tar of it, taken with the registry
stopped, and `--import registry` puts one back.

The registry runs as UID 10001 with a read-only root filesystem and no
capabilities. devopsy gives it the storage directory, which the deploy
user cannot do (the service's `devopsy.mnt.owner=user` label, from its
`user:`). The tools (`htpasswd`, `crane`) also run as 10001 without
capabilities.

## Uploads

Traefik limits how long a request may take, body included, to 60 seconds
by default, which can cut off a large layer pushed over a slow link.
Raise the limit in the server's Traefik settings (`600s`, or `0` for no
limit) with `devopsy @<server>-traefik --vars set --show
PROXY_READ_TIMEOUT`, then `devopsy @<server>-traefik --reload`.

Cloudflare's Free and Pro plans take a request body of at most 100 MB,
through its proxy or a tunnel, so a larger layer fails to push (413).
Keep the registry on a direct host (DNS only), or its layers small. On a
server with both tunneled and direct sites, the registry names its
entrypoint like any site, `PROXY_ENTRYPOINTS: tunnel` in its target's
`env` with `PROXY_CERTRESOLVER` unset (devopsy-template-traefik's [Behind
a Cloudflare
Tunnel](https://github.com/hanoii/devopsy-template-traefik/blob/main/docs/tunnel.md)).

## Destroying

`devopsy @prod --destroy` is devopsy's alone; the project has no destroy
step. It takes down the containers and volumes, empties `mnt/` as root,
then removes the environment's directory.

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
in the `tools` profile's `crane` service, as the push user, on the
project's network (`registry:5000`), without going through Traefik.
`devopsy @prod crane ...` runs any other crane command there, like
`devopsy @prod crane ls registry:5000/myapp`.

`delete` lists the tags first and asks you to type `delete`, or `delete
all` for a whole repository; `--yes` skips the prompt, and is needed
without a terminal. It removes only the tags. An image another tag points
to stays (`images` shows which tags share a digest), and the repository
stays listed, with no tags. Sizes count each tag's layers, shared or not,
so they add up to more than `--status` shows. A server that ran a deleted
tag keeps its copy for rollbacks, but a new release of that tag fails.

`gc` removes untagged manifests and the layers nothing else uses. An
image pushed only by digest, never tagged, counts as untagged. Older
registry versions deleted the platform images of multi-platform tags this
way; on 3.1.2 a kept tag still pulls on every platform (checked with
`crane export --platform`). The registry floats on
`registry:${REGISTRY_VERSION:-3}`, and each release pulls its fixes, so
check again as the image moves.

## Locally

With a local devopsy-template-traefik, the registry is at
`https://registry.localhost` (the config's project name):

```sh
devopsy --release
devopsy credentials
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
