# Examples of usage

```sh
ansible-playbook -i '192.168.0.221,192.168.0.222,' /etc/ansible/linuxcli-integration.yaml

ansible-playbook -i '192.168.0.221,192.168.0.222,' /etc/ansible/linuxcli-integration.yaml \
    --extra-vars autojoin='False'

ansible-playbook -i '192.168.0.221,192.168.0.222,' /etc/ansible/linuxcli-integration.yaml \
    --extra-vars autojoin='False' \
    --extra-vars target='*.222'
```


