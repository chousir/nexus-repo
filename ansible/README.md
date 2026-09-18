# Air-gap Nexus deployment (Ansible)

Containerized **Nexus Repository 3** behind a containerized **nginx** TLS reverse
proxy. Images are loaded into the local Docker image store beforehand — no
registry access required at run time. This repo doesn't track image tarballs;
see `../nexus-airgap-verification.md` for the save-on-a-connected-host /
load-on-the-air-gapped-host workflow.

## Layout

```
ansible.cfg / site.yml
inventory/hosts               # localhost, ansible_connection=local
inventory/group_vars/all.yml  # all tunables
roles/nexus/
  tasks/nexus.yml             # run the Nexus container, baseline (admin pw / EULA /
                              #   anonymous access / shared anonymous role), then the
                              #   nginx TLS proxy (SANs regenerated when the list changes,
                              #   base nginx.conf, conf.d/ dir, start proxy, verify)
  tasks/helm.yml              # helm-hosted + anonymous read/write (path-based, shares
                              #   nexus.lab like raw/cargo would — no vhost of its own)
  templates/docker-compose.yml.j2
  templates/nginx.conf.j2     # base: UI/API/Maven vhost + `include conf.d/*.conf`
  docker-registry/            # self-contained: Docker registry + its own nginx vhost
    tasks/main.yml            #   Bearer realm + docker-hosted + anon pull/push + verify
    templates/docker-nginx.conf.j2   # registry.lab + hijacked names -> conf.d
  maven-registry/             # self-contained: maven-hosted + its own nginx vhost
    tasks/main.yml            #   maven-hosted + anon read/write, then the Maven
                              #   Central domain-hijack vhost + verify
    templates/maven-nginx.conf.j2    # /maven2/ rewritten to /repository/maven-hosted/
```

A `<name>-registry/{tasks,templates}/` folder next to `roles/nexus/` exists
**only when a format needs its own nginx vhost** (docker: extra names on the
same vhost; maven: a `/maven2/` path rewrite) — each one is fully
self-contained: repo/privilege creation and the vhost that serves it live in
the same `tasks/main.yml`. Formats that are purely path-based under
`nexus.lab` (helm, and eventually raw/cargo) are a flat `tasks/<format>.yml`
instead, same shape as `helm.yml` — there's nothing for a template to render.

## Run

The play does not pull or load images itself — load `nexus_image` and
`nginx_image` (tags must match `inventory/group_vars/all.yml` exactly) into
the local Docker image store first, however you staged them:

```
docker load -i nexus3.tar
docker load -i nginx.tar
```

See `../nexus-airgap-verification.md` for the full connected-host `docker
save` → transfer → air-gapped-host `docker load` procedure.

```
ansible-playbook site.yml --become        # paths default to /opt/nexus
```

Override any value in `inventory/group_vars/all.yml`, e.g. hostname:

```
ansible-playbook site.yml --become -e nexus_server_name=repo.corp.local
```

Add `-e @test-vars.yml` for a rootless local run (writable paths + named volume).

## After deploy

Point DNS or `/etc/hosts` for `<nexus_server_name>` and `<nexus_registry_server_name>`
at the host, and trust the generated CA `<nexus_project_dir>/certs/nexus.crt`
(default `/opt/nexus/compose/certs/nexus.crt`; one cert covers both names).

- **UI / API** — `https://<nexus_server_name>/`
- **Admin password** — set to `nexus_admin_password` on first boot (default
  `admin-please-change`; override it, ideally with `ansible-vault`).
- **maven-hosted** (MIXED) — `https://<nexus_server_name>/repository/maven-hosted/`,
  **anonymous read/write** (no `mvn` credentials).
- **docker-hosted** — `<nexus_registry_server_name>` (default `registry.lab`),
  **anonymous pull/push** (no `docker login`). Docker clients need
  `/etc/docker/certs.d/<name>/ca.crt` = the generated `nexus.crt`.
- **helm-hosted** — `https://<nexus_server_name>/repository/helm-hosted/`,
  **anonymous read/write** (`helm push`/`helm repo add` need no credentials).

### Domain-hijacked pulls (docker-registry)

`nexus_registry_extra_server_names` (default: `[docker.elastic.co]`, as an
example) adds extra names to the *same* registry vhost/cert/connector as
`nexus_registry_server_name`. Nothing is proxied upstream — it's still one
`docker-hosted` repo, so whatever path a client pulls/pushes through any of
these names is what gets stored, e.g. `docker.elastic.co/elasticsearch/...`
lands at `elasticsearch/...`. Add more names to the list to add more.

Test without touching system DNS or trust stores:

```
curl --resolve docker.elastic.co:443:<nginx-host-ip> \
     --cacert <nexus_project_dir>/certs/nexus.crt \
     https://docker.elastic.co/v2/
```

Real `docker` client (needs `/etc/hosts` + a trusted cert — both need `sudo`):

```
echo "<nginx-host-ip> docker.elastic.co" | sudo tee -a /etc/hosts
sudo mkdir -p /etc/docker/certs.d/docker.elastic.co
sudo cp <nexus_project_dir>/certs/nexus.crt /etc/docker/certs.d/docker.elastic.co/ca.crt
docker tag docker.elastic.co/elasticsearch/elasticsearch:9.0.0 localhost:5000/elasticsearch/elasticsearch:9.0.0
docker push localhost:5000/elasticsearch/elasticsearch:9.0.0   # seed it once, from the Nexus host
docker pull docker.elastic.co/elasticsearch/elasticsearch:9.0.0   # now resolves to docker-hosted
```

### Domain-hijacked Maven Central (maven-registry)

`nexus_maven_extra_server_names` (default: `[repo1.maven.org, repo.maven.apache.org]`)
adds a vhost that rewrites `/maven2/<path>` to `/repository/maven-hosted/<path>`,
so unmodified `mvn`/`sbt`/`gradle` clients using the default Central URL resolve
straight to `maven-hosted` — unlike docker-registry, this **is** a path rewrite,
not just an extra name on an existing vhost.

Test without touching system DNS or trust stores:

```
curl --resolve repo1.maven.org:443:<nginx-host-ip> \
     --cacert <nexus_project_dir>/certs/nexus.crt \
     https://repo1.maven.org/maven2/com/example/probe/1.0/probe-1.0.txt
```

## Teardown

```
docker compose -p nexus down
```
