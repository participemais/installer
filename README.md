# Instalador do Participe+

Instalador do [Participe+](https://github.com/consul/consul) para ambientes de produção e homologação.


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
git clone https://github.com/juliangutierrez/installer
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

[How to setup Capistrano](https://youtu.be/ZCfPz_c_H6g)

Create your [fork](https://help.github.com/articles/fork-a-repo/)

Setup locally for your [development environment](https://docs.consulproject.org/docs/english-documentation/introduction/local_installation)

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

[How to setup email deliveries](https://youtu.be/9W6txGpe4v4)

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

Once you setup your domain, depending on your SMTP provider, you will have to do two things:

- Update the `server_name` with your domain in `/home/deploy/consul/shared/config/secrets.yml`.
- Update the `sender_email_address` from the admin section (`remote-server-ip-address/admin/settings`)

If your SMTP provider uses an authentication other than `plain`, check out the [Rails docs on email configuration](https://guides.rubyonrails.org/action_mailer_basics.html#action-mailer-configuration) for the different authentation options.

## Staging server

To setup a staging server to try things out before deploying to a production server:

Update your local `hosts` file with the staging server's ip address

```
remote-server-ip-address (maintain other default options)
```

And run the playbook with an extra var "env":

```
ansible-playbook -v consul.yml -i hosts --extra-vars "ansible_sudo_pass=<senha> env=staging"
```

Visit remote-server-ip-address in your browser and you should now see CONSUL running in your staging server.

## SSL with LetsEncrypt

Using https instead of http is an important security configuration. Before you begin, you will need to either buy a domain or get access to the configuration of an existing domain. Next, you need to make sure you have an A Record in the DNS configuration of your domain, pointing to the correponding IP address of your server. You can check if your domain is correctly configured at this url https://dnschecker.org/, where you should see your IP address when searching for your domain name.

Once you have that setup we need to configure the Installer to use your domain in the application.

First, uncomment the `domain` variable in the [configuration file](https://github.com/consul/installer/blob/1.2.0/group_vars/all) and update it with your domain name:

```
#domain: "your_domain.com"
```

Next, uncomment the `letsencrypt_email` variable in the [configuration file](https://github.com/consul/installer/blob/1.2.0/group_vars/all) and update it with a valid email address:

```
#letsencrypt_email: "your_email@example.com"
```

Re-run the installer:

```
ansible-playbook -v consul.yml -i hosts
```

You should now be able to see the application running at https://your_domain.com in your browser.

## Configuration Variables

These are the main configuration variables:

```
# Server Timzone + Locale
timezone: Europe/Madrid
locale: en_US.UTF-8

# Authorized Hosts
ssh_public_key_path: "change_me/.ssh/id_rsa.pub"

#Postgresql
postgresql_version: 9.6
database_name: "consul_production"
database_user: "deploy"
database_password: "change_me"
database_hostname: "localhost"
database_port: 5432

#SMTP
smtp_address:        "smtp.example.com"
smtp_port:           25
smtp_domain:         "your_domain.com"
smtp_user_name:      "username"
smtp_password:       "password"
smtp_authentication: "plain"
```

There are many more variables available check them out [here]((https://github.com/consul/installer/blob/1.2.0/group_vars/all))

## Other deployment options

### Split database from application code

The [`consul` playbook](consul.yml) creates the database on the same server as the application code. If you are using a cloud host that offers managed databases (such as [AWS RDS](https://aws.amazon.com/rds/), [Azure Databases](https://azure.microsoft.com/en-us/product-categories/databases/), or [Google Cloud SQL](https://cloud.google.com/sql/)), we recommend using that instead.

To set up the application by itself:

1. Fork this repository.
1. Specify your database credentials (see the `database_*` [group variables](group_vars/all)) in a [vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html).
1. Run the [`app` playbook](app.yml) instead of the [`consul`](consul.yml) one against a clean server.

```sh
ansible-playbook -v app.yml -i hosts
```

## Licença

Publicado sobre AFFERO GPL v3 ([LICENSE-AGPLv3.txt](LICENSE-AGPLv3.txt))
