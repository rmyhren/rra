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
* `-i` specifies the inventory file
* `-m` specifies the module to use (command, shell, etc.)
* `-a` provides arguments to the module

### Exercise 2 – Create a file

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

### Exercise 3 – Safe Patching

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

### Exercise 4 - Deploy Zabbix agent

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





