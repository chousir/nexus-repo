# nexus-doc

Air-gapped **Nexus Repository 3** (Docker) behind a containerized **nginx**
TLS reverse proxy, deployed with Ansible — no Galaxy collections, no registry
access required at deploy time. Ships Docker/Maven/Helm/PyPI hosted repos,
plus domain-hijack vhosts so unmodified `docker`/`mvn`/`sbt`/`pip`/`twine`
clients resolve straight to the local mirror instead of the public registry.
All formats are anonymous read/write except PyPI, where `pip install` is
anonymous but `twine upload` needs a dedicated `pypi-uploader` account.

## Layout

- [`ansible/`](ansible/) — the Ansible project itself. See
  [`ansible/README.md`](ansible/README.md) for deploy instructions
  (prerequisites, variables, layout, teardown).
- [`nexus-airgap-verification.md`](nexus-airgap-verification.md) — post-deploy
  verification (Maven/Docker/Helm upload/download, domain-hijack tests), plus
  two fully offline appendices: mirroring a Maven/sbt project's whole
  dependency tree, and installing Ansible itself on a machine with zero
  network access. Written in Traditional Chinese.

## Quick start

```bash
cd ansible
docker pull sonatype/nexus3:3.93.2 && docker pull nginx:1.27-alpine
ansible-playbook site.yml -e @test-vars.yml   # rootless local test, writes to ansible/.deploy/
# ansible-playbook site.yml --become          # real deploy, defaults to /opt/nexus
```

For an actual air-gapped target — no image pulls, no `ansible-playbook`
pre-installed — see the image-transfer section and Appendix B in
`nexus-airgap-verification.md`.

## What's not tracked here

This repo is source only — see `.gitignore`. Nothing binary or
machine-generated is committed:

- `ansible/.deploy/` — the local-test deploy (generated TLS cert/key, rendered
  `nginx.conf`/`docker-compose.yml`) created by `-e @test-vars.yml`.
- Image tarballs and any `stage/`/`bundle/` transfer staging — built fresh
  from the procedures in `nexus-airgap-verification.md`, not shipped as files.
- `*.crt`/`*.key`/`*.pem` — always self-signed and generated at deploy time,
  never checked in.

The only "secret-shaped" value in the repo is
`nexus_admin_password: admin-please-change` in
`ansible/inventory/group_vars/all.yml` — a placeholder default. If you set it
to a real password for your own deployment, override it with `-e` or
`ansible-vault` rather than editing it in place (see `ansible/README.md`).
