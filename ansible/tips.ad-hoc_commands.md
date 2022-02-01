# AD-HOC Exec

Forma unitária e simplificada.

```
ansible -i host1, -u zeca -k -m shell -a hostname all
```

> _Atentar que a vírugula seguinte ao nome do host é essencial se utilizado um hostname ao invés de informar um arquivo de inventário._

# Referências

- [Ansible Docs - Index of all Modules](https://docs.ansible.com/ansible/latest/collections/index_module.html)
- [Ansible Docs - Introduction to ad hoc commands](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html)
