# shawly.compose_artificer

The `shawly.compose_artificer` Ansible role can be used as utility for deploying
Docker Compose stacks in a unified way. The idea is to a have a base role that
executes all common tasks that are necessary to get a compose project up and
running, it should do the following:

**set_facts.yml**

- Sets facts with all the variables that are used in the project

**install.yml**

- Creates all the necessary directories for the project
- Uploads the contents of the docker-compose.yml file

**run.yml**

- Executes docker compose config for validation
- Run the project

## Usage

`shawly.compose_artificer` should be used with the `ansible.builtin.import_role`
task, the minimum required variables are
`compose_artificer_compose_project_name` and `compose_artificer_role_prefix`. An
example `main.yml` would look like this:

```yaml
# it is very important to set these via set_fact and not as vars for import_role, otherwise `role_name` will contain "compose_artificer" instead of "myrole"!
- name: Set facts for compose_artificer
  ansible.builtin.set_fact:
    # make sure to keep {{ role_name }}
    compose_artificer_role_prefix: "{{ role_name }}"
    compose_artificer_compose_project_name:
      "{{ myrole_compose_project_name | default(role_name) }}"
  tags:
    - myrole_install
    - myrole_run

- name: Set facts for {{ role_name }}
  ansible.builtin.import_role:
    name: shawly.compose_artificer
    tasks_from: set_facts
  tags:
    - myrole_install
    - myrole_run

- name: Install files for {{ role_name }}
  ansible.builtin.import_role:
    name: shawly.compose_artificer
    tasks_from: install
  tags:
    - myrole_install

- name: Run services for {{ role_name }}
  ansible.builtin.import_role:
    name: shawly.compose_artificer
    tasks_from: run
  tags:
    - myrole_run
```

The next requirement is a `docker-compose.yml` inside your role's `vars/`
directory that contains your compose stack. To get all that set up quickly, you
can use the `shawly.compose_artificer_skeleton` skeleton role.

### Skeleton

To make creating new roles that use `shawly.compose_artificer` easier, use the
`shawly.compose_artificer_skeleton` role e.g.:

```bash
git clone https://github.com/shawly/ansible-compose_artificer_skeleton.git shawly.compose_artificer_skeleton
ansible-galaxy init --role-skeleton shawly.compose_artificer_skeleton "${role_name:?}"
```

After you created your role you must adjust the `vars/docker-compose.yml` in
your role and set your required vars within `defaults/main.yml`. Then you can
configure necessary project dirs within `vars/main.yml` which can be used for
bind mounts inside your compose project.

#### Example

Creating an nginx role is as simple as executing
`ansible-galaxy init --role-skeleton shawly.compose_artificer_skeleton nginx`.

and editing the following files for example like this:

**`nginx/defaults/main.yml`**

```yaml
nginx_container_name: nginx-static-mywebsite
nginx_container_image_tag: alpine
nginx_compose_project_name: mywebsite
nginx_traefik_subdomain: mysubdomain
nginx_traefik_domain: example.com
nginx_traefik_port: 80
```

**`nginx/vars/docker-compose.yml`**

```yaml
---
version: "3"

services:
  app:
    image:
      docker.io/library/nginx:{{ nginx_container_image_tag | default('latest')
      }}
    container_name:
      "{{ nginx_container_name | default(nginx_compose_project_name) }}"
    labels:
      # if you use traefik you can set your traefik labels here
      traefik.enable: true
      traefik.http.routers.nginx-http.service: nginx
      traefik.http.routers.nginx-http.entrypoints: web
      traefik.http.routers.nginx-http.rule:
        Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`)
      traefik.http.routers.nginx.service: nginx
      traefik.http.routers.nginx.entrypoints: web-secure
      traefik.http.routers.nginx.middlewares: secHeaders@file
      traefik.http.routers.nginx.rule:
        Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`)
      traefik.http.routers.nginx.tls: true
      traefik.http.routers.nginx.tls.certresolver: cloudflare
      traefik.http.services.nginx.loadbalancer.server.port:
        "{{ nginx_traefik_port }}"
    volumes:
      - "{{ nginx_www_dir.dir }}:/usr/share/nginx/html:ro"
      - "{{ nginx_project_root_dir.dir }}/nginx.conf:/etc/nginx/nginx.conf:ro"
    networks:
      # this is for reverese proxying with traefik
      traefik_net:
    # if you do not use traefik, you can simply open the ports like this
    ports:
      - "80:80"

networks:
  traefik_net:
    external: true
```

**`nginx/vars/main.yml`**

```yaml
nginx_default_compose_project_dirs:
  nginx_www_dir:
    dir: "{{ nginx_compose_project_root_dir.dir }}/www"
    # nginx uid and gid is 101
    owner: "101"
    group: "101"
    mode: "0755"
    recurse: true
```

**`nginx/tasks/install.yml`**

```yaml
- name: install | Upload nginx.conf file
  become: true
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "{{ nginx_project_root_dir.dir }}/nginx.conf"
    owner: "101"
    group: "101"
    mode: "0644"
  # noqa loop-var-prefix[missing]
  with_first_found:
    # load nginx.conf from any of the following paths under files/
    - nginx/{{ inventory_hostname }}/nginx.conf
    - nginx/nginx.conf
    - nginx.conf
  notify: Restart services # this will restart the compose project if the docker-compose.yml didn't change
```

Overriding the some defaults for a specific host is as simple as creating a host
var with `nginx_compose_overrides` like this:

**`inventories/myhost/mycustomnginx.yml`**

```yaml
---
nginx_traefik_subdomain: myothersubdomain

nginx_compose_overrides:
  services:
    app:
      labels:
        # for myhost the subdomain is myothersubdomain and the nginx is also available without subdomain
        traefik.http.routers.nginx-http.rule:
          Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`) ||
          Host(`{{ nginx_traefik_domain }}`)
        traefik.http.routers.nginx.rule:
          Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`) ||
          Host(`{{ nginx_traefik_domain }}`)
```

## Role Variables

This is a list of all role variables for `shawly.compose_artificer`:

- `compose_artificer_compose_project_name`: The name of the compose project to
  use
  - Mandatory: must be set through your dependant role via `set_fact`!
- `compose_artificer_role_prefix`: The prefix for the role name
  - Mandatory: must be set through your dependant role via `set_fact`!
- `compose_artificer_timezone`: The timezone to use for all timezone variables
  etc.
  - Default: `Europe/Berlin`
- `compose_artificer_default_dir_owner`: The owner to use for a project dir.
  - Default: `root`
- `compose_artificer_default_dir_group`: The group to use for a project dir.
  - Default: `root`
- `compose_artificer_default_dir_mode`: The mode to use for a project dir.
  - Default: `0755`
- `compose_artificer_default_dir_recurse`: Whether or not to create the project
  dir recursively.
  - Default: `false`
- `compose_artificer_default_service_template`: A template that is used as base
  for all services. It should contain default environment variables (like `TZ`),
  restart policy and default labels that you want to set on all services.
  - Default: [see defaults/main.yml](./defaults/main.yml)
- `compose_artificer_default_zfs_root_pool`: The ZFS pool to use for the root
  dataset of the compose project.
  - Default: `zroot`
- `compose_artificer_default_zfs_root_dataset`: The ZFS dataset to use for the
  root dataset of the compose project.
  - Default: `zroot/ROOT/ubuntu`

## Example Playbook

```yaml
- hosts: servers
  roles:
    - { role: shawly.compose_artificer, x: 42 }
```	

## License

GPLv3
