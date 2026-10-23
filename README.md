# devopsy-template-registry

A private Docker registry ([CNCF distribution](https://distribution.github.io/distribution/))
on a [devopsy](https://github.com/hanoii/devopsy-cli) server, behind
[devopsy-template-traefik](https://github.com/hanoii/devopsy-template-traefik), with two users: one
that pushes, for CI or a build server, and a read-only one, for the servers
that pull the images. Use it for devopsy's image mode: build and push images
from CI or a build server, and have each environment pull its commit's image.

## Using this template

A starting point to own, not a dependency: start a project from it on
GitHub ("Use this template"), or clone it and keep this repository as a
remote (`upstream`) to pull its changes when you choose. Nothing updates
your copy, or what runs on your servers, but your own release.

## Try it

Point the target at your server in `.devopsy/.env` (gitignored), or in the
environment:

```sh
echo DEVOPSY_SERVER=devopsy@203.0.113.10 >> .devopsy/.env
```

It gets `<project>-<environment>.<server's wildcard domain>` (its compose
project name, like `registry-prod`), whose domain each release imports
from the server's proxy (devopsy-template-traefik), through the
`devopsy.import` label in `compose.yaml`: a release fails while no proxy
runs. The release says what it imported; `devopsy @prod --debug imports`
shows it later, and whether the proxy has changed it since. Override it,
or set it empty for none, per target: `devopsy @prod --vars set --show
DEVOPSY_WILDCARD_DOMAIN`.

```sh
devopsy @prod --release          # runs deploy: https://registry-prod.<server's wildcard domain>
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

And on the servers that run the images, the `pull` user: it can pull and
list, never push or delete, so a server holds no credential that could
replace an image.

```sh
docker login registry-prod.example.com -u pull
```

## How it works

- **Secrets.** The first release generates `REGISTRY_PASSWORD`,
  `REGISTRY_PULL_PASSWORD` and `REGISTRY_HTTP_SECRET` into the server's
  `shared/.env` (`secrets`, its prepare step, before it goes live). Every
  `deploy` checks `mnt/auth/htpasswd` against the two users, `REGISTRY_USER`
  (default `devopsy`) and `REGISTRY_PULL_USER` (default `pull`), and their
  passwords, and rewrites it (bcrypt, through `httpd`'s `htpasswd`) when
  they differ.
- **Rotating.** `devopsy @prod rotate pull` (or `push`) writes a new random
  password into `shared/.env`, deploys, and prints the new login. The old
  password stops working at once: update it where that user logs in. To
  choose a password instead, `devopsy @prod --vars set
  REGISTRY_PULL_PASSWORD`, then `devopsy @prod deploy`.
- **The read-only user.** The registry's htpasswd has no permissions: every
  user in it can push. Traefik makes `pull` read-only: pulls are `GET` and
  `HEAD` requests (manifests, blobs, the catalog), and a second router sends
  every other method (uploads, tags, deletes) through a basic auth of the
  push user alone, before the registry checks the same credentials again.
  `deploy` writes that user's htpasswd line into `.env` as
  `REGISTRY_PUSH_USERS` for the router's label, in single quotes (compose
  expands `$` in unquoted values). The `pull` user can still list every
  repository and tag.
- **Storage.** Images live in `shared/mnt/registry`: back it up, or accept
  rebuilding images. The registry runs as UID 10001 with a read-only root
  filesystem and no capabilities, and devopsy gives it the storage
  directory (the `devopsy.mnt.owner` label on the service), which the
  deploy user cannot do. The tools (`htpasswd`, `crane`) run as 10001
  without capabilities too.
- **Destroying.** `devopsy @prod --destroy` is devopsy's alone: it takes
  the containers and volumes down, empties `mnt/` as root, then removes
  the environment's directory. No destroy step.
- **Uploads.** Traefik limits how long a request may take, body included,
  to 60 seconds by default, which can cut a large layer pushed over a slow
  link. Raise it in the server's Traefik settings (`600s`, or `0` for no
  limit): `devopsy @<server>-traefik --vars set --show
  DEVOPSY_READ_TIMEOUT`, then `devopsy @<server>-traefik deploy`.

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

They run [crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane)
in the `tools` profile's `crane` service, as the push user, on the
project's network (`registry:5000`), never through Traefik; `devopsy
@prod crane ...` runs any other crane command there, like `devopsy @prod
crane ls registry:5000/myapp`. `delete` lists the tags first and asks
to type `delete`, or `delete all` for a whole repository (`--yes` skips
it, and is needed without a terminal). It removes only the tags: an
image another tag points to stays (`images` shows which tags share a
digest), and the repository stays listed, with no tags. Sizes count each
tag's layers, shared or not, so they add up to more than `--status` shows. A server that ran a deleted tag keeps its
copy for rollbacks, but a new release of that tag fails.

`gc` removes untagged manifests and the layers nothing else uses. An image
pushed only by digest, never tagged, counts as untagged. Older registry
versions deleted the platform images of multi-platform tags this way; on
3.1.2 a kept tag still pulls on every platform (checked with
`crane export --platform`). Check again when upgrading the registry image.

## Locally

With a local devopsy-template-traefik, the registry is at
`https://registry.localhost` (the config's project name):

```sh
devopsy --release
devopsy credentials
```

## License

MIT, so you can start your own project from it. See [LICENSE](LICENSE).
