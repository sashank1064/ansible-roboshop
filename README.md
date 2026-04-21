# ansible-roboshop

> Ansible-driven deployment of the 11-service RoboShop microservices platform on AWS EC2.

![Ansible](https://img.shields.io/badge/Ansible-EE0000?logo=ansible&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-CB171E?logo=yaml&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?logo=amazonaws&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black)

## What this is

The Ansible version of the RoboShop deployment — same 11 services (`mongodb`, `redis`, `mysql`, `rabbitmq`, `catalogue`, `user`, `cart`, `shipping`, `payment`, `dispatch`, `web`), now provisioned through flat Ansible playbooks instead of bash.

This is the "before the roles refactor" step: everything works, but logic lives in playbooks rather than reusable units. The [`ansible-roboshop-roles`](https://github.com/sashank1064/ansible-roboshop-roles) repo shows the refactor.

## Why rewrite the shell version in Ansible?

| Concern | Bash | Ansible |
|---|---|---|
| Idempotency | Manual (if/else around every step) | Built in (modules are declarative) |
| State inspection | None | `--check` and `--diff` |
| Parallelism across hosts | `xargs`, ssh loops | Native `forks` |
| Inventory management | Static scripts | Dynamic inventory, group vars |
| Readability for reviewers | Medium | High |

## Repo layout

```
.
├── inventory.ini             # static inventory of service hosts
├── ansible.cfg               # forks, become, ssh config
├── group_vars/
│   └── all.yml               # shared vars (app version, DNS zone, etc.)
├── mongodb.yml               # data-tier playbooks
├── redis.yml
├── mysql.yml
├── rabbitmq.yml
├── catalogue.yml             # app-tier playbooks
├── user.yml
├── cart.yml
├── shipping.yml
├── payment.yml
├── dispatch.yml
└── web.yml                   # nginx + reverse proxy config
```

## Running it

```bash
# Install collections
ansible-galaxy collection install -r requirements.yml

# Dry-run a single service
ansible-playbook -i inventory.ini catalogue.yml --check --diff

# Deploy every service (order matters: data tier first)
ansible-playbook -i inventory.ini mongodb.yml redis.yml mysql.yml rabbitmq.yml
ansible-playbook -i inventory.ini catalogue.yml user.yml cart.yml shipping.yml payment.yml dispatch.yml
ansible-playbook -i inventory.ini web.yml
```

## Design choices

- **Tasks grouped by service, not by function.** A developer debugging `payment` reads one file, not six.
- **`become: yes` at play level, not task level.** Fewer surprises when tasks are re-used.
- **Handlers for restarts.** Service reloads fire only when config actually changed.
- **Tags per service phase.** `--tags install,config,service` lets me iterate on one phase at a time.
- **No hard-coded secrets.** Sensitive values come from Ansible Vault or AWS SSM.

## What I'd change in production

- Pull host inventory from AWS EC2 dynamic inventory plugin instead of static `inventory.ini`
- Add Molecule tests for each play
- Gate the data-tier plays behind a CI approval step
- Replace the `web` Nginx config with an ALB + target groups

## Progression

1. [`shell-roboshop`](https://github.com/sashank1064/shell-roboshop) — bash baseline
2. **`ansible-roboshop`** ← you are here
3. [`ansible-roboshop-roles`](https://github.com/sashank1064/ansible-roboshop-roles) — refactored to reusable roles
4. [`terraform-multi-env`](https://github.com/sashank1064/terraform-multi-env) — infra layer

---

Part of my DevOps portfolio. Suggestions welcome.
