# Examples of usage

```sh
ansible-playbook -i '192.168.0.221,192.168.0.222,' /etc/ansible/linux-cl-integration.yaml

ansible-playbook -i '192.168.0.221,192.168.0.222,' /etc/ansible/linux-cl-integration.yaml \
    --extra-vars autojoin='False'

ansible-playbook -i '192.168.0.221,192.168.0.222,' /etc/ansible/linux-cl-integration.yaml \
    --extra-vars autojoin='False' \
    --extra-vars taget='*.222'
```


