# shawly.compose_artificer (WIP)

The `shawly.compose_artificer` Ansible role can be used as utility for deploying Docker Compose stacks in a unified way.
The idea is to a have a base role that executes all common tasks that are necessary to get a compose project up and
running, it should do the following:

**NOTE:** This is still a work in progress and it is heavily tailored to my needs currently. Making things more dynamic
always means it will make things also more complex, so feature requests will probably not yet be implemented. Also we
need tests.

## Features

- Automatic filesystem detection and management (ZFS and BTRFS support)
- Automatic snapshot creation for data protection
- Flexible directory and subvolume/dataset management
- Docker Compose project validation and deployment
- UnRAID integration with composeman support
- Migration functionality for moving existing deployments
- Error handling and rollback capabilities

**set_facts.yml**

- Sets facts with all the variables that are used in the project

**install.yml**

- Creates all the necessary directories for the project
- Uploads the contents of the docker-compose.yml file
- Installs UnRAID-specific files for composeman integration (if running on UnRAID)
- Includes error handling with automatic rollback on failure

**migrate.yml**

- This task will migrate data from another location to the new project directory if necessary. This is useful for
  migrating existing data to a new Docker Compose project.
- Automatically creates necessary ZFS datasets or BTRFS subvolumes during migration
- Handles service shutdown before migration and restart after migration

**run.yml**

- Executes docker compose config for validation
- Run the project

**handlers/main.yml**

The role provides the following handler:

- `Restart services`: Restarts the Docker Compose project. This can be triggered by tasks that modify configuration
  files.

## Requirements

This role requires the following Ansible collections:

- `community.docker` - For Docker Compose operations
- `ansible.posix` - For mounting filesystems
- `community.general` - For ZFS and BTRFS operations

These are defined in `meta/requirements.yml` and can be installed with:

```bash
ansible-galaxy collection install -r meta/requirements.yml
```

## Usage

`shawly.compose_artificer` is not a standalone role that you execute directly. Instead, it's a role framework that
provides reusable components and defaults for creating Docker Compose-based roles. This design allows you to create
minimal roles with very little duplicated code by leveraging the common functionality provided by `compose_artificer`.

The role should be used with the `ansible.builtin.import_role` task from within your own role's tasks. An example
`main.yml` would look like this:

```yaml
- name: Set facts for {{ role_name }}
  ansible.builtin.import_role:
    name: shawly.compose_artificer
    tasks_from: set_facts
  tags:
    - myrole_migrate
    - myrole_install
    - myrole_run

- name: Migrate data for {{ role_name }} (optional)
  ansible.builtin.import_role:
    name: shawly.compose_artificer
    tasks_from: migrate
  vars:
    myrole_migrate_dir: "/old/path/to/migrate/from"
  tags:
    - myrole_migrate

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

The next requirement is a `docker-compose.yml` inside your role's `vars/` directory that contains your compose stack. To
get all that set up quickly, you can use the
[`shawly.compose_artificer_skeleton`](https://github.com/shawly/ansible-role-skeleton-compose_artificer.git) skeleton
role.

By using this framework approach, you can focus on the specific configuration and compose files for your service while
letting `compose_artificer` handle the common tasks like filesystem management, directory creation, snapshot handling,
and deployment logic. This significantly reduces code duplication across multiple Docker Compose-based roles.

### Skeleton

To make creating new roles that use `shawly.compose_artificer` easier, use the
[`shawly.compose_artificer_skeleton`](https://github.com/shawly/ansible-role-skeleton-compose_artificer.git) role e.g.:

```bash
git clone https://github.com/shawly/ansible-role-skeleton-compose_artificer.git shawly.compose_artificer_skeleton
ansible-galaxy init --role-skeleton shawly.compose_artificer_skeleton "${role_name:?}"
```

After you created your role you must adjust the `vars/docker-compose.yml` in your role and set your required vars within
`defaults/main.yml`. Then you can configure necessary project dirs within `vars/main.yml` which can be used for bind
mounts inside your compose project.

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
services:
  app:
    image: docker.io/library/nginx:{{ nginx_container_image_tag | default('latest') }}
    container_name: "{{ nginx_container_name | default(nginx_compose_project_name) }}"
    labels:
      # if you use traefik you can set your traefik labels here
      traefik.enable: true
      traefik.http.routers.nginx-http.service: nginx
      traefik.http.routers.nginx-http.entrypoints: web
      traefik.http.routers.nginx-http.rule: Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`)
      traefik.http.routers.nginx.service: nginx
      traefik.http.routers.nginx.entrypoints: web-secure
      traefik.http.routers.nginx.middlewares: secHeaders@file
      traefik.http.routers.nginx.rule: Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`)
      traefik.http.routers.nginx.tls: true
      traefik.http.routers.nginx.tls.certresolver: cloudflare
      traefik.http.services.nginx.loadbalancer.server.port: "{{ nginx_traefik_port }}"
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

Overriding the some defaults for a specific host is as simple as creating a host var with `nginx_compose_overrides` like
this:

**`inventories/myhost/mycustomnginx.yml`**

```yaml
---
nginx_traefik_subdomain: myothersubdomain

nginx_compose_overrides:
  services:
    app:
      labels:
        # for myhost the subdomain is myothersubdomain and the nginx is also available without subdomain
        traefik.http.routers.nginx-http.rule: >-
          Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`) || Host(`{{ nginx_traefik_domain }}`)
        traefik.http.routers.nginx.rule: >-
          Host(`{{ nginx_traefik_subdomain }}.{{ nginx_traefik_domain }}`) || Host(`{{ nginx_traefik_domain }}`)
```

## Role Variables

This is a list of all role variables for `shawly.compose_artificer`:

- `compose_artificer_compose_project_name`: The name of the compose project to use
  - Mandatory: must be set through your dependant role via `set_fact`!
- `compose_artificer_role_prefix`: The prefix for the role name
  - Mandatory: must be set through your dependant role via `set_fact`!
- `compose_artificer_timezone`: The timezone to use for all timezone variables etc.
  - Default: `Europe/Berlin`
- `compose_artificer_default_dir_owner`: The owner to use for a project dir.
  - Default: `root`
- `compose_artificer_default_dir_group`: The group to use for a project dir.
  - Default: `root`
- `compose_artificer_default_dir_mode`: The mode to use for a project dir.
  - Default: `0755`
- `compose_artificer_default_dir_recurse`: Whether or not to create the project dir recursively.
  - Default: `false`
- `compose_artificer_default_service_template`: A template that is used as base for all services. It should contain
  default environment variables (like `TZ`), restart policy and default labels that you want to set on all services.
  - Default: [see defaults/main.yml](./defaults/main.yml)
- `compose_artificer_default_zfs_skip_snapshot`: Whether to skip creating ZFS snapshots by default when
  installing/running services.
  - Default: `false`
- `compose_artificer_default_zfs_root_pool`: The ZFS pool to use for the root dataset of the compose project.
  - Default: `zroot`
- `compose_artificer_default_zfs_root_dataset`: The ZFS dataset to use for the root dataset of the compose project.
  - Default: `zroot/ROOT/ubuntu`
- `compose_artificer_default_btrfs_skip_snapshot`: Whether to skip creating BTRFS snapshots by default when
  installing/running services.
  - Default: `false`
- `compose_artificer_default_btrfs_filesystem_label`: The label of the BTRFS filesystem to use for subvolumes.
  - Default: `data`
- `compose_artificer_default_btrfs_subvolume_prefix`: The prefix for BTRFS subvolumes (the @ is a SUSE convention).
  - Default: `@`
- `compose_artificer_default_btrfs_default_mount_options`: Default mount options for BTRFS subvolumes.
  - Default: `defaults,compress=zstd,noatime,nodiratime,ssd,discard=async`

### Role-specific Variables

When using this role, you can also configure additional variables in your dependent role's `defaults/main.yml`:

#### ZFS Configuration

- `{role_prefix}_zfs_skip_snapshot`: Skip creating ZFS snapshots for this role.
- `{role_prefix}_zfs_root_dataset`: Override the ZFS root dataset for this role.
- `{role_prefix}_zfs_additional_datasets`: List of additional ZFS datasets to create.

#### Btrfs Configuration

- `{role_prefix}_btrfs_skip_snapshot`: Skip creating Btrfs snapshots for this role.
- `{role_prefix}_btrfs_filesystem_label`: Override the Btrfs filesystem label for this role.
- `{role_prefix}_btrfs_subvolume_prefix`: Override the Btrfs subvolume prefix for this role.
- `{role_prefix}_btrfs_additional_subvolumes`: List of additional Btrfs subvolumes to create.
- `{role_prefix}_btrfs_mount_options`: Dictionary of custom mount options per path.
- `{role_prefix}_btrfs_default_mount_options`: Override default mount options for this role.

#### Other Configuration

- `{role_prefix}_force_restart`: Force restart of services even if compose files haven't changed.
- `{role_prefix}_maintenance`: Enable maintenance mode (skips snapshots and some operations).
- `{role_prefix}_migrate_dir`: Path to migrate data from (triggers migration functionality).
- `{role_prefix}_compose_files`: Custom compose file names to use (default: docker-compose.yml).

#### UnRAID-specific Configuration

- `{role_prefix}_unraid_description`: Description for UnRAID composeman integration.
- `{role_prefix}_unraid_icon`: Icon file for UnRAID composeman integration.
- `{role_prefix}_unraid_shell`: Shell command for UnRAID composeman integration.
- `{role_prefix}_unraid_webui`: Web UI URL for UnRAID composeman integration.

#### Traefik Integration

- `{role_prefix}_traefik_domain`: Domain for Traefik reverse proxy integration.
- `{role_prefix}_traefik_subdomain`: Subdomain for Traefik integration (defaults to project name).
- `{role_prefix}_traefik_port`: Port for Traefik service configuration (default: 80).

## License

GPL-3.0-or-later

## Author Information

Created by [shawly](https://github.com/shawly).
