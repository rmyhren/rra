# This is the repository for RRA / NTA coopertaion
The repo contains both code for production use and training material for workshops with RRA

# Test code / workshops

## Production code
You find it in ansible/production

### Exercise 1 - Run a command on another server

### Exercise 2 – Create a file 

### Exercise 3 – Safe Patching

Description: This exercise teaches safe, production-grade operating system patching on RHEL servers using Ansible. The goal is to ensure that updates are applied in a controlled manner, avoiding service disruption and alert noise. Participants learn how to limit blast radius, detect when a reboot is actually required, and rely on automation rather than manual intervention.

### Exercise 4 - Deploy Zabbix agent

### Additional tasks - Setup inventory from VMware automatically
This is the documentation for setting up inventory from VMware automatically
https://github.com/ansible-collections/vmware.vmware


 


### Task:
* Run patching only against test servers
* Patch only one server at a time
* Reboot the server only if required

### Answer:
* Use serial: 1
* Use the dnf module
* Use needs-restarting -r to detect reboot requirement

```ansible-playbook -i inventories/test/hosts.ini playbooks/patch.yml --limit test```
