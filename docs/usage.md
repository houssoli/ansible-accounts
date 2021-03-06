## User guide

### Requirements

  This roles contains tasks relying on the `synchronize` module (rsync) and
  therefore it requires the `openssh` daemon to be running.

  However, if no underlying connection is not `ssh`, it will fall back to the
  copy module.

### Introduction

**Usergroups**

This role works in a usergroups paradigm. As your coworkers also needs to
login on the same servers, user accounts are configured by adding one or
multiple usergroups composed of one or more individual users that will end up
attached to the unix group named after the usergroup name. Phew!

There is also `noadmin` mode for when you have no control over the remote
machine. You can use this on the restricted machines to deploy your public
ssh keys and your dotfiles. You will be touching **only your account**

**SSH keys management**

Private ssh keys are **only managed on specified hosts**. Agent-fowarding should be
considered when using a `jumphost` or hopping through a bastion host. To make
things manageable and flexible, following logic is used to allow a user to
have have multiple ssh keys.

  * default keyname id_rsa and id_rsa.pub are not use for complexity reasons
  * default keys give access to an internal domain (i.e. company domain)
  * other keys might optionaly exist and give access to other external domains
  * keynames follow a `<user>.<domain>`

The `domain` part is important as it hints you what it gives access to. As
such, this is the role behavior regarding your keyring management.

  * default keys are deployed as <user>.<domain> and <user>.<domain>.pub on workstations
  * public keys will be deployed exclusively on target where ansible_fqdn == <user>.<domain>
  * ssh keys can be rotated, this behavior is disabled by default
  * keypairs are created on specified hosts at user account creations
  * all public keys of configured keypairs are fetched back to usergroup keyring
  * private keys are never fetched to the usergroup keyring
  * other keys can be requested to be generated at user account creation
  * every public key is deployed in an exclusive fashion
  * other keys are not deployed in the internal domain (keys are exclusive)
  * to make use of other keys, the user must configure his `ssh_config` properly

### Usage

**Usergroup directory scaffolding**

To add users, you need to configure usergroups inside `group_vars` or `host_vars`.

```yaml
usergroups:
  - name: vendorgroup
    # gid is optional
    gid: 1001
    # create vendorgroup sudoers file inside /etc/sudoers.d
    sudos:
      - ALL=(ALL) ALL

  - name: customergroup
```

You can also tweak the behavior on a per group or per machine basis. See
variables in 'Default vars' .

Then at the path defined by `groups_dir` variable (`{{ playbook_dir }}/private/groups`)
create a folder by the `usergroup name` with content such as

```bash
$ > tree
.
├── customergroup
│   └── users.yml
└── vendorgroup
     └── users.yml
```

for each of the groups you declared in `usergroups`. Other missing
directories inside the group directory will be created by the play.

**Usergroup members configuration**

In each group you will have a file where you define and configure the unix
group members. See the Role Variables` section for more information on ways
to personalise a user account. Most settings in `users_defaults` can be
overridden.

```shell
$ > cat vendorgroup/users.yml
---

users:

  - name: foo
    comments: 'foo account'
    groups:
      - adm
      - lp
      - users
    shell: "/bin/zsh"

    # optional
    ssh_domains:
      - company.domain

    # optional
    dotfiles_dir: dotfiles
    vim_dir: .vim

    # optional
    dotfiles_symlinks:
      - vimrc
      - bashrc
      - zshrc
      - gitconfig
      - git_template
      - profile
      - zprofile
      - tmux.conf
      - ansible.cfg
      - ctags
      - pypirc

  - name: bar
    comments: 'bar user'
    ssh_domains:
      - lan
      - example.com

  - name: baz
    comments: 'baz user'
    groups:
      - users
    ssh_domains:
      - lan
      - example.org
```

**When remote_user is different than local_user **

see `users_usermap` in `defaults/main.yml`

**User ssh_configuration**

Each user can optionaly manage their `ssh_config` with ansible. For that to happen,
you have to create a `<username>_ssh_config.yml file inside the group directory.

For example for the *foo* user above, the statements below will be transmuted into a
`~/.ssh/config` Parameters from `users_defaults['ssh_config']` are also
considered. This file is *totaly optional*. The playbook will not fail if it
is missing for any given user.

```bash
$ > cat vendorgroup/foo_ssh_config.yml
---

ssh_config:
  - Host: bitbucket.org
    User: git
    ForwardX11: no
    PreferredAuthentications: publickey
    ControlMaster: no

  - Host: github.com
    User: git
    ForwardX11: no
    PreferredAuthentications: publickey
    ControlMaster: no

  - Host: "*lb-*"
    User: archambf
    Hostname: "%h.lb.labfqdn"
    ForwardAgent: yes
    StrictHostKeyChecking: yes
    ProxyCommand: ssh -W %h:%p jumphost
```

As you might notice,

* Every key must be a valid `ssh_config` option.
* `Match` statements are supported.
* Quote strings with special chars else yaml parsing will fail

**Using the 'noadmin' mode**

For this restricted mode, just set the `users_noadmin' boolean to yes|True
either on the cli (`-e users_noadmin=True`) or in your playbook variables.
