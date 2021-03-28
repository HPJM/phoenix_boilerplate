# Server setup

- Once provisioned run: `ansible-playbook -u ubuntu -v -l web_servers playbooks/setup-web.yml -D -i inventory`
- After that, `ubuntu` user won't be available
- Set up app `ansible-playbook -u hugo -v -l web_servers playbooks/deploy-app.yml --skip-tags deploy -D`
- Configure `ansible-playbook -u hugo -v -l web_servers playbooks/config-web.yml -D`
- Set up db `ansible-playbook -u hugo -v -l db_servers playbooks/setup-db.yml -D`
- Build `ansible-playbook -u hugo -v -l build_servers playbooks/setup-build.yml -D`
- Deploy `ansible-playbook -u deploy -v -l web_servers playbooks/deploy-app.yml --tags deploy --extra-vars ansible_become=false -D`

# Build

Assumes building on Ubuntu 20.04 (AWS).

Make sure to change all references to `[app]` to the app name.

## 1. Create deploy user

- [Improvement with Ansible](https://www.cogini.com/blog/managing-user-accounts-with-ansible/)
- Execute commands in `bin/create-deploy-user` script
- Log out of session and `ssh -A` back in as `deploy` (`-A` flag here is to allow copying of local SSH keys, which we need if GH repo is private)
- Test `deploy` user can run commands with sudo: `sudo -s`
- Remove Ubuntu user: `userdel ubuntu -r`
- Then `exit` and log back in as `deploy`

## 2. Check out app source

- `cd ~/build`
- `git clone [repo]` in `/home/deploy/build`
- `cd` into repo

## 3. Copy example systemd file

- `cd [app]`
- `sudo cp [app].service /lib/systemd/system`

## 4. Install deps

- `bin/asdf-deps`
- `bin/asdf`
- Log out and back in
- `bin/asdf-install`
- Check app compiles

## 5. Install and configure Postgres (if DB on same host)

- Install Postgres: `sudo apt-get install postgresql postgresql-contrib -y`
- Log into Postgres session: `sudo -u postgres psql`
- (In psql console) `CREATE DATABASE [name];`
- (In psql console) `CREATE USER [user] WITH PASSWORD '[password]';`
- (In psql console) `GRANT ALL PRIVILEGES ON DATABASE [name] to [user];`

## 6. Set up env vars (TODO: improve config management)

- Add `SECRET_KEY_BASE` (`mix phx.gen.secret`) and `DATABASE_URL` (example: `ecto://hugo:password@localhost:5432/my_app`) as app secrets

## 7. Initialise system (set up groups and users) (ONE time)

- `sudo bin/deploy-init-local`
- Log out and back in

# Deploy

## 1. Git pull (if changes since cloned)

## 2. Build release

- Make sure `releases` key is defined in `mix.exs`
- Run: `MIX_ENV=prod bin/build`

## 3. Extract release to target dir, creating symlink

- `bin/deploy-release`

## 4. Migrate

- `bin/deploy-migrate`

## 5. Restart and test

- `sudo bin/deploy-restart`
- `curl -v http://localhost:4000/`

# Monitoring and other options

## Check status and logs

- `sudo systemctl status [app]`
- `sudo journalctl -r -u [app]`

## Rollbacks

- `bin/deploy-rollback`
- `sudo bin/deploy-restart`
