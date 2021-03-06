Galaxy
======

An [Ansible][ansible] role for installing and managing [Galaxy][galaxyproject] servers.  Despite the name confusion,
[Galaxy][galaxyproject] bears no relation to [Ansible Galaxy][ansiblegalaxy].

[ansible]: http://www.ansible.com/
[galaxyproject]: https://galaxyproject.org/
[ansiblegalaxy]: https://galaxy.ansible.com/

Requirements
------------

This role has the same dependencies as the git module. In addition, [Python virtualenv][venv] is required (as is
[pip][pip], but pip will be automatically installed with virtualenv). These can easily be installed via a pre-task in
the same play as this role:

```yaml
- hosts: galaxyservers
  pre_tasks:
    - name: Install Dependencies
      apt:
        name: "{{ item }}"
      become: yes
      when: ansible_os_family == 'Debian'
      with_items:
        - git
        - python-virtualenv
    - name: Install Dependencies
      yum:
        name: "{{ item }}"
      become: yes
      when: ansible_os_family == 'RedHat'
      with_items:
        - mercurial
        - python-virtualenv
  roles:
    - galaxyproject.galaxy
```

If your `git` executable is not on `$PATH`, you can specify its location with the `git_executable` variable. Likewise
with the `virtualenv` executable and corresponding `galaxy_virtualenv_command` variable.

[git]: http://git-scm.com/
[venv]: http://virtualenv.readthedocs.org/
[pip]: http://pip.readthedocs.org/

Role Variables
--------------

Not all variables are listed or explained in detail. For additional information about less commonly used variables, see
the [defaults file][defaults].

[defaults]: defaults/main.yml

Many variables control where specific files are placed and where Galaxy writes data. In order to simplify configuration,
you may select a *layout* with the `galaxy_layout` variable. Which layout you choose affects the required variables.

### Required variables ###

If using any layout other than `root-dir`:

- `galaxy_server_dir`: Filesystem path where the Galaxy server code will be installed (cloned).

If using `root-dir`:

- `galaxy_root`: Filesystem path of the root of a Galaxy deployment, the Galaxy server code will be installed in to a
  subdirectory of this directory.

### Optional variables ###

**Layout control**

- `galaxy_layout`: available layouts can be found in the [vars/][vars] subdirectory and possible values include:
  - `root-dir`: Everything is laid out in subdirectories underneath a single root directory.
  - `opt`: An [FHS][fhs]-conforming layout across multiple directories such as `/opt`, `/etc/opt`, etc.
  - `legacy-improved`: Everything underneath the Galaxy server directory, as with `run.sh`.
  - `legacy`: The default layout prior to the existence of `galaxy_layout` and currently the default so as not to break
    existing usage of this role.
  - `custom`: Reasonable defaults for custom layouts, requires setting a few variables as described in
    [vars/layout-custom.yml][custom]

Either the `root-dir` or `opt` layout is recommended for new Galaxy deployments.

Options below that control individual file or subdirectory placement can still override defaults set by the layout.

[vars]: vars/
[custom]: vars/layout-custom.yml
[fhs]: http://www.pathname.com/fhs/

**New options for Galaxy 18.01 and later**

- `galaxy_config_style` (default: `ini-paste`): The type of Galaxy configuration file to write, `ini-paste` for the
  traditional PasteDeploy-style INI file, or `yaml` for the YAML format supported by uWSGI.
- `galaxy_app_config_section` (default: depends on `galaxy_config_style`): The config file section under which the
  Galaxy config should be placed (and the key in `galaxy_config` in which the Galaxy config can be found. If
  `galaxy_config_style` is `ini-paste` the default is `app:main`. If `galaxy_config_style` is `yaml`, the default is
  `galaxy`.
- `galaxy_uwsgi_yaml_parser` (default: `internal`): Controls whether the `uwsgi` section of the Galaxy config file will
  be written in uWSGI-style YAML or real YAML. By default, uWSGI's internal YAML parser does not support real YAML. Set
  to `libyaml` to write real YAML, if you are using uWSGI that has been compiled with libyaml. see
  [unbit/uwsgi#863][uwsgi-863] for details.
- To override the default uWSGI configuration, place your uWSGI options under the `uwsgi` key in the `galaxy_config`
  dictionary explained below. Note that the defaults are not merged with your config, so you should fully define the
  `uwsgi` section if you choose to set it. Note that **regardless of which `galaxy_uwsgi_yaml_parser` you use, this
  should be written in real YAML** because Ansible parses it with libyaml, which does not support the uWSGI internal
  parser's duplicate key syntax. This role will automatically convert the proper YAML to uWSGI-style YAML as necessary.

[uwsgi-863]: https://github.com/unbit/uwsgi/issues/863

**Feature control**

Several variables control which functions this role will perform (all default to `yes` except where noted):

- `galaxy_create_user` (default: `no`): Create the Galaxy user. Running as a dedicated user is a best practice, but most
  production Galaxy instances submitting jobs to a cluster will manage users in a directory service (e.g.  LDAP). This
  option is useful for standalone servers. Requires superuser privileges.
- `galaxy_manage_paths` (default: `no`): Create and manage ownership/permissions of configured Galaxy paths. Requires
  superuser privileges.
- `galaxy_manage_clone`: Clone Galaxy from the source repository and maintain it at a specified version (commit), as
  well as set up a [virtualenv][virtualenv] from which it can be run.
- `galaxy_manage_static_setup`: Manage "static" Galaxy configuration files - ones which are not modifiable by the Galaxy
  server itself. At a minimum, this is the primary Galaxy configuration file, `galaxy.ini`.
- `galaxy_manage_mutable_setup`: Manage "mutable" Galaxy configuration files - ones which are modifiable by Galaxy (e.g.
  as you install tools from the Galaxy Tool Shed).
- `galaxy_manage_database`: Upgrade the database schema as necessary, when new schema versions become available.
- `galaxy_fetch_dependencies`: Fetch Galaxy dependent modules to the Galaxy virtualenv.
- `galaxy_build_client`: Build the Galaxy client application (web UI).
- `galaxy_client_make_target` (default: `client-production-maps`): Set the client build type. Options include: `client`, 
  `client-production` and `client-production-maps`. See [Galaxy client readme][client-build] for details.
- `galaxy_manage_errordocs` (default: `no`): Install Galaxy-styled 413 and 502 HTTP error documents for nginx. Requires
  write privileges for the nginx error document directory.

[client-build]: https://github.com/galaxyproject/galaxy/blob/dev/client/README.md#complete-client-build

**Galaxy code and configuration**

Options for configuring Galaxy and controlling which version is installed.

- `galaxy_config`: The contents of the Galaxy configuration file (`galaxy.ini` by default) are controlled by this
  variable. It is a hash of hashes (or dictionaries) that will be translated in to the configuration
  file. See the Example Playbooks below for usage.
- `galaxy_config_files`: List of hashes (with `src` and `dest` keys) of files to copy from the control machine.
- `galaxy_config_templates`: List of hashes (with `src` and `dest` keys) of templates to fill from the control machine.
- `galaxy_local_tools`: List of local tool files or directories to copy from the control machine, relative to
  `galaxy_local_tools_src_dir` (default: `files/galaxy/tools` in the playbook).
- `galaxy_local_tools_dir`: Directory on the Galaxy server where local tools will be installed.
- `galaxy_dynamic_job_rules`: List of dynamic job rules to copy from the control machine, relative to
  `galaxy_dynamic_job_rules_src_dir` (default: `files/galaxy/dynamic_job_rules` in the playbook).
- `galaxy_dynamic_job_rules_dir` (default: `{{ galaxy_server_dir }}/lib/galaxy/jobs/rules`): Directory on the Galaxy
  server where dynamic job rules will be installed. If changed from the default, ensure the directory is on Galaxy's
  `$PYTHONPATH` (e.g. in `{{ galaxy_venv_dir }}/lib/python2.7/site-packages`) and configure the dynamic rules plugin in
  `job_conf.xml` accordingly.
- `galaxy_repo` (default: `https://github.com/galaxyproject/galaxy.git`): Upstream Git repository from which Galaxy
  should be cloned.
- `galaxy_commit_id` (default: `master`): A commit id, tag, branch, or other valid Git reference that Galaxy should be
  updated to. Specifying a branch will update to the latest commit on that branch. Using a real commit id is the only
  way to explicitly lock Galaxy at a specific version.
- `galaxy_force_checkout` (default: `no`): If `yes`, any modified files in the Galaxy repository will be discarded.

**Path configuration**

Options for controlling where certain Galaxy components are placed on the filesystem.

- `galaxy_venv_dir` (default: `<galaxy_server_dir>/.venv`): The role will create a [virtualenv][virtualenv] from which
  Galaxy will run, this controls where the virtualenv will be placed.
- `galaxy_config_dir` (default: `<galaxy_server_dir>`): Directory that will be used for "static" configuration files.
- `galaxy_mutable_config_dir` (default: `<galaxy_server_dir>`): Directory that will be used for "mutable" configuration
  files, must be writable by the user running Galaxy.
- `galaxy_mutable_data_dir` (default: `<galaxy_server_dir>/database`): Directory that will be used for "mutable" data
  and caches, must be writable by the user running Galaxy.
- `galaxy_config_file` (default: `<galaxy_config_dir>/galaxy.ini`): Galaxy's primary configuration file.

**User management and privilege separation**

- `galaxy_separate_privileges` (default: `no`): Enable privilege separation mode.
- `galaxy_user` (default: user running ansible): The name of the system user under which Galaxy runs.
- `galaxy_privsep_user` (default: `root`): The name of the system user that owns the Galaxy code, config files, and
  virtualenv (and dependencies therein).
- `galaxy_group`: Common group between the Galaxy user and privilege separation user. If set and `galaxy_manage_paths`
  is enabled, directories containing potentially sensitive information such as the Galaxy config file will be created
  group- but not world-readable. Otherwise, directories are created world-readable.

**Access method control**

The role needs to perform tasks as different users depending on which features you have enabled and how you are
connecting to the target host. By default, the role will use `become` (i.e. sudo) to perform tasks as the appropriate
user if deemed necessary. Overriding this behavior is discussed in the [defaults file][defaults].

**Error documents**

- `galaxy_errordocs_dir`: Install Galaxy-styled HTTP 413 and 502 error documents under this directory. The 502 message
  uses nginx server side includes to allow administrators to create a custom message in `~/maint` when Galaxy is down.
  nginx must be configured separately to serve these error documents.
- `galaxy_errordocs_server_name` (default: Galaxy): used to display the message "`galaxy_errdocs_server_name` cannot be
  reached" on the 502 page.
- `galaxy_errordocs_prefix` (default: `/error`): Web-side path to the error document root.

**Miscellaneous options**

- `galaxy_restart_handler_name`: The role doesn't restart Galaxy since it doesn't control how Galaxy is started and
  stopped. Because of this, you can write your own restart handler and inform the role of your handler's name with this
  variable. See the examples
- `galaxy_admin_email_to`: If set, email this address when Galaxy has been updated. Assumes mail is properly configured
  on the managed host.
- `galaxy_admin_email_from`: Address to send the aforementioned email from.

Dependencies
------------

None

Example Playbook
----------------

### Basic ###

Install Galaxy on your local system with all the default options:

```yaml
- hosts: localhost
  vars:
    galaxy_server_dir: /srv/galaxy
  connection: local
  roles:
     - galaxy
```

Once installed, you can start with:

```console
$ cd /srv/galaxy
$ sh run.sh
```

### Best Practice ###

Install Galaxy as per the current production server best practices:

- Galaxy code (clone) is "clean": no configs or mutable data live underneath the clone
- Galaxy code and static configs are privilege separated: not owned/writeable by the user that runs Galaxy
- Configuration files are not world-readable
- PostgreSQL is used as the backing database
- The 18.01+ style YAML configuration is used
- Two [job handler mules][deployment-options] are started
- When the Galaxy code or configs are updated by Ansible, Galaxy will be restarted using `supervisorctl`

[deployment-options]: https://docs.galaxyproject.org/en/master/admin/scaling.html#deployment-options

```yaml
- hosts: galaxyservers
  vars:
    galaxy_config_style: yaml
    galaxy_layout: root-dir
    galaxy_root: /srv/galaxy
    galaxy_commit_id: release_18.09
    galaxy_separate_privileges: yes
    galaxy_create_user: yes
    galaxy_manage_paths: yes
    galaxy_user: galaxy
    galaxy_privsep_user: gxpriv
    galaxy_group: galaxy
    galaxy_restart_handler_name: Restart Galaxy
    postgresql_objects_users:
      - name: galaxy
        password: null
    postgresql_objects_databases:
      - name: galaxy
        owner: galaxy
    galaxy_config:
      uwsgi:
        socket: 127.0.0.1:4001
        buffer-size: 16384
        processes: 1
        threads: 4
        offload-threads: 2
        static-map:
          - /static/style={{ galaxy_server_dir }}/static/style/blue
          - /static={{ galaxy_server_dir }}/static
        master: true
        virtualenv: "{{ galaxy_venv_dir }}"
        pythonpath: "{{ galaxy_server_dir }}/lib"
        module: galaxy.webapps.galaxy.buildapp:uwsgi_app()
        thunder-lock: true
        die-on-term: true
        hook-master-start:
          - unix_signal:2 gracefully_kill_them_all
          - unix_signal:15 gracefully_kill_them_all
        py-call-osafterfork: true
        enable-threads: true
        mule:
          - lib/galaxy/main.py
          - lib/galaxy/main.py
        farm: job-handlers:1,2
      galaxy:
        database_connection: "postgresql:///galaxy?host=/var/run/postgresql"
  pre_tasks:
    - name: Install Dependencies
      apt:
        name: ['git', 'python-psycopg2', 'python-virtualenv']
      become: yes
  roles:
    # Install with:
    #   % ansible-galaxy install galaxyproject.postgresql
    - role: galaxyproject.postgresql
      become: yes
    # Install with:
    #   % ansible-galaxy install natefoo.postgresql_objects
    - role: natefoo.postgresql_objects
      become: yes
      become_user: postgres
    - role: galaxyproject.galaxy
  handlers:
    - name: Restart Galaxy
      supervisorctl:
        name: galaxy
        state: restarted
```

License
-------

[Academic Free License ("AFL") v. 3.0][afl]

[afl]: http://opensource.org/licenses/AFL-3.0

Author Information
------------------

This role was written and contributed to by the following people:

- [Enis Afgan](https://github.com/afgane)
- [Dannon Baker](https://github.com/dannon)
- [Simon Belluzzo](https://github.com/simonalpha)
- [John Chilton](https://github.com/jmchilton)
- [Nate Coraor](https://github.com/natefoo)
- [Helena Rasche](https://github.com/erasche)
