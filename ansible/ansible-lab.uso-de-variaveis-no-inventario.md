# Ansible LAB - Uso de variáveis em inventário de hosts

__Configuração inicial__

> ansible.cfg

```
[default]
deprecation_warnings = false
interpreter_python = auto_silent
host_key_checking = false
[ssh_connection]
ssh_args = -o "StrictHostKeyChecking=no" -J jumpuser@jumphost1
```

_IMPORTANTE: Considera-se neste lab o uso de um "bastion host" (ProxyJump) 
onde a chave pública "id_rsa" já foi importada no serviço "sshd" 
destino em "jumphost1" para o usuário corrente._


## AD-Hoc Command

Forma unitária e simplificada. Antendar que a vírugula seguinte ao nome do host 
é essencial se utilizado um hostname ao invés de informar um arquivo de inventário.

```
ansible -i host1, -u zeca -k -m shell -a hostname all
```

## Execução múltipla

- Inventário em YAML
- Múltiplas senhas por pool
- Senhas baseadas em variáveis globais

__Preparação do inventário YAML__

// Utilizado apenas formato YAML e não INI, porque o formato INI
// não permite apenas informar a pasta "inventory" e obrigar o 
// ansible a fazer o parser de todos os arquivos, para tal
// seria necessário que todo o conteúdo fosse aglutinado em um 
// único arquivo INI o que torna ruim para grandes massas ou
// para uso de um script/programa gerador de inventário baseado
// em CMDB por exemplo.

Criar string de senha criptografada via Vault

```
ansible-vault encrypt_string
```

> inventory/all.yaml

```
# global
all:
  vars:
    pass_global: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      00000000000000000000000000000000000000000000000000000000000000000000036135356464
      63386438303433363533663111111111111111111111111111111111111111111111111111111111
      1234
```

> inventory/urano.yaml

```
# usuário nominal + auth local
urano:
  vars:
    ansible_ssh: "{{ssh_nominal_login}}"
    ansible_ssh_pass: "{{ssh_nominal_localpassword}}"
  hosts:
    urano1:
      ansible_host: urano.host.intranet
```

> inventory/jupyter.yaml

```
# usuário nominal + auth dominio
jupyter:
  vars:
    ansible_ssh: "{{ssh_nominal_login}}"
    ansible_ssh_pass: "{{ssh_nominal_domainpassword}}"
  hosts:
    jupyter1:
      ansible_host: jupyter1.host.intranet
```

> inventory/mercurio.yaml

```
# usuário fixo + auth via global vars
mercurio:
  vars:
    ansible_user: spock
    ansible_ssh_pass: "{{pass_global}}"
    ansible_ssh_common_args: -o "KexAlgorithms=+diffie-hellman-group1-sha1" -o "StrictHostKeyChecking=no"
  hosts:
    mercurio1:
      ansible_host: mercurio1.host.intranet
```

__Testes : Cenário 1__

Acessar "mercurio" utilizando senha global cadastrada dentro do Vault 
porém respeitando que existe um usuário fixo "spock".

```
ansible -i inventory --ask-vault-pass -m shell -a 'echo "Conectado ao host $(hostname) via login $USER."' mercurio
```

__Testes : Cenário 2__

Acessar "urano" e "jupyter" com usuário nominal e com auth local, 
onde a senha é enviada via variável de ambiente shell.

```
read ssh_nominal_login
read ssh_nominal_localpassword
read ssh_nominal_domainpassword
export ssh_nominal_login ssh_nominal_localpassword ssh_nominal_domainpassword
ansible -i inventory \
  -e "$(env |grep ^ssh_nominal_)" \
  -m shell \
  -a 'echo "Conectado ao host $(hostname) via login $USER."' \
  urano,jupyter
```

__Testes : Cenário 3__

Acessar qualquer ambiente, ao fornecer todas as senhas necessárias (execução massiva)

```
read ssh_nominal_login
read ssh_nominal_localpassword
read ssh_nominal_domainpassword
export ssh_nominal_login ssh_nominal_localpassword ssh_nominal_domainpassword
ansible -i inventory \
  -e "$(env |grep ^ssh_nominal_)" \
  --ask-vault-pass \
  -m shell \
  -a 'echo "Conectado ao host $(hostname) via login $USER."' \
  all
```

__Teste Cenário 4__

// Uso do Vault também para o usuário nominal

Criar o novo arquivo

```
ansible-vault create vault@zeca
```

Conteúdo do arquivo vault

```
ssh_nominal_login=zeca
ssh_nominal_localpassword=xyz
ssh_nominal_domainpassword=qwe
```

Executar o ansible onde a chamada do Vault traz a variável de linha de comando. 
Atentar que a primeira senha solicitada é para "vault@zeca" e a segunda a ser 
enviada à opção "--ask-vault-pass" a qual vai descriptogravar o conteúdo dentro do YAML de inventário.

```
ansible -i inventory \
  -e "$(ansible-vault view vault@zeca)" \
  --ask-vault-pass \
  -m shell \
  -a 'echo "Conectado ao host $(hostname) via login $USER."' \
  all
```
