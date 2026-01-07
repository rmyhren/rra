# This is the repository for RRA / NTA coopertaion
The repo contains both code for production use and training material for workshops with RRA

# Test code / workshops

## Production code
You find it in ansible/production

### Exercise 1 - Run a command on another server

**Description:** This exercise introduces the basics of running ad-hoc commands on remote servers using Ansible. You'll learn how to execute simple commands without writing a playbook.

**Task:**
* Run the `uptime` command on all test servers
* Check the disk usage with `df -h`
* View system information with `uname -a`

**Step-by-step:**

1. Navigate to the workshop directory:
   ```bash
   cd ansible/workshop
   ```

2. Run a simple command on all test servers:
   ```bash
   ansible --ask-pass -i inventories/test/hosts.ini test -m command -a "uptime"
   ```
   If you've never connected to these servers before, you may need to prefix the command with ANSIBLE_HOST_KEY_CHECKING=False the first time you connect.

3. Check disk usage:
   ```bash
   ansible --ask-pass -i inventories/test/hosts.ini test -m command -a "df -h"
   ```

4. Get system information:
   ```bash
   ansible --ask-pass -i inventories/test/hosts.ini test -m shell -a "uname -a"
   ```

**Key concepts:**
* `--ask-pass` prompts for the SSH password
* `-i` specifies the inventory file
* `-m` specifies the module to use (command, shell, etc.)
* `-a` provides arguments to the module

### Exercise 2 - Setup SSH Key Authentication

**Description:** This exercise teaches you how to set up passwordless SSH authentication using SSH keys. This is a best practice for automation and eliminates the need to use `--ask-pass` in every command. You'll generate an SSH key pair (if you don't have one) and copy your public key to the remote servers.

**Task:**
* Generate an SSH key pair on your local machine (if you don't have one)
* Copy your public SSH key to all test servers
* Test passwordless authentication

**Step-by-step:**

1. Check if you already have an SSH key:
   ```bash
   ls -la ~/.ssh/id_rsa*
   ```
   If you see `id_rsa` and `id_rsa.pub`, you already have a key pair and can skip to step 3.

2. Generate a new SSH key pair (if needed):
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
   ```
   Press Enter to accept the default file location and optionally set a passphrase.

3. Use Ansible to copy your SSH public key to all test servers:
   ```bash
   cd ansible/workshop
   ansible --ask-pass -i inventories/test/hosts.ini test -m authorized_key -a "user=YOUR_USERNAME key='{{ lookup('file', '~/.ssh/id_rsa.pub') }}' state=present" -b
   ```
   Replace `YOUR_USERNAME` with your actual username on the remote servers.
   
   You'll be prompted for the SSH password one last time.

4. Test passwordless authentication (no `--ask-pass` needed):
   ```bash
   ansible -i inventories/test/hosts.ini test -m ping
   ```

5. Run a command without password prompt:
   ```bash
   ansible -i inventories/test/hosts.ini test -m command -a "whoami"
   ```
   
   This should return your username (on the remote servers) without asking for a password.

**Alternative method using ssh-copy-id:**

If you prefer a more traditional approach, you can use `ssh-copy-id` for each server:

```bash
# For each server in your inventory
ssh-copy-id username@servername
```

Then verify with Ansible:
```bash
ansible -i inventories/test/hosts.ini test -m ping
```

**Key concepts:**
* **SSH keys** provide secure, passwordless authentication
* **authorized_key module** manages SSH public keys on remote servers
* **lookup() function** reads files on the Ansible control node
* Once configured, you no longer need `--ask-pass` for subsequent commands
* This is essential for automation and scheduled tasks

**Troubleshooting:**
* If you get "Permission denied", ensure the SSH key was copied correctly
* Check that `~/.ssh/authorized_keys` exists on the remote server with correct permissions (600)
* Verify your private key has correct permissions: `chmod 600 ~/.ssh/id_rsa`

### Exercise 3 – Create a file

**Description:** Learn how to create and manage files on remote servers using Ansible playbooks. This exercise covers basic file operations and introduces playbook structure.

**Task:**
* Create a file called `/tmp/ansible-test.txt` with specific content
* Set appropriate permissions and ownership
* Verify the file was created successfully

**Step-by-step:**

1. Navigate to the workshop directory (if not already there):
   ```bash
   cd ansible/workshop
   ```

2. Create a playbook file called `playbooks/create_file.yml`:
   ```yaml
   ---
   - name: Create a test file
     hosts: test
     become: yes
     tasks:
       - name: Create file with content
         copy:
           content: "This file was created by Ansible\n"
           dest: /tmp/ansible-test.txt
           owner: root
           group: root
           mode: '0644'
       
       - name: Verify file exists
         stat:
           path: /tmp/ansible-test.txt
         register: file_status
       
       - name: Display file status
         debug:
           msg: "File exists: {{ file_status.stat.exists }}"
   ```

3. Run the playbook:
   ```bash
   ansible-playbook -i inventories/test/hosts.ini playbooks/create_file.yml
   ```

4. Verify the file was created:
   ```bash
   ansible -i inventories/test/hosts.ini test -m command -a "cat /tmp/ansible-test.txt"
   ```

**Key concepts:**
* Playbook structure with hosts, tasks, and modules
* Using `copy` module to create files with content
* Using `stat` module to check file status
* Using `become: yes` for privilege escalation

### Exercise 4 – Safe Patching

**Description:** This exercise teaches safe, production-grade operating system patching on RHEL servers using Ansible. The goal is to ensure that updates are applied in a controlled manner, avoiding service disruption and alert noise. Participants learn how to limit blast radius, detect when a reboot is actually required, and rely on automation rather than manual intervention.

**Task:**
* Run patching only against test servers
* Patch only one server at a time
* Reboot the server only if required

**Step-by-step:**

1. Navigate to the workshop directory (if not already there):
   ```bash
   cd ansible/workshop
   ```

2. Review or create the playbook file `playbooks/patch.yml`:
   ```yaml
   ---
   - name: Safe patching with controlled reboot
     hosts: test
     serial: 1
     become: yes
     tasks:
       - name: Update all packages
         dnf:
           name: "*"
           state: latest
         register: dnf_output
       
       - name: Check if reboot is required
         command: needs-restarting -r
         register: reboot_required
         failed_when: false
         changed_when: reboot_required.rc == 1
       
       - name: Reboot if required
         reboot:
           msg: "Rebooting after patching"
           reboot_timeout: 600
         when: reboot_required.rc == 1
       
       - name: Wait for system to be available
         wait_for_connection:
           delay: 30
           timeout: 300
         when: reboot_required.rc == 1
   ```

3. Run the patching playbook:
   ```bash
   ansible-playbook -i inventories/test/hosts.ini playbooks/patch.yml --limit test
   ```

4. To run against a single server:
   ```bash
   ansible-playbook -i inventories/test/hosts.ini playbooks/patch.yml --limit testserver01
   ```

**Key concepts:**
* `serial: 1` ensures only one server is patched at a time (limits blast radius)
* `dnf` module updates packages safely
* `needs-restarting -r` detects if reboot is required (exit code 1 means reboot needed)
* Conditional reboot only when necessary (avoids unnecessary downtime)
* `wait_for_connection` ensures server is back online before continuing

**Answer:**
* Use `serial: 1` to patch one server at a time
* Use the `dnf` module for package updates
* Use `needs-restarting -r` to detect reboot requirement

**Command:**
```bash
ansible-playbook -i inventories/test/hosts.ini playbooks/patch.yml --limit test
```

### Exercise 5 - Deploy Zabbix agent

**Description:** This exercise demonstrates how to use Ansible roles to deploy and configure the Zabbix monitoring agent on remote servers. You'll learn about role structure, variables, templates, and handlers to create reusable automation.

**Task:**
* Install the Zabbix agent package on test servers
* Configure the agent with the correct Zabbix server address
* Deploy a configuration file using Jinja2 templates
* Ensure the service is enabled and running

**Step-by-step:**

1. Navigate to the workshop directory (if not already there):
   ```bash
   cd ansible/workshop
   ```

2. Review the role structure in `roles/zabbix_agent/`:
   - `defaults/main.yml` - Default variables
   - `tasks/main.yml` - Main installation and configuration tasks
   - `templates/zabbix_agentd.conf.j2` - Configuration template
   - `handlers/main.yml` - Service restart handler

3. Set variables in `roles/zabbix_agent/defaults/main.yml`:
   ```yaml
   ---
   zabbix_server: "10.0.0.100"
   zabbix_server_active: "10.0.0.100"
   zabbix_agent_hostname: "{{ ansible_hostname }}"
   ```

4. Create a playbook called `playbooks/deploy_zabbix.yml`:
   ```yaml
   ---
   - name: Deploy Zabbix Agent
     hosts: test
     become: yes
     roles:
       - zabbix_agent
   ```

5. Run the playbook:
   ```bash
   ansible-playbook -i inventories/test/hosts.ini playbooks/deploy_zabbix.yml
   ```

6. Verify the Zabbix agent is running:
   ```bash
   ansible -i inventories/test/hosts.ini test -m command -a "systemctl status zabbix-agent"
   ```

7. Override variables for specific environments:
   ```bash
   ansible-playbook -i inventories/test/hosts.ini playbooks/deploy_zabbix.yml -e "zabbix_server=10.0.1.50"
   ```

**Key concepts:**
* **Roles** provide reusable, structured automation (tasks, variables, templates, handlers)
* **Templates** use Jinja2 to dynamically generate configuration files
* **Handlers** ensure services are restarted only when configuration changes
* **Variables** can be defined in defaults and overridden at runtime
* Role structure follows Ansible best practices for maintainability

### Additional tasks - Setup inventory from VMware automatically
This is the documentation for setting up inventory from VMware automatically
https://github.com/ansible-collections/vmware.vmware





