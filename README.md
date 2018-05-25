# deploy-template

This is a working example app which shows how to deploy a Phoenix app based on
this [best practices for deploying elixir
apps](https://www.cogini.com/blog/best-practices-for-deploying-elixir-apps/)
blog post.

It is based on a default Phoenix project without Ecto. The [changes](#changes)
are all additions, so you can easily add them to your own project.

It's tested deploying to [Digital Ocean](https://m.do.co/c/150575a88316) with
CentOS 7, Ubuntu 16.04, Ubuntu 18.04 and Debian 9.4.

It is based on [Ansible](https://www.ansible.com/resources/get-started), which
is an easy-to-use standard platform for managing servers.
Unlike edeliver, it is based on a reliable and well documented set of primitives
to handle logging into servers, uploading files and executing commands.
It can also be used to [support more complex deployment
scenarios](https://www.cogini.com/blog/setting-ansible-variables-based-on-the-environment/).

# Overall approach

1. Set up the web server, running Linux.
2. Set up a build server matching the architecture of the web server.
   This can be the same as the web server.
3. Check out code on the build server from git and build a release.
4. Deploy the release to the web server.

The actual work of checking out and deploying is handled by simple shell
scripts which you run on the build server or from your dev machine via
ssh, e.g.:

```shell
# Check out latest code and build release on server
ssh -A deploy@build-server build/deploy-template/scripts/build-release.sh

# Deploy release
ssh -A deploy@build-server build/deploy-template/scripts/deploy-local.sh
```

# Set up dev machine

Check out the code from git to your local dev machine:

```shell
git clone https://github.com/cogini/elixir-deploy-template
```

## Set up ASDF

Install ASDF as described in [the ASDF docs](https://github.com/asdf-vm/asdf).

Install ASDF plugins for our tools:

```shell
asdf plugin-add erlang
asdf plugin-add elixir
asdf plugin-add nodejs
```

Install the versions of Erlang, Elixir and Node.js specified in the
`.tool-versions` file:

```shell
asdf install
```
Run this multiple times until everything is installed (should be twice).

Install libraries into the ASDF Elixir dirs:

```shell
mix local.hex --force
mix local.rebar --force
mix archive.install https://github.com/phoenixframework/archives/raw/master/phx_new.ez --force
```

## Confirm that it works

Build the app the same as you normally would:

```shell
mix deps.get
mix deps.compile
mix compile
```

At this point you should be able to run the app locally with:

```shell
iex -S mix phx.server
open http://localhost:4000/
```

## Install Ansible

Install Ansible on your dev machine. May be as simple as:

```shell
pip install ansible
```

See [the Ansible docs](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
for other options.

# Set up server

An easy option is [Digital Ocean](https://m.do.co/c/150575a88316).
Their smallest $5/month Droplet will run Phoenix fine.

## Configure Ansible

### Define servers

Add the hosts to the `~/.ssh/config` file on your dev machine:

    Host web-server
        HostName 123.45.67.89

Add the hosts to the groups in the Ansible inventory `ansible/inventory/hosts`
file in the project:

    [web-servers]
    web-server

    [build-servers]
    web-server

The host name is not important, you can use an existing server. Use the
host name from your `.ssh/config` file in the `inventory/hosts` config and in
the `ansible-playbook` commands below.

If you are using a recent Ubuntu or Debian version that defaults to Python 3,
add the host to the `[py3-hosts]` group.

(The template has multiple hosts in the groups for testing different OS
versions, comment them out.)

### Set variables

Configuration vars defined in `inventory/group_vars/all` apply to all hosts in
your project. They are overridden by variables in more specific groups, e.g.
`inventory/group_vars/web-servers` or for individual hosts, e.g.
`inventory/group_vars/web-server`.

You should be using ssh keys to control access to server accounts, not
passwords. Ansible uses ssh to deploy the releases, and the `users` Ansible
role manages the keys to allow login.

The `inventory/group_vars/all/users.yml` defines a global list of users and
system admins. It has a live user (me!), change it to match your details:

```yaml
users_users:
  - user: jake
    name: "Jake Morrison"
    github: reachfh

users_global_admin_users:
 - jake
```

To generate an ssh key:

```shell
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
Set a pass phrase to protect access to your key. Modern Linux and macOS will
remember your pass phrase so you don't have to enter it every time (see `ssh-add`).
Add your `~/.ssh/id_rsa.pub` key to your GitHub account.

The `inventory/group_vars/all/elixir-release.yml` file specifies the
app settings:

```yaml
# External name of the app, used to name directories and the systemd process
elixir_release_name: deploy-template

# Internal "Elixir" name of the app, used to by Distillery to name directories
elixir_release_name_code: deploy_template

# Name of your organization or overall project, used to make a unique dir prefix
elixir_release_org: myorg

# OS user the app runs under
elixir_release_app_user: foo

# OS user for deploy
elixir_release_deploy_user: deploy

# Port that Phoenix listens on
elixir_release_http_listen_port: 4001
```

The `inventory/group_vars/build-servers/vars.yml` file specifies the build settings.

Set the name of the repo to be checked out on the build server:

```yaml
# App git repo
app_repo: https://github.com/cogini/elixir-deploy-template
```

## Set up web server

Run these and other Ansible commands from the `ansible` dir.

Do initial server setup:

```shell
ansible-playbook -u root -v -l web-servers playbooks/setup-web.yml -D
```

In this command, `web-servers` is the group of servers. Ansible allows you to
work on groups of servers simultaneously. Configuration tasks are written to be
idempotent, so we can run the playbook against all our servers and it will make
whatever changes are needed to get them up to date.

Set up the app (create app dirs, etc.).

```shell
ansible-playbook -u $USER -v -l web-servers playbooks/deploy-app.yml --skip-tags deploy -D
```

Configure runtime secrets, setting the `$HOME/.erlang.cookie` file and
generate a Conform file at `/etc/deploy-template/deploy_template.conf` with
variables such as `secret_key_base` and db config:

```shell
ansible-playbook -u $USER -v -l web-servers playbooks/config-web.yml -D
```

For ease of getting started, this generates secrets on your local machine and
stores them in `/tmp`.  See below for discussion about managing secrets.

At this point, the web server is set up, but we need to build and deploy
the app code to it.

## Set up build server

This can be the same as the web server or a separate server.

Set up the server, e.g. install ASDF:

```shell
ansible-playbook -u root -v -l build-servers playbooks/setup-build.yml -D
```

Configure `config/prod.secret.exs` on the build server.

```shell
ansible-playbook -u $USER -v -l build-servers playbooks/config-build.yml -D
```

See below for discussion about managing secrets.

## Build the app

Log into the `deploy` user on the build machine:

```shell
ssh -A deploy@build-server
```

The `-A` flag on the ssh command gives the session on the server access to your
local ssh keys. If your local user can access a GitHub repo, then the server
can do it, without having to put keys on the server. Similarly, if your ssh key
is on the prod server, then you can push code from the build server using
Ansible without the web server needing to trust the build server.

If you are using a CI server to build and deploy code, then it runs in the
background.  Create a deploy key in GitHub so it can access to your source and
add the ssh key on the build server to the `deploy` user account on the web
servers so the CI server can push releases.

Build the production release:

```shell
scripts/build-release.sh
```

That script runs:

```shell
# Pull latest code from git
git pull

# Update versions of Erlang/Elixir/Node.js if necessary
asdf install
asdf install

# Update Elixir libs
mix local.hex --force
mix local.rebar --force

# Build app and release
mix deps.get --only prod
MIX_ENV=prod mix do compile, phx.digest, release
```

`asdf install` builds Erlang from source, so the first time it runs
it can take a long time. You may want to run it under `tmux`.

## Deploy the release locally

If you are building on the web web server, then you can use the custom mix
tasks in `lib/mix/tasks/deploy.ex` to deploy locally.

In `mix.exs`, set `deploy_dir` to match Ansible, i.e.
`deploy_dir: /opt/{{ org }}/{{ elixir_release_name }}`:

```elixir
deploy_dir: "/opt/myorg/deploy-template/",
```

Deploy the release:

```shell
scripts/deploy-local.sh
```

That script runs:

```shell
MIX_ENV=prod mix deploy.local
sudo /bin/systemctl restart deploy-template
```

The build is being done under the `deploy` user, who owns the files under
`/opt/myorg/deploy-template` and has a special `/etc/sudoers.d` config which
allows it to run the `/bin/systemctl restart deploy-template` command.

### Verify it works

Make a request to the app supervised by systemd:

```shell
curl -v http://localhost:4001/
```

Have a look at the logs:
```shell
# systemctl status deploy-template
# journalctl -r -u deploy-template
```

Make a request to the machine over the network on port 80 through the magic of
[iptables port forwarding](https://www.cogini.com/blog/port-forwarding-with-iptables/).

You can get a console on the running app by logging in as the `foo` user the
app runs under and executing:

```shell
/opt/myorg/deploy-template/scripts/remote_console.sh
```

## Deploy to a remote machine using Ansible

### Install Ansible on the build machine

From your dev machine:

```shell
ansible-playbook -u $USER -v -l build-servers playbooks/setup-ansible.yml -D
```

On the build server:

```shell
ssh -A deploy@build-server
cd ~/build/deploy-template/ansible
```

Add the servers in the `ansible/inventory/hosts` `web-servers` group to `~/.ssh/config`:

    Host web-server
        HostName 123.45.67.89

For projects with lots of servers, we normally maintain the list of servers in
a `ssh.config` file in the repo. See `ansible/ansible.cfg` for config.

### Deploy the app

On the build server:

```shell
scripts/deploy-remote.sh
```

That script runs:

```shell
ansible-playbook -u deploy -v -l web-servers playbooks/deploy-app.yml --tags deploy --extra-vars ansible_become=false -D
```

### Managing secrets with Ansible

Ansible has a [vault](http://docs.ansible.com/ansible/2.5/user_guide/vault.html) function
which you can use to store keys. The lets you encrypt variable data and check
it into source control, and only people with the password can read it.

There are trade-offs in managing secrets.

For a small team of devs who are also the admins, then you trust your
developers and your own dev machine with the secrets. It's better not to have
secrets in the build environment, though. You can push the prod secrets
directly from your dev machine to the web servers. If you are using a 3rd-party
CI server, then that goes double. You don't want to give the CI service access
to your production keys.

For secure applications like health care, developers should not have access to
the prod keys. You can restrict vault password access to your ops team, or use
different keys for different environments.

You can also set up a build/deploy server in the cloud which has access to the
keys and configure the production instances from it. When we run in an AWS auto
scaling group, we build an AMI with [Packer](https://www.packer.io/) and
Ansible, putting the keys on it the same way. Even better, however, is to not
store keys on the server at all. Pull them when the app starts up, reading from
an S3 bucket or Amazon's KMS, with access controlled by IAM instance roles.

The one thing that really needs to be there at startup is the Erlang cookie,
everything else we can pull at runtime. If we are not using the Erlang
distribution protocol, then we don't need to share it, it just needs to be
secure.

The following shows describes how you can use the vault.

Generate a vault password and put it in the file `ansible/vault.key`:

```shell
openssl rand -hex 16
```

You can specify the password when you are running a playbook with the
`--vault-password-file vault.key` option, or you can make the vault password always
available by setting it in `ansible/ansible.cfg`:

    vault_password_file = vault.key

The `ansible/inventory/group_vars/web-servers/secrets.yml` file specifies deploy secrets.

Generate a cookie for deployment and copy it into the `secrets.yml` file:

```shell
openssl rand -hex 32 | ansible-vault encrypt_string --vault-id vault.key --stdin-name 'erlang_cookie'
```
That generates encrypted data like:

```yaml
erlang_cookie: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          64346139623638623838396261373265666363643264333664633965306465313864653033643530
          3830366538366139353931323662373734353064303034660a326232343036646339623638346236
          39623832656466356338373264623331363736636262393838323135663962633339303634353763
          3935623562343131370a383439346166323832353232373933613363383435333037343231393830
          35326662353662316339633732323335653332346465383030633333333638323735383666303264
          35663335623061366536363134303061323861356331373334653363383961396330386136636661
          63373230643163633465303933396336393531633035616335653234376666663935353838356135
          36323866346139666462
```

Generate `secret_key_base` for the server the same way:

```shell
openssl rand -base64 48 | ansible-vault encrypt_string --vault-id vault.key --stdin-name 'secret_key_base'
```

Generate `db_pass` for the db user:

```shell
openssl rand -hex 16 | ansible-vault encrypt_string --vault-id vault.key --stdin-name 'db_pass'
```

This playbook configures the production server, setting the
`$HOME/.erlang.cookie` file on the web server and generates a Conform file at
`/etc/deploy-template/deploy_template.conf` with the other vars:

```shell
ansible-playbook --vault-password-file vault.key -u $USER -v -l web-servers playbooks/config-web.yml -D
```

This playbook configures `config/prod.secret.exs` on the build server.

```shell
ansible-playbook --vault-password-file vault.key -u $USER -v -l build-servers playbooks/config-build.yml -D
```

TODO: link to config blog post when it's live

## Database

For a real app, you will generally need a database. In the simple scenario, a
single server is used to build and deploy the app, and also runs the db.

Log into the build environment.

After building the release, but before deploying the code, update the db to
match the code:

```shell
scripts/db-migrate.sh
```

That script runs:

```shell
MIX_ENV=prod mix ecto.migrate
```

Surprisingly, the same process also works when we are deploying in an AWS cloud
environment. Create a build instance in the VPC private subnet which has
permissions to talk to the RDS database. Run the Ecto commands to migrate the
db, build the release, then do a Blue/Green deployment to the ASG using AWS
CodeDeploy.

# Changes

Following are the steps used to set up this repo. You can do the same to add
it to your own project.

It all began with a new Phoenix project:

```shell
mix phx.new deploy_template
```

## Set up distillery

Generate initial `rel` files:

```shell
mix release.init
```

Modify `rel/config.exs` to set the cookie from a file and update `vm.args.eex`
to tune the VM settings.

## Set up ASDF

Add the `.tool-versions` file to specify versions of Elixir and Erlang.

## Configure for running in a release

Edit `config/prod.exs`

Uncomment this so Phoenix will run in a release:

```elixir
config :phoenix, :serve_endpoints, true
```

## Add Ansible

Add the tasks to set up the servers and deploy code, in the `ansible`
directory. Configure the vars in the playbooks to match your app name.

To make it easier to run, this repository contains local copies
of roles from Ansible Galaxy in `roles.galaxy`. To update them, run:

```shell
ansible-galaxy install --roles-path roles.galaxy -r install_roles.yml
```

## Add mix tasks for local deploy

Add `lib/mix/tasks/deploy.ex`

## Add Conform for configuration

Add [Conform](https://github.com/bitwalker/conform) to `deps` in `mix.exs`:

```elixir
 {:conform, "~> 2.2"}
```

Generate schema to the `config/deploy_template.schema.exs` file.

```elixir
mix conform.new
```

Generate a sample `deploy_template.dev.conf` file:

```elixir
mix conform.configure
```

Integrate with Distillery, by adding `plugin Conform.ReleasePlugin`
to `rel/config.exs`:

```elixir
release :deploy_template do
  set version: current_version(:deploy_template)
  set applications: [
    :runtime_tools
  ]
  plugin Conform.ReleasePlugin
end
```

## Add shutdown_flag library

This supports restarting the app after deploying a release [without needing
sudo permissions](https://www.cogini.com/blog/deploying-elixir-apps-without-sudo/).

Add [shutdown_flag](https://github.com/cogini/shutdown_flag) to `mix.exs`:

    {:shutdown_flag, github: "cogini/shutdown_flag"},

Add to `config/prod.exs`:

```elixir
config :shutdown_flag,
  flag_file: "/var/tmp/deploy/deploy-template/shutdown.flag",
  check_delay: 10_000
```
