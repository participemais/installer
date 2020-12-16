# Instalador do Participe+

Instalador do [Participe+](https://github.com/participemais/consul) para ambientes de produção e homologação.


## Pré requisitos

- Ubuntu 18.04 x64

Accesso ao servidor remoto via ssh com chave pública sem senha.
O usuário padrão é `deploy`. A execução do instalador pressupõe um usuário de igual nome na máquina de origem.

```
ssh root@remote-server-ip-address
```

Update da lista de pacotes

```
sudo apt-get update
```

Python 2.7 instalado no servidor

```
sudo apt-get -y install python-simplejson
```

## Preparando para instalação

Na máquina remota:
```
$ sudo adduser deploy
$ sudo usermod -aG sudo deploy
```

Na máquina local (com usuário deploy):
```
$ ssh-keygen -t rsa
$ ssh-copy-id deploy@server_ip_address
$ ssh deploy@server_ip_address
```


## Executando o instalador

Os comandos abaixo devem ser executados na máquina de origem.

[Install Ansible >= 2.7](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

Download do Ansible playbook

```
git clone https://github.com/participemais/installer
cd installer
```

Criação do arquivo `hosts` local

```
cp hosts.example hosts
```

Atualização do `hosts` com o ip do servidor remoto

```
remote-server-ip-address (maintain other default options)
```

Execução do ansible playbook

```
ansible-playbook -v consul.yml -i hosts --extra-vars "ansible_sudo_pass=<senha>"
```

## Deploys com Capistrano

Para reiniciar o servidor e fazer deploys é preciso configurar o Capistrano.

### Screencast

[Como configurar o capistrano](https://youtu.be/ZCfPz_c_H6g)

Comfigurar localmente seu [ambiente de desenvolvimento](https://docs.consulproject.org/docs/english-documentation/introduction/local_installation)

Criação do `deploy-secrets.yml`

```
cp config/deploy-secrets.yml.example config/deploy-secrets.yml
```

Atualização do `deploy-secrets.yml` com informações do servidor

```
deploy_to: "/home/deploy/consul"
server1: "your_remote_ip_address"
db_server: "localhost"
server_name: "your_remote_ip_address"
```

Deploy para o servidor

```
branch=stable cap production deploy
```

## Dump e Restore da base de dados

```
sudo -u postgres psql
ALTER ROLE deploy Superuser;
\q
pg_dump -U deploy -F t consul_<env> > db.dump
pg_restore -d consul_<env> db.dump -c -U deploy
```
### Anexos

O nome dos arquivos anexos em public/system seguem uma lógica utilizando como parâmtero a chave paperclip_secret_base em secrets.yml. Essa chave não é criada por padrão na instalação, devendo ser copiada da instalação anterior.

## Configuração de Email

### Screencast

[Configurar entrega de emails](https://youtu.be/9W6txGpe4v4)

Para ver o log de erros de email abrir o console do rails (`cd /home/deploy/consul/current/ && bin/rails c -e production`) e procurar pelo último erro da fila ( `Delayed::Job.last.last_error`)

Atualização do arquivo abaixo no servidor
`/home/deploy/consul/shared/config/secrets.yml`

Preencher com as credenciais SMTP:

```
  mailer_delivery_method: "smtp"
  smtp_settings:
    address:              "smtp.example.com"
    port:                 "25"
    domain:               "your_domain.com"
    user_name:            "username"
    password:             "password"
    authentication:       "plain"
    enable_starttls_auto: true
```

Reiniciar o a aplicação

```
cap production deploy:restart
```

Após configurar o domínio atualizar `server_name` em `/home/deploy/consul/shared/config/secrets.yml`.

## Servidor de homologação

Atualizar arquivo `hosts` com o ip do servidor de homologação

```
remote-server-ip-address (manter as outras opções)
```

Executar o play book com a variável "env":

```
ansible-playbook -v consul.yml -i hosts --extra-vars "ansible_sudo_pass=<senha> env=staging"
```

## SSL com LetsEncrypt

Mudar para a branch ssl e executar o instalador. Se o domínio for diferente do padrão, atualizar a variável `domain` no [arquivo de configuração](https://github.com/consul/installer/blob/1.2.0/group_vars/all) com o nome de domínio:

```
#domain: "your_domain.com"
```

Executar o instalador somente com o arquivo app:

```
ansible-playbook -v app.yml -i hosts --extra-vars "ansible_sudo_pass=<senha>"
```

## Licença

Publicado sobre AFFERO GPL v3 ([LICENSE-AGPLv3.txt](LICENSE-AGPLv3.txt))
