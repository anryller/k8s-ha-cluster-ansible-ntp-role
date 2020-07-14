# k8s-ha-cluster-ansible

```
ansible-playbook -i inventory/test_env.ini playbooks/prepare_hosts.yml
ansible-playbook -i inventory/test_env.ini playbooks/deploy_ha_cluster.yml
```
Add Worker node
```
ssh-copy-id root@k8s-node3.test.lab

cd k8s-ha-cluster-ansible
add host to inventory/test_env.ini in section [k8s_workers]
k8s-node3.test.lab ansible_hostname=192.168.100.56

run in dry-run mode, without making any change
ansible-playbook -i inventory/test_env.ini --limit k8s-node3.test.lab playbooks/add_worker_node.yml --check
```
