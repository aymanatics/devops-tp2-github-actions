# TP3 Ansible

## Inventory

Inventory file: `inventories/setup.yml`

Target host:

```text
ayman.benhadid.takima.school
```

Default SSH user: `admin`

The local inventory uses `../id_rsa`. In GitHub Actions the key is injected from the
`ANSIBLE_SSH_PRIVATE_KEY` secret and passed with `--private-key`.

## Useful Commands

From this directory:

```bash
ANSIBLE_HOME=.ansible ansible all -m ping
ANSIBLE_HOME=.ansible ansible all -m setup -a "filter=ansible_distribution*"
ANSIBLE_HOME=.ansible ansible-playbook playbook.yml --syntax-check
ANSIBLE_HOME=.ansible ansible-playbook playbook.yml
```

When DockerHub images are not available, build the images directly on the server:

```bash
ANSIBLE_HOME=.ansible BUILD_IMAGES_ON_REMOTE=true ansible-playbook playbook.yml
```

## Roles

- `docker`: installs Docker Engine and the Python Docker SDK in `/opt/docker_venv`.
- `build_images`: optional local fallback to copy sources and build Docker images on the server.
- `network`: creates the Docker network used by the stack.
- `database`: starts PostgreSQL with a persistent Docker volume.
- `app`: starts the Spring Boot API.
- `proxy`: starts Apache HTTPD, serving the front and proxying API calls.
