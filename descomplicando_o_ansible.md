# Ansible

```
ssh-add chave.pem
ssh ip

ssh copy id (site)

ssh-keygen

ifconfig pra pegar o ip da maquina

ssh-copy-id
```

No GCloud é o IP interno

---

## Conectando na máquina

```
ssh -o ServerAliveInterval=30 -i chave.pem host
```

IP externo vs IP interno


Nas máquinas trocar o host

```
sudo su -
hostname elliot-01
hostname elliot-02
hostname elliot-03
```

```
vim /etc/hostname

elliot-01

mkdir ansible
cd ansible
```

Chave ssh

```
scp -i chave.pem chave.pem ubuntu@ec2-...com:/tmp
```

Na outra maquina

```
mv /tmp/chave.pem ~

ssh-agent bash
ssh-add chave.pem
```


```
ifconfig eth0

xxx.xx.xx.xxx elliot-01

mkdir ansible
cd ansible

sudo vim /etc/hosts
```

```
# hosts
...
xxx.xx.xx.xx2 elliot-01
xxx.xx.xx.xx3 elliot-02
xxx.xx.xx.xx4 elliot-03
```

Faça o teste com

```
ping elliot-01
```

Agora dentro do ansible

```
mkdir ansible
cd ansible
vim ~/ansible/hosts
```

```
# hosts
elliot-01
elliot-02
elliot-03
```

## Instalação

```
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt update
sudo apt install -y ansible
```


### Inventário

Antes você pode fazer `ssh elliot-01`

-m é módulo

```
ansible -i hosts all -m ping  # opcional -k pra pedir senha
```

```
# hosts
[giropops]
elliot-01

[webservers]
elliot-02
elliot-03
```

```
ansible -i hosts webservers -m ping -u usuario -k
```

Para executar comandos digite `-a`

```
ansible -i hosts webservers -a "/sbin/ifconfig"
```

Executando comandos no bash

```
ansible -i hosts webservers -a "bash -c 'uptime'"
ansible -i hosts webservers -m copy -a "scr=hosts dest=/tmp"
ansible -i hosts webservers -m shell -a "uptime"
ansible -i hosts webservers -m git -a "repo=https://github.com/badtuxx/giropops-monitoring.git dest=/tmp/giropops version=HEAD"
ansible -i hosts elliot-02 -m setup
ansible -i hosts elliot-02 -m setup -a "filter=ansible_distribution"
```

```
ansible -i hosts all -m apt -a "name=vim state=present"
```

Vai dar permission denied

```
ansible -i hosts all -b -m apt -a "name=vim state=present"

cd /etc/sudoers.d
vim sudoers
```


```
ubuntu ALL=(ALL) NOPASSWD:ALL

[giropops:vars]
ansible_python_interpreter=/usr/bin/python3

mkdir day1
vim nginx_playbook.yml

# nginx_playbook.yml
---
- hosts: webservers
  become: yes
  remote_user: root
  tasks:
  - name: Instalando o nginx
    apt:
      name: nginx
      state: latest
      update_cache: yes
  - name: Iniciando o nginx
    service:
      name: nginx
      state: started
  - name: Copiando index.html personalizado
    copy:
      src: index.html
      dest: /var/www/html/index.html
  - name: Restartando o nginx
    service:
      name: nginx
      state: restarted
```

```
ansible-playbook -i hosts nginx_playbook.yml -b

sudo netstat -atunp

cd /usr/share/nginx/html

```


Crie o index.html

```
ansible-playbook -i hosts nginx_playbook.yml -b

sudo systemctl restart nginx
```


```
# nginx_playbook.yml
---
- hosts: webservers
  become: yes
  remote_user: root
  tasks:
  - name: Instalando o nginx
    apt:
      name: nginx
      state: latest
      update_cache: yes
  - name: Iniciando o nginx
    service:
      name: nginx
      state: started
  - name: Copiando index.html personalizado
    copy:
      src: index.html
      dest: /var/www/html/index.html
  - name: Copiando nginx.conf
    copy:
      src: nginx.conf
      dest: /etc/nginx/nginx.conf
    notify: Restartando o nginx
  handlers:
  - name: Restartando o nginx
    service:
      name: nginx
      state: restarted
```


```
# nginx.conf
...
```


---

# Aula 2

Na sua máquina

```
pip install ansible
```

e crie o playbook localmente

```
cd ~/gh/my/descomplicando-ansible-treinamento
mkdir -p aula2/roles
cd aula2
```


Editando main.yml

```
cat << EOF > main.yml
---
- hosts: local
  roles:
  - create
EOF
```

Editando hosts

```
cat << EOF > hosts
[local]
localhost ansible_connection=local ansible_python_interpreter=python gather_facts=false

[kubernetes]
```

Copie a chave.pem.

Criando a pasta `roles`.

```
# Criando pasta
mkdir roles
cd roles
```

```
# Criando projeto Ansible
ansible-galaxy init create

cd create
```


Google: amazon ec2 ami locator

https://cloud-images.ubuntu.com/locator/ec2/

Criar chave `descomplicando-ansible` na AWS. 

```
cat << EOF > vars/main.yml
---
# vars file for create
instance_type: t2.medium
security_group: giropops
image: ami-064a0193585662d74
keypair: descomplicando-ansible
region: us-east-1
count: 3
EOF
```

```
printf "\n- include: provisioning.yml" >> tasks/main.yml
```


Google: ansible all modules


```
cat << EOF > tasks/provisioning.yml
- name: Criando o Security Group
  local_action:
    module: ec2_group
    name: "{{ security_group }}"
    description: Security Group Giropops
    region: "{{ region }}"
    rules:
    - proto: tcp
      from_port: 22
      to_port: 22
      cidr_ip: 0.0.0.0/0
    rules_egress:
    - proto: all
      cidr_ip: 0.0.0.0/0
  register: basic_firewall

- name: Criando a instancia EC2
  local_action: ec2
    group={{ security_group }}
    instance_type={{ instance_type }}
    image={{ image }}
    wait=true
    region={{ region }}
    keypair={{ keypair }}
    count={{ count }}
  register: ec2

- name: Adicionando a instancia ao inventario temp
  add_host: name={{ item.public_ip }} groups=giropops-new
  with_items: "{{ ec2.instances }}"

- name: Adicionando a instancia criada no arquivo hosts
  local_action: lineinfile
    dest="./hosts"
    regexp={{ item.public_ip }}
    insertafter="[kubernetes]" line={{ item.public_ip }}
  with_items: "{{ ec2.instances }}"

- name: Esperando o SSH
  local_action: wait_for
    host={{ item.public_ip }}
    port=22
    state=started
  with_items: "{{ ec2.instances }}"

- name: Adicionando um nome tag na instancia
  local_action: ec2_tag resource={{ item.id }} region={{ region }} state=present
  with_items: "{{ ec2.instances }}"
  args:
    tags:
      Name: regis-ansible-day2
EOF
```


Em EC2 --> Key Pairs --> Create KeyPair
IAM --> Create User

Pegar as chaves de acesso.

### Exportando AWS_ACCESS_KEY

```
export AWS_ACCESS_KEY_ID="FAKE"
export AWS_SECRET_ACCESS_KEY="FAKE"
```


> Copie a chave para dentro da pasta `aula2`.

```
cd ~/gh/my/descomplicando-ansible-treinamento/aula2/
cp ~/Downloads/descomplicando-ansible.pem .
```

Dando permissão de acesso a chave.

```
chmod 0400 descomplicando-ansible.pem
```

Rodando `ssh-add`

```
ssh-add descomplicando-ansible.pem
```

Estando na pasta `aula2`...

```
pip install boto3
```

Executar na mesma pasta do main.yml.

```
# Rodando o Ansible
# Provisionando as máquinas
# Criando as máquinas no EC2.

ansible-playbook -i hosts main.yml
```

### Profile

Se quiser no playbook pode adicionar um profile em todos

profile: "{{ profile }}"

Em vars/main.yml coloque

profile: default # ou giropops

Requer o arquivo

```
~/.aws/credentials
```

```
# Rodando o Ansible
ansible-playbook -i hosts main.yml
```

Movendo tudo para a pasta `provisioning`.

```
mkdir provisioning
mv descomplicando-ansible.pem hosts hosts_example main.yml provisioning
mv roles provisioning
```

Criando outro playbook.

```
mkdir install_k8s
cd install_k8s
cp ../provisioning/hosts .
cp ../provisioning/main.yml .
mkdir roles
```

Editando o main.yml

```
cat << EOF > main.yml
---
- hosts: all
  become: yes
  user: ubuntu
  gather_facts: no
  pre_tasks:
  - name: 'install python'
    raw: 'apt-get -y install python'
  roles:
  - install-k8s

- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - create-cluster

- hosts: k8s-workers
  become: yes
  user: ubuntu
  roles:
  - join-workers

- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - install-helm
EOF
```

Criar os roles

```
cd roles

ansible-galaxy init install-k8s
ansible-galaxy init create-cluster
ansible-galaxy init join-workers
ansible-galaxy init install-helm
```

```
printf "\n- include: install.yml" >> install-k8s/tasks/main.yml

```

```
cat << EOF > install-k8s/tasks/install.yml
- name: Instalando o Docker
  shell: curl -fsSL https://get.docker.com | bash -

- name: Adicionando as chaves repo k8s no apt
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Adicionando o repo do k8s
  apt_repository:
    repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
    state: present

- name: Instalando pacotes k8s
  apt:
    name: "{{ packages }}"
  vars:
    packages:
    - kubelet
    - kubeadm
    - kubectl
EOF
```

cd ..

Limpar os IPs de hosts

E criar as máquinas novamente.

```
cd provisioning
ansible-playbook -i hosts main.yml -u ubuntu
```

Copiar os hosts para

```
cp hosts ../install_k8s/

cd ../install_k8s

printf "\n[k8s-master]\n[k8s-workers]" >> hosts
```

Edite seus grupos e IPs de tal forma que você tenha isso (remova o grupo local):

```
[k8s-master]
x.xx.xx.xx

[k8s-workers]
xx.xxx.xx.xx
xx.xxx.xx.xx

[k8s-workers:vars]
K8S_MASTER_NODE_IP=xxx.xx.xx.xxx
K8S_API_SECURE_PORT=6443
```

Na pasta install_k8s...

```
ansible-playbook -i hosts main.yml -u ubuntu
```

Vai dar erro de confirmação da conexão ssh

```
ssh ubuntu@xx.xxx.xx.xx
ssh ubuntu@xx.xxx.xx.xx
ssh ubuntu@xx.xxx.xx.xx
```



```
ansible-playbook -i hosts main.yml
```

Dentro de uma das máquinas, faça:

```
ssh ubuntu@3.81.248.9

sudo su -
docker --version
kubelet --version
```

Saia do host e volte pra sua máquina

```
ansible -i hosts all -a "docker --version" -u ubuntu
ansible -i hosts all -a "kubectl version" -u ubuntu
```


Para não precisar confirmar a conexão ssh toda vez, faça:

```
cd ../provisioning/roles/create/
vim tasks/provisioning.yml
```

e adicione o role a seguir:

```
- name: Adicionando a maquina criada no known_hosts
  shell: ssh-keyscan -H {{ item.public_ip }} >> ~/.ssh/known_hosts
  with_items: "{{ ec2.instances }}"
```

Delete as máquinas, e crie outras novamente.

```
ansible-playbook -i hosts main.yml -u ubuntu
```

Conecte na máquina novamente.

```
ssh ubuntu@x.xx.xxx.x
```

Vá na pasta install_k8s e rode o playbook novamente.

```
cd ../install_k8s
ssh-agent bash
ssh-add descomplicando-ansible.pem
ansible-playbook -i hosts main.yml -u ubuntu
```




