**T1 — Control node & inventory**\
Stand up an Ansible control node, build a static inventory targeting `178.104.200.64`, connect over SSH with a key (no passwords). Verify with `ansible all -m ping`.

*Solution*\
Create an ssh key for ansible
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_ansible -C dum ansible
# do not create a passphrase so that ansible can automatically run commands without me but keep the key very secure
# copy the ansible public key to the remote server
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub 178.104.200.64
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub -o IdentityFile=~/.ssh/id_ed25519_work dumebi@178.104.200.64
```

Create git repo

Install ansible on local machine
```bash
sudo apt update
sudo apt install ansible
```
Create inventory file in git repo folder
``` bash
vim inventory
178.104.200.64 ansible_user=dumebi
```

Run ansible ad hoc command while in git repo
```bash
ansible all --key-file ~/.ssh/id_ed25519_ansible -i inventory -m ping
```

---

**T2 — Idempotent deploy playbook**\
Rebuild the round-1 deploy script as a real playbook — `template`/`copy`/`service`/`cron` modules, no raw `shell:` hacks. A second run must report `changed=0`.

*Solution*\
Create the playbook

Run the playbook
```bash
ansible-playbook --ask-become-pass deploy_script_cron.yaml
# enter become password
```



View the original content

```bash
#/usr/local/bin/healthping.sh
#!/bin/bash
echo "$(date "+%Y-%m-%d %H:%M:%S") - health check start" >> /var/log/healthping.log
notifyping.sh "healthcheck ran" 2>> /var/log/healthping.log
echo "$(date "+%Y-%m-%d %H:%M:%S") - health check done" >> /var/log/healthping.log
```

```bash
#/usr/local/bin/notifyping.sh
#!/bin/bash
echo "$(date "+%Y-%m-%d %H:%M:%S") PING-NOTIFY: $1" >> /var/log/healthping-notify.log
```

---
**T3 — Live bug: the PATH strikes back**\
There's a cron job on the box right now (`healthping.sh`, every 5 minutes) failing exactly the way `notify.sh` did in round 1 — works fine run by hand, fails under cron's minimal PATH. This time, don't SSH in and hand-fix it: find it, then write an idempotent Ansible task that corrects it. Prove it self-heals — revert the bug by hand, re-run your playbook, confirm it fixes itself again without you touching the box directly.

```yaml
---
# health ping yaml
- name: Overwrite Cron Healthscript
  hosts: all
  become: true

  tasks:
  - name: Copy Correct Healthscript Content
    ansible.builtin.template:
      src: "{{ playbook_dir }}/../templates/health_ping.sh.j2"
      dest: /usr/local/bin/healthping.sh
```

Create the template file
``` bash
#!/bin/bash
echo "$(date '+%Y-%m-%d %H:%M:%S') - health check start" >> /var/log/healthping.log
/usr/local/bin/notifyping.sh "healthcheck ran" 2>> /var/log/healthping.log
echo "$(date '+%Y-%m-%d %H:%M:%S') - health check done" >> /var/log/healthping.log
```

Run the Health Ping playbook
```bash
ansible-playbook -i inventory --ask-become-pass playbooks/T3.yaml --private-key ~/.ssh/id_ed25519_ansible
# enter become password
```

Run 
```bash
sudo EDITOR=vim visudo
# add dumebi ALL=(ALL) NOPASSWD: ALL to the file
```
---

### Structure & data

**T4 — Roles refactor**
Turn a monolithic playbook into a proper role (`tasks/`, `handlers/`, `templates/`, `vars/`, `defaults/`). Use a Jinja2-templated config file, with a handler that restarts the service only when that config actually changes.

*Solution*\
We will simply restart cron when my conf file changes. The conf file will contain random information. Nth serious

```yaml
---

- name: Restart service on conf change
  hosts: all
  become: true
  vars:
    - db_user: "Jane"

  tasks:
   - name: Change conf file
     ansible.builtin.template:
      src: "{{ playbook_dir }}/../templates/T4.conf.j2"
      dest: /home/dumebi/T4.conf
     notify: Restart service
  
  handlers:
   - name: Restart service
     ansible.builtin.service:
      service: cron
      state: restarted
```
Setting template file
```yaml
db_user={{ db_user }}
```

---

**T5 — Live bug: exposed secret**
`/etc/appconfig/db_credentials.conf` exists on the box right now, world-writable, holding a real-looking DB credential in plaintext. Find it, then write an Ansible task (`file`/`template` module) that locks down ownership and permissions properly. Your task must never print the secret's contents anywhere — including in `-v` output.

```bash
---

- name: Change Config Permissions
  hosts: all
  become: true

  tasks:
  - name: Changing the file permissions
    file:
     path: /etc/appconfig/db_credentials.conf
     owner: root
     group: root
     mode: '0640'
    no_log: true
```

**T6 — Vault**
Encrypt a secret with `ansible-vault`, reference it in a playbook via a template, and set `no_log: true` correctly on the task that touches it. It should never appear in plaintext in output or logs — that's what gets checked.

*Solution*\
Create a seccret file in project directory
```bash
mkdir vars && vim vars/mysecrets.yaml
```

Create a vault password to encrypt the files in vars
```bash
openssl rand -base64 32 > .vault_pass
```

Encrypt the files with the password
```bash
ansible-vault encrypt vars/my_secrets.yaml --vault-password-file .vault_pass
```

Create the playbook and template files
```yaml
---

- name: Logging into DB
  hosts: all
  become: true
  vars_files:
    - "{{ playbook_dir }}/../vars/my_secrets.yaml"
  tasks:
  - name: Load the database password
    ansible.builtin.template:
     src: "{{ playbook_dir }}/../templates/T6.conf.j2"
     dest: /usr/local/bin/T6.conf
     owner: root
     group: root
     mode: '0644'
    no_log: true
```

```yaml
DB_PASSWORD={{ db_password }}
```

Run the playbook with the password file
```bash
ansible-playbook --ask-become-pass playbooks/T6.yaml --private-key ~/.ssh/id_ed25519_ansible --vault-password-file .vault_pass
```

### Control flow

**T7 — Handlers & notify chains**
Build a change → reload → restart → notify chain. Handlers should fire only when something actually changed, never on every run.

*Solution*\
By default, handlers only reruns when sth changes
```yaml
- name: Apply security rules for SSH
  hosts: all
  become: true

  tasks:
    - name: Configure SSH hardening
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^#?PermitRootLogin', line: 'PermitRootLogin prohibit-password' }
        - { regexp: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
      notify: Restart SSH

  handlers:
    - name: Restart SSH
      ansible.builtin.systemd:
        name: sshd
        state: restarted
```

---

**T8 — Facts, conditionals, loops**
Gather facts and apply a task conditionally based on real host state (`when:`), and loop correctly over a list of items instead of repeating tasks.

*Solution*\
We shall run a task that prints some things depending on the value of a simple variable

```yaml

- name: Loop over text
  hosts: all
  become: true
  gather_facts: true
  vars:
   package_manager: "apt"
   text:
     - "apple"
     - "banana"
     - "cucumber"

  tasks:
    - name: Run Loop Based on Package Manager
      ansible.builtin.debug:
        msg: "{{ item }}"
      loop: "{{ text }}"
      when: ansible_facts['pkg_mgr'] == package_manager 
      
```

**T9 — Live bug: a service that should be running isn't**
`healthbeat.service` is installed on the box but disabled and stopped — nothing is currently checking for that. Write an Ansible task that detects the state and corrects it (`systemd` module: enabled + started). Then prove resilience: stop it by hand again, re-run your playbook, confirm it comes back up without manual intervention.

```yaml
---

- name: Restart Healthbeat Service
  hosts: all
  become: true

  tasks:
   - name: Get service facts
     ansible.builtin.service_facts:

  - name: Show Healthbeat State
    ansible.builtin.debug:
     var: ansible_facts.services["healthbeat.service"].state

  - name: Ensure Healthbeat Service is Running
    ansible.builtin.systemd:
     name: healthbeat
     state: started
     enabled: true
```

---

**T10 — Codify the round-1 hardening**
Turn your manual SSH/firewall hardening from round 1 into an idempotent role. It should reproduce the exact same secure baseline on a fresh box with zero manual SSH work.

*Solution*
Line in file module helps us to find and replace a line
Notify calls the handler that was defined in the handlers section

```yaml
---
- name: Apply security rules for SSH
  hosts: all
  become: true

  tasks:
    - name: Configure SSH hardening
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^#?PermitRootLogin', line: 'PermitRootLogin prohibit-password' }
        - { regexp: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
      notify: Restart SSH

     - name: Set incoming policies to deny
      community.general.ufw:
        direction: incoming
        default: deny

    - name: Set default outgoing to allow
      community.general.ufw:
        direction: outgoing
        default: allow

    - name: Allow SSH connections
      community.general.ufw:
        rule: allow
        port: '22'
        proto: tcp
        comment: 'Allow SSH access'

    - name: 'Allow connections to port 8081'
      community.general.ufw:
        rule: allow
        port: '8081'
        proto: tcp
        comment: 'Allow 8081 access'

    - name: Enable UFW
      community.general.ufw:
        state: enabled

  handlers:
    - name: Restart SSH
      ansible.builtin.systemd:
        name: sshd
        state: restarted
```

Find out what this means
```
[WARNING]: Module remote_tmp /root/.ansible/tmp did not exist and was created with a mode of 0700, this may cause issues when
running as another user. To avoid this, create the remote_tmp dir with the correct permissions manually
```

Using roles:
```bash
# tip: you can setup roles using ansible galaxy
ansible-galaxy init roles/T10_base
```

---

**T11 — Drift detection**
Set up a scheduled convergence run (`ansible-pull` or a cron'd push) against a setting that gets manually drifted out of band — e.g. someone flips a firewall rule by hand. The next scheduled run should silently self-heal it. Document how you verified the correction actually happened.

*Solution*\
I shall use the playbooks that already exist in my github repo like maybe the one for SSH hardeing in T10.yaml.
We can git clone into the review playbook folder because Ihave an inventory file that sets connection to local. 

I used the absolute path to my inventory file
```bash
#dry run
ansible-pull -U https://github.com/DumebiD/ansible-week1.git -C main -i /home/dumebi/review-playbook/inventory --check --diff playbooks/T10.yaml

#actual run
ansible-pull -U https://github.com/DumebiD/ansible-week1.git -C main -i /home/dumebi/review-playbook/inventory playbooks/T10.yaml

# run with --only-if-changed to schedule the task with cron when there is a new commit

ansible-pull -U https://github.com/DumebiD/ansible-week1.git -C main -i /home/dumebi/review-playbook/inventory --only-if-changed playbooks/T10.yaml 2>&1 | tee /var/log/ansible-pull.log

# in crontab, set to run every hour
* */1 * * * ansible-pull -U https://github.com/DumebiD/ansible-week1.git -C main -i /home/dumebi/review-playbook/inventory --only-if-changed playbooks/T10.yaml 2>&1 | sudo tee /var/log/ansible-pull-ssh.log
```
---

**T12 — `--check`/`--diff` discipline**
There's a review-only playbook waiting for you at `/home/dumebi/review-playbook/nightly_cleanup.yml` on the box, meant to prune old entries from `/srv/releases/` (three sample release folders are already there — two old, one current). It looks reasonable at a glance. Do not run it for real. Run it with `--check --diff` first, read the predicted output carefully, and figure out exactly what it would actually do versus what it claims to do (the `run_cleanup` flag and the "keep releases newer than N days" logic are worth looking at closely). Write up what the dry run caught and how you knew — you should be able to point at the specific line that's wrong.

*Solution*\
The original file contains:
```yaml
---
- name: Nightly release cleanup
  hosts: all
  become: true
  vars:
    days_to_keep: 30

  tasks:
    - name: Announce cleanup run
      when: run_cleanup | default(false)
      ansible.builtin.debug:
        msg: "Running nightly cleanup, keeping releases newer than {{ days_to_keep }} days"

    - name: Remove old release directories
      ansible.builtin.file:
        path: "/srv/releases/{{ item }}"
        state: absent
      loop: "{{ lookup(\"ansible.builtin.fileglob\", \"/srv/releases/*\") | map(\"basename\") | list }}"
```

Dry run output
```
PLAY [Nightly release cleanup] ******************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************
ok: [178.104.200.64]

TASK [Announce cleanup run] *********************************************************************************************************
skipping: [178.104.200.64]

TASK [Remove old release directories] ***********************************************************************************************
[WARNING]: Unable to find '/srv/releases' in expected paths (use -vvvvv to see paths)
skipping: [178.104.200.64]

PLAY RECAP **************************************************************************************************************************
178.104.200.64             : ok=1    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

The run cleanup variable is set to false when it's value cannot be loaded, hence why the debug task is skipped.

Fileglob looks up files on the control node (my laptop) based on the pattern. It doesn't find directories by default. I guessed that after the array kept returning empty.

So use find module instead

```yaml
---
- name: Nightly release cleanup
  hosts: all
  become: true
  vars:
    days_to_keep: 30

  tasks:
    - name: Announce cleanup run
      when: run_cleanup | default(false)
      ansible.builtin.debug:
        msg: "Running nightly cleanup, keeping releases newer than {{ days_to_keep }} days"

    - name: Remove old release directories
      ansible.builtin.file:
        path: "/srv/releases/{{ item }}"
        state: absent
      loop: "{{ lookup(\"ansible.builtin.fileglob\", \"/srv/releases/*\") | map(\"basename\") | list }}"
```


```yaml
---
- name: Nightly release cleanup
  hosts: all
  become: true
  vars:
    days_to_keep: 30

  tasks:
    - name: Announce cleanup run
      when: run_cleanup | default(true)
      ansible.builtin.debug:
        msg: "Running nightly cleanup, keeping releases newer than {{ days_to_keep }} days"

    - name: Return folder names
      ansible.builtin.find:
        paths: /srv/releases
        file_type: directory
        age: "{{ days_to_keep }}d"
        age_stamp: mtime
        recurse: true
      register: found_dirs

    - name: Debug find lookup
      ansible.builtin.debug:
        msg: "The directories are {{ found_dirs.files }}"

    - name: Remove old release directories
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: absent
      loop: "{{ found_dirs.files }}"
      when: (item.path | basename != 'releases')

```