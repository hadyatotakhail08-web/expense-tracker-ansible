# expense-tracker-ansible — server setup, automated and provable

These Ansible playbooks turn a fresh Ubuntu server into a fully working [expense-tracker](https://github.com/hadyatotakhail08-web/expense-tracker) server: nginx configured as a reverse proxy, the app deployed and running. The difference between this and a set of notes is that these playbooks prove themselves: they're safe to rerun, and they can repair damage.

**For non-engineers:** Ansible is a tool that runs a list of setup instructions ("install these packages, write this config file, restart nginx") on a server automatically, from my laptop. Instead of running 20 commands by hand, I run one, and it behaves the same way every time.

## What it does

| Piece | Purpose |
| --- | --- |
| `inventory` | which server to configure, and how to reach it |
| `ansible.cfg` | project settings (which inventory, which user) |
| `site.yml` | the entry point — runs all the roles below |
| `roles/common` | base setup: packages, system basics |
| `roles/app` | nginx reverse proxy + the app itself |
| Jinja2 templates | config files generated from variables, so values live in one place instead of being hard-coded |

## Proof 1: safe to rerun (idempotence)

The first run reported `changed=11` (it did 11 things). The second run reported `changed=0` — nothing to do, server already correct. A playbook that "changes" things every run isn't automation, it's a gamble. The zero proves it's safe.

## Proof 2: it heals

I deliberately deleted an nginx config symlink to simulate damage. Ran the playbook again: it repaired exactly that one thing (`changed=1`) and touched nothing else. Surgical, not shotgun.

## The outage this repo taught me about

After a break-and-heal test, `systemctl status` said nginx was running — but the site was down. Checking actual listening sockets with `ss -tlnp` showed nothing on port 443. The repair had restored the right config *file*, but the *process* was never restarted, so the old, broken state kept running.

The fix went into the playbook: the task now notifies a handler that restarts nginx whenever the config changes. The lesson became a rule: **a process being alive is not proof a service works — verify with a real request.**

## Mistakes I made here

- Typed the inventory and `ansible.cfg` into the terminal instead of saving them as files. Nothing worked until they existed as real files — the shell doesn't save your pastes.
- Confused controller and target machines: Ansible runs on my Mac (the controller) and configures the server (the target). Every path mistake I made came from forgetting which machine I was standing on.

## Using it

```
ansible-playbook -i inventory site.yml
```

## The other repos in this platform

- [expense-tracker](https://github.com/hadyatotakhail08-web/expense-tracker) — the app, CI/CD and monitoring
- [expense-tracker-infra](https://github.com/hadyatotakhail08-web/expense-tracker-infra) — the Terraform infrastructure
