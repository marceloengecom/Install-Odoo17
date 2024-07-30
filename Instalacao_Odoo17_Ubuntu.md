# Instalação Odoo 17 no Ubuntu 22.04



## 1. Pré-requisitos

- Sistema Operacional Ubuntu 22.04 com Python >= 3.10.
- Usuário com privilégios "sudo".
- IP estático (ex: 172.16.15.10) devidamente configurado e com acesso a internet.
- Acesso SSH ao servidor onde será instalado o Odoo. Se estiver usando firewall (ex: UFW), libere coenxões com o serviço ou respectiva porta.


> Nota: Caso seja possível, crie um snapshot do estado atual de sua máquina a fim de que caso haja algum problema, possa ser restaurado com rapidez. 

> Nota: É possível executar todos os comandos a seguir a partir do usuário root, mas por razões de segurança, é recomendad o uso de um usuário diferente com privilégios administrativos "sudo".



## 2. Preparando o Sistema
Faça login em sua máquina com seu usário com permissões administrativas de root e atualize o sistema:

```sh
sudo apt update && sudo apt upgrade -y
```

Definir Fuso Horário:
```sh
sudo timedatectl set-timezone America/Sao_Paulo
```

Python 3.10: O Ubuntu 22.04 já possui nativamente a versão 3.10 do python. Confira a versão, usando o comando a seguir:
```sh
sudo python3 --version
```
> O resultado da saída deste comando deve ser:
> Python 3.10.12
 


## 3. Definir regras de firewall UFW (Uncomplicated Firewall)
Precisaremos que somente as portas 22, 80, 443, 6010, 5432, 8069 e 8072 estejam abertas para acesso externo. 22 é usado para SSH, 80 é para HTTP, 443 é para HTTPS, 6010 é usado para comunicação Odoo, 5432 é usado pelo PostgreSQL, 8069 é usado pelo aplicativo de servidor Odoo e 8072 é Long Polling.

> Nota: Long Polling é uma tecnologia em que o cliente solicita informações do servidor sem esperar uma resposta imediata ou basicamente envolve fazer uma solicitação HTTP a um servidor e, em seguida, manter a conexão aberta para permitir que o servidor responda mais tarde.

Para evitar desconexões involuntárias, vamos manter o firewall UFW desabilitado:
```sh
sudo ufw disable
```

Por padrão, o UFW é configurado para negar todas as conexões de entrada e permitir todas as conexões de saída. Isso significa que qualquer um que tentasse acessar o seu servidor não conseguiria conectar-se, ao passo que os aplicativos dentro do servidor conseguiriam alcançar o mundo exterior.

Vamos definir as regras do seu UFW para essas configurações padrão:
```sh
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Permita conexão com o protocolo SSH:
```sh
sudo ufw allow 22/tcp
```

> Nota: Caso utilize um porta diferente do padrão para algum dos serviços descritos no início dessa etapa, atualize a respectiva regra de firewall. Ex: Se utilizar a porta 2222 para SSH, libere essa porta ao invés da 22.


Com o firewall já configurado para permitir as conexões de entrada via SSH, podemos habilitar o firewall:
```sh
sudo ufw enable
```

> Confirme digitando "y":
> *Command may disrupt existing ssh connections. Proceed with operation (y|n)? y*
>
> A mensagem de saída deste comando, deve ser:
> *Firewall is active and enabled on system startup*


Liberar as demais portas:
```sh
sudo ufw allow 80/tcp
```
```sh
sudo ufw allow 443/tcp
```
```sh
sudo ufw allow 5432/tcp
```
```sh
sudo ufw allow 6010/tcp
```
```sh
sudo ufw allow 8069/tcp
```
```sh
sudo ufw allow 8072/tcp
```

Recarregue o firewall:
```sh
sudo ufw reload
```

> A mensagem de saída deve ser:
> Firewall reloaded


Para conferir quais regras estão ativas, digite o seguinte comando:
```sh
sudo ufw status
```

> A saída deve ser:
> Status: active
> To                         Action      From
> --                         ------      ----
> 22/tcp                     ALLOW       Anywhere                  
> 80/tcp                     ALLOW       Anywhere                  
> 443/tcp                    ALLOW       Anywhere                  
> 5432/tcp                   ALLOW       Anywhere
> 6010/tcp                   ALLOW       Anywhere                  
> 8069/tcp                   ALLOW       Anywhere                  
> 8072/tcp                   ALLOW       Anywhere                  
> 22/tcp (v6)                ALLOW       Anywhere (v6)
> 80/tcp (v6)                ALLOW       Anywhere (v6)             
> 443/tcp (v6)               ALLOW       Anywhere (v6)             
> 5432/tcp (v6)              ALLOW       Anywhere (v6)             
> 6010/tcp (v6)              ALLOW       Anywhere (v6)             
> 8069/tcp (v6)              ALLOW       Anywhere (v6)             
> 8072/tcp (v6)              ALLOW       Anywhere (v6)



## 4. Instalação de ferramentas para instalação das dependências necessárias

Instalar Git, PIP, NPM, NodeJS e as ferramentas requiridas para poder instalar as dependencias do Odoo:
```sh
sudo apt install git python3-pip python3-dev libxml2-dev libxslt1-dev zlib1g-dev libsasl2-dev libldap2-dev build-essential libssl-dev libffi-dev libmysqlclient-dev libjpeg-dev libpq-dev libjpeg8-dev liblcms2-dev libblas-dev libatlas-base-dev gdebi-core python3-wheel python3-venv libxslt-dev libzip-dev -y
```

Instalar dependências WEB (NPM, NodeJS, Less):
```sh
sudo apt install npm -y
```
```sh
sudo ln -s /usr/bin/nodejs /usr/bin/node
```
```sh
sudo npm install -g less less-plugin-clean-css
```
```sh
sudo apt-get install -y node-less
```



## 5. Criar usuário Odoo

Crie um novo usuário chamado **odoo17** com seu respectivo diretório home **/opt/odoo17**. Isso evita os riscos de segurança representados pela execução do Odoo sob o usuário root. Pode ser usado qualquer nome, mas é importante que o usuário PostgreSQL, que será visto mais adiante, deve ter o mesmo nome. Use os seguintes comandos:

```sh
sudo useradd -m -d /opt/odoo17 -U -r -s /bin/bash odoo17
```



## 6. Instalar e Configurar PostgreSQL

Nesta etapa, é necessário configurar o servidor de banco de dados. O Odoo usa o PostgreSQL como banco de dados. Instale o servidor de banco de dados para Odoo usando o comando a seguir:

Instalar o pacote PostgreSQL a partir do repositório padrão do Ubuntu:

```sh
sudo apt install postgresql postgresql-client -y
```

Após completada a instalação, inicie o serviço do banco de dados:
```sh
sudo systemctl start postgresql
```

Caso queira saber a versão instalada, digite o seguinte comando:
```sh
sudo psql -V
```

Crie um usuário PostgreSQL com o mesmo nome previamente criado, em nosso caso, **odoo17**:
```sh
sudo su - postgres -c "createuser --encrypted --createdb --createrole --superuser odoo17"
```

> Nota: Por padrão, o PostgreSQL só permite conexão por soquetes UNIX e conexões loopback. O soquete UNIX é bom se você desejar que o Odoo e o PostgreSQL sejam executados no mesmo servidor, mas se você precisar que Odoo e PostgreSQL executem em servidores diferentes, é necessário configurar o PostgreSQL para aceitar outras conexões de rede.
> Nesse último caso, que não é objetivo deste tutorial, os arquivos *pg_hba.conf* e *postgresql.conf*, necessitarão ser ajustados. 






## 7. Criar pasta para armazenamento de LOGs. Vamos usar o mesmo nome do usuário, usado anteriormente.
```sh
sudo mkdir /var/log/odoo17
```
```sh
sudo chown odoo17:odoo17 /var/log/odoo17
```
```sh
sudo chmod 755 /var/log/odoo17
```



## 8. Instalar Wkhtmltopdf

Os pacotes wkhtmktox fornecem um conjunto de ferramentas de linha de comando de código aberto que podem renderizar HTML em PDF e vários formatos de imagem. Para imprimir relatórios em PDF, você precisará da ferramenta wkhtmltopdf. Odoo 17 requer uma versão superior a 0.12.2 e no **Ubuntu 22.04**, diferente de verões anteriores, o wkHTMLtoPDF foi adicionado ao repositório padrão do sistema operacional.

```sh
sudo apt install wkhtmltopdf
```

Para verificar a versão instalada, digite o seguinte comando:
```sh
sudo wkhtmltopdf -V
```



## 9. Instalação do Odoo

No **Ubuntu 22.04**, podemos instalar o Odoo a partir do repositório padrão, mas neste artigo, vamos instalar o Odoo 17 em um ambiente virtual python e clonar a partir do repositório oficial.

Altere para o usuário do sistema **odoo17**, criado anteriormente:
```sh
sudo su - odoo17
```

> O comando acima mudará o seu usuário (ex: odoo17) e deve levá-lo para a pasta "/opt/odoo17".


Agora, faça o clone do Odoo a partir do repositório oficial do Github para a subpasta "server", indicada no final da linha. O conteúdo deve ficar em "/opt/odoo17/server".
```sh
git clone https://github.com/odoo/odoo --depth 1 --branch 17.0 server/
```

Após concluído o downloaded, crie um novo ambiente virtual Python para a sua instalação:
```sh
cd /opt/odoo17
```

```sh
python3 -m venv venv-odoo17
```

> O comando acima criará uma pasta com o mesmo nome de seu ambiente virtual
> (/opt/odoo17/env-odoo17).


O ambiente virtual está instalado, agora ative-o executando este comando:
```sh
source venv-odoo17/bin/activate
```

> Uma vez executado, o prompt do shell deve ficar como mostrado a seguir:
> 
> *(venv-odoo17) odoo17@seuServidorOdoo:~$*


Atualizar o gerenciador de pacotes PIP
```sh
pip3 install --upgrade pip
```

Instalar todos os módulos python requiridos pelo Odoo com pip:

```sh
pip install wheel && pip install -r server/requirements.txt
```

> Nota: Se for encontrado qualquer erro de compilação durante a instalação, tenha certeza de que você tem todas as requeridas depend~encias instaladas, listadas no início da seção


Caso você não encontre nenhum erro, use o comando abaixo para testar se todos os requisitos declarados no arquivo ***requirements.txt*** foram instalados:
```sh
pip list
```

> Se for encontrado algum erro, vá para o arquivo  ***requirements.txt*** e instale cada pacote um por um com o seu respectivo comando:
> ```sh
> pip install nome_pacote==versão
>```
> 
> Ex: Caso haja alguma falha no pacote *geoip2*:
> ```sh
> pip install geoip2==2.9.0
> ```


Desative o ambiente virtual, usando o seguinte comando:
```sh
deactivate
```
> Note, no prompt do shell, que o ambiente virtual não é mais mostrado e o usuário do Odoo continua ativo
> *odoo17@seuServidorOdoo:~$*



## 10. Módulos de Terceiros

Mesmo que não seja usado nesse momento, é recomendado criar uma pasta exclusiva para armazenamento de módulos personalizados ou de terceiros

```sh
mkdir /opt/odoo17/server/custom-addons
```

> Nota: Lembre-se de sempre adicionar o caminho completo desta pasta no arquivo de configuração do Odoo, na diretiva "addons_path", ao qual veremos mais adiante.


Volte para o seu usuário sudo:

```sh
exit
```
> Confira, no prompt do shell, se vocês voltou a estar com seu superusuário (root ou outro) ativo
> *root@seuServidorOdoo:~#*



## 11. Arquivo de Configuração para a seu servidor Odoo

Crie a pasta para armazenamento do arquivo de configuração:
```sh
sudo mkdir -p /etc/odoo
```

Crie um arquivo de configuração, copiando o arquivo exemplo:
```sh
sudo cp /opt/odoo17/server/debian/odoo.conf /etc/odoo/odoo17-server.conf
```

Abra o arquivo e edite comforme é mostrado a seguir:
```sh
sudo nano /etc/odoo/odoo17-server.conf
```

```ini
[options]
; admin_password é uma senha de autenticação, definida por você, para operação relacionada ao banco de dados que permite criar, restaurar, remover. Escolha uma senha segura e anote-a.
admin_passwd = admin
db_host = False
db_port = False
db_user = odoo17
db_password = False
xmlrpc_port = 8069
addons_path = /opt/odoo17/server/addons,/opt/odoo17/server/custom-addons
logfile = /var/log/odoo17/odoo-server.log
logrotate = True
log_level  = debug
```
Saia do editor, salvando o arquivo.

> Nota: Não se esqueça de definir a senha **admin** para algo mais seguro.



## 12. Criar o arquivo de inicialização Systemd Unit

Para rodar o Odoo como um serviço e assim poder inicia/parar/reiniciar, necessitamos criar um arquivo de serviço no diretório: /etc/systemd/system/

Crie o arquivo com seu editor de textos:

```sh
sudo nano /etc/systemd/system/odoo17.service
```

E cole a seguinte configuração:

```ini
[Unit]
Description=Odoo17 Community Edition
Requires=postgresql.service
After=network.target postgresql.service

[Service]
Type=simple
SyslogIdentifier=odoo17
PermissionsStartOnly=true
User=odoo17
Group=odoo17
ExecStart=/opt/odoo17/venv-odoo17/bin/python3 /opt/odoo17/server/odoo-bin -c /etc/odoo/odoo17-server.conf
StandardOutput=journal+console

[Install]
WantedBy=multi-user.target
```
Saia do editor, salvando o arquivo.

Notifique o systemd de que um novo arquivo unit existe:
```sh
sudo systemctl daemon-reload
```

Inicie o serviço, rodando:
```sh
sudo systemctl start odoo17
```

Habilite o serviço para ser iniciado automaticamente na inicialização do seu sistema operacional:
```sh
sudo systemctl enable odoo17
```

Confira o status do serviço, com o seguinte comando:
```sh
sudo systemctl status odoo17
```
> A saída deste comando deve ser semelhante a esta:
>
> ● odoo17.service - Odoo17 Community Edition
>     Loaded: loaded (/etc/systemd/system/odoo17.service; enabled; vendor preset: enabled)
>     Active: active (running) since Wed 2024-01-03 21:51:22 -03; 2s ago
>   Main PID: 37715 (python3)
>      Tasks: 1 (limit: 2220)
>     Memory: 44.2M
>        CPU: 2.056s
>     CGroup: /system.slice/odoo17.service
>             └─37715 /opt/odoo17/venv-odoo17/bin/python3 /opt/odoo17/server/odoo-bin -c /etc/odoo/odoo17-server.conf
>
>jan 03 21:51:22 seuServidorOdoo systemd[1]: Started Odoo17 Community Edition.
> 


Se desejar visualizar as mensagens registradas do serviço, use o comando a seguir:
```sh
sudo journalctl -u odoo17
```



## 13. Testando a Instalação

OAbra o seu navegador web e digite o seu endereço e respectiva porta do Odoo: 'http://<SeuDominio_ou_EndereçoIP>:8069'

Assumindo que a instalação foi um sucesso, uma tela similar a mostrada abaixo deve surgir e você poderá configurar o seu banco de dados


![Getting Started](imagens/PrimeiroAcesso-Odoo.jpg)





