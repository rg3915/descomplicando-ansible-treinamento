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
EOF
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

```
# Criando arquivos
cat << EOF > hosts
[k8s-master]
xx.xxx.xxx.xxx

[k8s-workers]
xx.xxx.xxx.xxx
xx.xxx.xxx.xxx

[k8s-workers:vars]
K8S_MASTER_NODE_IP=xxx.xx.xx.xxx
K8S_API_SECURE_PORT=6443
EOF
```

Para editar os IPs, vá em provisioning/hosts e digite os novos IPs em install_k8s/hosts...

```
cat ../provisioning/hosts
```

... para editar o K8S_MASTER_NODE_IP, vá na AWS EC2 e pegue o IP privado da máquina que você definiu como master.



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




# Aula 3

```
ssh-agent
ssh-add
```

https://www.freecodecamp.org/news/openssl-command-cheatsheet-b441be1e8c4a/

https://www.weave.works/blog/weave-net-kubernetes-integration/

https://docs.ansible.com/ansible/latest/user_guide/playbooks_checkmode.html


```
ansible-playbook -i hosts main.yml -u ubuntu

ssh-add chave.pem
```

* provisiong
* install
* create

```
kubeadm token create --print-join-command 
```


## Começando a aula com dois playbooks:

* provisioning
* install_k8s

E quatro pastas em `install_k8s`:

* create-cluster
* install-helm
* install-k8s
* join-workers

Após instalado o k8s, precisamos fazer com que ele seja um cluster.


Criando uma nova pasta com o conteúdo da aula2.

```
cp -r aula2/ descomplicando-ansible
```

Vá para a pasta `provisioning`. Certifique-se de ter colocado a chave.pem.

```
# Rodando o Ansible
# Provisionando as máquinas
# Criando as máquinas no EC2.
ansible-playbook -i hosts main.yml -u ubuntu
```


## install-k8s

Faça o mesmo na pasta `install_k8s`.

```
# Rodando o Ansible
# Provisionando as máquinas
# Criando as máquinas no EC2.
ansible-playbook -i hosts main.yml -u ubuntu
```

Vá para a pasta...

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/install_k8s/roles/create-cluster/tasks/
```

```
printf "\n- include: init-cluster.yml" >> main.yml
```

Edite `init-cluster.yml`

```
cat << EOF > init-cluster.yml
- name: Reset Cluster
  command:
    kubeadm reset --force
  register: kubeadmin_init

- name: Initialize Kubernetes master with kubeadm init.
  command:
    kubeadm init
  register: kubeadmin_init

- name: Ensure .kube directory exists.
  file:
    path: ~/.kube
    state: directory

- name: Symlink the kubectl admin.conf to ~/.kube/conf.
  file:
    src: /etc/kubernetes/admin.conf
    dest: ~/.kube/config
    state: link

- name: Configure weavenet networking.
  shell: kubectl apply -f {{ default_kubernetes_cni_weavenet_manifestUrl }}
  register: weavenet_result

- name: "Cluster token"
  shell: kubeadm token list | cut -d ' ' -f1 | sed -n '2p'
  register: K8S_TOKEN

- name: "CA Hash"
  shell: openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
  register: K8S_MASTER_CA_HASH

- name: "Add K8S Token and Hash to dummy host"
  add_host:
    name:   "K8S_TOKEN_HOLDER"
    token:  "{{ K8S_TOKEN.stdout }}"
    hash:   "{{ K8S_MASTER_CA_HASH.stdout }}"

- name:
  debug:
    msg: "[Master] K8S_TOKEN_HOLDER K8S token is {{ hostvars['K8S_TOKEN_HOLDER']['token'] }}"

- name:
  debug:
    msg: "[Master] K8S_TOKEN_HOLDER K8S Hash is  {{ hostvars['K8S_TOKEN_HOLDER']['hash'] }}"
EOF
```

Editando vars

Pesquise por Run Weave Net with Kubernetes in Just One Line

https://www.weave.works/blog/weave-net-kubernetes-integration/


```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/install_k8s/roles/create-cluster
printf '\ndefault_kubernetes_cni_weavenet_manifestUrl: "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\\n')"' >> vars/main.yml
```

Rode o playbook

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/install_k8s
ansible-playbook -i hosts main.yml -u ubuntu
```

Conecte-se nas novas máquinas. Abra 3 terminais, um pra cada máquina.



```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/provisioning
ssh-add descomplicando-ansible.pem # nos 3 terminais
ssh ubuntu@54.162.48.173
ssh ubuntu@52.23.179.250
ssh ubuntu@54.81.192.27
```

Em um deles rode:

```
sudo su -
docker ps
kubectl get nodes
```

Fazer join-workers

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/install_k8s/roles/join-workers

printf "\n- include: join-cluster.yml" >> tasks/main.yml

cat << EOF > tasks/join-cluster.yml
- name: 
  debug:
    msg: "[Worker] K8S_TOKEN_HOLDER K8S token is {{ hostvars['K8S_TOKEN_HOLDER']['token'] }}"

- name:
  debug:
    msg: "[Worker] K8S_TOKEN_HOLDER K8S Hash is  {{ hostvars['K8S_TOKEN_HOLDER']['hash'] }}"

- name: "Kubeadm reset node cluster config"
  command: 
    kubeadm reset --force
  register: kubeadm-reset_node

- name: "Kubeadm join"
  shell: 
    kubeadm join --token={{ hostvars['K8S_TOKEN_HOLDER']['token'] }}
    --discovery-token-ca-cert-hash sha256:{{ hostvars['K8S_TOKEN_HOLDER']['hash'] }}
    {{K8S_MASTER_NODE_IP}}:{{K8S_API_SECURE_PORT}}
EOF
```

> Não pode ter swap ligada.

Pra pegar o token do k8s

```
kubeadm token create --print-join-command
```

Editar hosts

```
cd install_k8s/hosts
vim hosts

[k8s-workers:vars]
K8S_MASTER_NODE_IP=xxx.xx.xx.xxx
K8S_API_SECURE_PORT=6443
```

Em `K8S_MASTER_NODE_IP` colocar IP interno Private IP.

Na AWS, editar Security Groups

Inbound

Type: All trafic
Protocol: All
Source: custom

Para o grupo 'giropops'.

E rodar o playbook novamente.

```
ansible-playbook -i hosts main.yml -u ubuntu
```

Editar

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/provisioning/roles/create/tasks
vim provisioning.yml
```

## Install-helm

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/install_k8s/roles/install-helm/
printf "\n- include: install-helm.yml\n- include: install-monit-tools.yml" >> tasks/main.yml
```

Editar install-helm.yml

```
cat << EOF > tasks/install-helm.yml
- name: Install helm via curl
  shell: curl -L https://git.io/get_helm.sh | bash -
  register: helm_result

- name: Helm init
  shell: helm init
  register: helm_init_result

- name: Create service account to tiller
  shell: kubectl create serviceaccount --namespace=kube-system tiller
  register: tiller_result

- name: Create clusterrolebinding for tiller
  shell: kubectl create clusterrolebinding tiller-cluster-role --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
  register: clusterrolebinding_result

- name: Apply patch to tiller-deploy
  shell: kubectl patch deployments -n kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
  register: patch_result

- name: Waiting tiller pod
  pause:
    minutes: 2
EOF
```

Editar install-monit-tools.yml

```
cat << EOF > tasks/install-monit-tools.yml
- name: Install Prometheus
  shell: helm install {{ deploy_prometheus }}
  register: prometheus_result

- name: Install Grafana
  shell: helm install {{ deploy_grafana }}
  register: grafana_result
EOF
```

Edite `vars/main.yml`

```
printf '\ndeploy_prometheus: "--namespace=monitoring --name=prometheus --version=7.0.0 --set alertmanager.persistentVolume.enabled=false,server.persistentVolume.enabled=false stable/prometheus"' >> vars/main.yml

printf '\ndeploy_grafana: "--namespace=monitoring --name=grafana --version=1.12.0 --set=adminUser=admin,adminPassword=admin,service.type=NodePort stable/grafana"' >> vars/main.yml
```

Entre no nó master, e

```
kubectl get deployment --all-namespaces
kubectl get services -n monitoring
kubectl get nodes
```

Vá em Security groups e libere a porta do Grafana

Custom
TCP
port: 30730
description: Grafana

Entre em uma das máquinas pelo browser, na porta 30730

Grafana: admin, admin

No Grafana, clique em *Add Data Source*

Name: Prometheus
URL: http://prometheus-server

```
kubectl port-forward -n monitoring svc/prometheus-server --address 0.0.0.0 32000:80
```


---

# Aula 4


```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/

mkdir deploy-app-v1
cd deploy-app-v1

cp ../install_k8s/hosts .

mkdir roles
cd roles
```

```
ansible-galaxy init common

cd common
```

```
printf "\n- include: deploy-app.yml" >> tasks/main.yml
```

```
cat << EOF > tasks/deploy-app.yml
- name: Creating Giropops App directory
  file: path={{ item }} state=directory
  with_items:
    - /opt/giropops
    - /opt/giropops/logs
    - /opt/giropops/conf
  register: directory_app_register

- name: Copying deployment file to host
  template:
    src: app-v1.yml.j2
    dest: /opt/giropops/app-v1.yml
    owner: root
    group: root
    mode: 0644
  register: copying_template_register

- name: Copying service file to host
  copy: src={{ item.src }} dest={{ item.dest }}
  with_items:
    - { src: 'service-app.yml', dest: '/opt/giropops/service-app.yml' }
  register: copying_register

- name: Deploy Giropops App deployment
  shell: kubectl apply -f /opt/giropops/app-v1.yml
  register: deploy_deployment_register

- name: Deploy Giropops App service
  shell: kubectl apply -f /opt/giropops/service-app.yml
  register: deploy_service_register
EOF
```

Editar app-v1.yml

```
cat << EOF > files/app-v1.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: giropops-v1
spec:
  replicas: {{ number_replicas }}
  template:
    metadata:
      labels:
        app: giropops
        version: {{ version }}
      annotations:
        prometheus.io/scrape: "{{ prometheus_scrape }}"
        prometheus.io/port: "{{ prometheus_port }}"
    spec:
      containers:
      - name: giropops
        image: linuxtips/nginx-prometheus-exporter:{{ version }}
        env:
        - name: VERSION
          value: {{ version }}
        ports:
        - containerPort: {{ nginx_port }}
        - containerPort: {{ prometheus_port }}
EOF
```

```
cat << EOF > files/service-app.yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: giropops
    run: nginx
    track: stable
  name: giropops
  namespace: default
spec:
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 32222
    name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - nodePort: 32111
    name: prometheus
    port: 32111
    protocol: TCP
    targetPort: 32111
  selector:
    app: giropops
  sessionAffinity: None
  type: NodePort
EOF
```

```
cat << EOF > vars/main.yml
---
# vars file for common

# Giropops app
number_replicas: 10
version: 1.0.0
prometheus_scrape: "true"
prometheus_port: 32111
nginx_port: 80
environment: production
EOF
```

Copiar o arquivo

cp files/app-v1.yml templates/app-v1.yml.j2


```
cd ../..
cat << EOF > main.yml
- hosts: k8s-master
  become: yes
  user: ubuntu
  roles:
  - common
EOF
```


### Criar as máquinas e liberar as portas.

Fazer:

1. provisioning
2. install_k8s
3. deploy-app-v1

```
cd provisioning
ansible-playbook -i hosts main.yml -u ubuntu
```

Não esqueça de limpar os ips de hosts

Não esqueça de fazer ssh-add chave.pem

```
cd ../install_k8s
ansible-playbook -i hosts main.yml -u ubuntu
```

Copiar o hosts para a pasta `deploy-app-v1`.

```
cd ../deploy-app-v1/
cp ../install_k8s/hosts .
```

Rodar o playbook.

```
ansible-playbook -i hosts main.yml -u ubuntu
```

O erro sobre `kubectl apply -f` sobre selector, faça o seguinte:

Entre no servidor e faça:

```
kubectl apply -f app-v1.yml
```

Depois edite `app-v1.yml`:

```
...
replicas: {{ number_replicas }}
  selector:
    matchLabels:
      app: giropops
```

Se tudo der certo, você pode fazer:

```
# no servidor
kubectl get svc
```

Depois vá na AWS, EC2, clique na instância e clique lá embaixo em Security groups.

Inbound: All trafic, anywhere

Você pode navegar a partir da url do servidor na porta 32222.

http://PUBLIC_IP:32222

```
source <(kubectl completion bash)
kubectl logs giropops-v1-689bf72kcus --follow # pra ver os logs
```

E

http://PUBLIC_IP:32111/metrics

Pra ver todos os outros serviços digite:

```
kubectl get svc --all-namespaces
```

O Grafana está na porta PORT.

Add data source

Name: Prometheus Server

Type: Prometheus

URL: http://prometheus-server


New Dashboard > Graph > Edit

Data Source: Prometheus Server

```
sum(rate(nginx_http_requests{app='giropops'}[5m])) by (version)
```

Legend format: {{version}}

Display


## App v2

Crie deploy-app-v2

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible
mkdir -p deploy-app-v2/roles
cd deploy-app-v2/roles
ansible-galaxy init common
```

Volte dois níveis...

```
cd ../..
```

Crie canary-deploy-app

```
mkdir -p canary-deploy-app/roles
cd canary-deploy-app/roles
ansible-galaxy init common
```

Em canary-deploy-app, faça:

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/canary-deploy-app/roles/common
```

Crie uma task:

```
printf "\n- include: deploy-app.yml" >> tasks/main.yml
```

```
cat << EOF > tasks/deploy-app.yml
- name: Copying deployment file to host
  template:
    src: app-v2-canary.yml.j2
    dest: /opt/giropops/app-v2-canary.yml
    owner: root
    group: root
    mode: 0644
  register: copying_template_register

- name: Deploy Giropops App deployment
  shell: kubectl apply -f /opt/giropops/app-v2-canary.yml
  register: deploy_deployment_register
EOF
```

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/canary-deploy-app/roles/common/

cat << EOF > templates/app-v2-canary.yml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giropops-v2
spec:
  replicas: {{ number_replicas }}
  selector:
    matchLabels:
      app: giropops
  template:
    metadata:
      labels:
        app: giropops
        version: {{ version }}
      annotations:
        prometheus.io/scrape: "{{ prometheus_scrape }}"
        prometheus.io/port: "{{ prometheus_port }}"
    spec:
      containers:
      - name: giropops
        image: linuxtips/nginx-prometheus-exporter:{{ version }}
        env:
        - name: VERSION
          value: {{ version }}
        ports:
        - containerPort: {{ nginx_port }}
        - containerPort: {{ prometheus_port }}
EOF
```

```
cat << EOF > vars/main.yml
---
# vars file for common

# Giropops app
number_replicas: 1
version: 2.0.0
prometheus_scrape: "true"
prometheus_port: 32111
nginx_port: 80
environment: production
EOF
```

Depois copie

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/
cp deploy-app-v1/main.yml canary-deploy-app/
cp deploy-app-v1/hosts canary-deploy-app/
cd canary-deploy-app
```

Rode o playbook

```
ansible-playbook -i hosts main.yml
```

Para conferir no servidor, digite:

```
kubectl get deploy
```

Agora vamos em:

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/deploy-app-v2/roles/common

printf "\n- include: deploy-app.yml" >> tasks/main.yml

cat << EOF > tasks/deploy-app.yml
- name: Copying deployment file app v1 to host
  template:
    src: app-v1.yml.j2
    dest: /opt/giropops/app-v1.yml
    owner: root
    group: root
    mode: 0644
  register: copying_app1_template_register

- name: Copying deployment file app v2 to host
  template:
    src: app-v2.yml.j2
    dest: /opt/giropops/app-v2.yml
    owner: root
    group: root
    mode: 0644
  register: copying_app2_template_register

- name: Deploy new version of Giropops App deployment
  shell: kubectl apply -f /opt/giropops/app-v2.yml
  register: deployment_v2_register

- name: Scale down old version of Giropops App deployment
  shell: kubectl apply -f /opt/giropops/app-v1.yml
  register: deployment_v1_register

- name: The old version of Giropops App deployment will be removed in two minutes
  pause:
    minutes: 2

- name: Delete old version of Giropops App deployment
  shell: kubectl delete -f /opt/giropops/app-v1.yml
  register: deployment_deleted_register
EOF
```

```
cat << EOF > templates/app-v1.yml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giropops-v1
spec:
  replicas: {{ number_replicas_old_version }}
  selector:
    matchLabels:
      app: giropops
  template:
    metadata:
      labels:
        app: giropops
        version: {{ old_version }}
      annotations:
        prometheus.io/scrape: "{{ prometheus_scrape }}"
        prometheus.io/port: "{{ prometheus_port }}"
    spec:
      containers:
      - name: giropops
        image: linuxtips/nginx-prometheus-exporter:{{ old_version }}
        env:
        - name: VERSION
          value: {{ old_version }}
        ports:
        - containerPort: {{ nginx_port }}
        - containerPort: {{ prometheus_port }}
EOF
```

```
cat << EOF > templates/app-v2.yml.j2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giropops-v2
spec:
  replicas: {{ number_replicas_new_version }}
  selector:
    matchLabels:
      app: giropops
  template:
    metadata:
      labels:
        app: giropops
        version: {{ new_version }}
      annotations:
        prometheus.io/scrape: "{{ prometheus_scrape }}"
        prometheus.io/port: "{{ prometheus_port }}"
    spec:
      containers:
      - name: giropops
        image: linuxtips/nginx-prometheus-exporter:{{ new_version }}
        env:
        - name: VERSION
          value: {{ new_version }}
        ports:
        - containerPort: {{ nginx_port }}
        - containerPort: {{ prometheus_port }}
EOF
```

```
cat << EOF > vars/main.yml
---
# vars file for common

# Giropops app
number_replicas_old_version: 1
number_replicas_new_version: 10
old_version: 1.0.0
new_version: 2.0.0
prometheus_scrape: "true"
prometheus_port: 32111
nginx_port: 80
environment: production
EOF
```

Copiando main e hosts

```
cd ~/gh/my/descomplicando-ansible-treinamento/descomplicando-ansible/deploy-app-v2
cp ../deploy-app-v1/main.yml .
cp ../deploy-app-v1/hosts .
```

Rodando o playbook

```
ansible-playbook -i hosts main.yml
```



# Aula 5

* Criar uma máquina Ubuntu

```
sudo su -
```

# instalar ansible

```
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt update
sudo apt install -y ansible
```

# instalar docker

```
curl -fsSL https://get.docker.com | bash -

apt-get install -y python-pip

pip install docker-compose==1.9.0

apt-get install -y nodejs npm

npm install npm --global
```

Clonar o repo do Ansible awx

```
git clone https://github.com/ansible/awx.git

cd awx/installer

cat install.yml
cat inventory
```



Gerar um secretkey

```
openssl rand -hex 32
```

```
...
project_data_dir

grep -v '^#' inventory | grep - '^$' > inventory_limpo
```

---

```
localhost ansible_connection=local ansible_python_interpreter="/usr/bin/env python"
[all:vars]
dockerhub_base=ansible
awx_task_hostname=awx
awx_web_hostname=awxweb
postgres_data_dir=/var/lib/pgdocker
host_port=8080
host_port_ssl=443
docker_compose_dir=/var/lib/awx
pg_username=awx
pg_password=giropops
pg_admin_password=giropops
pg_database=awx
pg_port=5432
rabbitmq_password=giropops
rabbitmq_erlang_cookie=cookiemonster
admin_user=admin
admin_password=giropops
create_preload_data=True
secret_key=91f1db6f6d7691121bcfa62708d59c2cd3f4aa9bd6f6d25f261395684c659dd3
project_data_dir=/var/lib/awx/projects
```

---

```
ansible-playbook -i inventory install.yml

docker ps
```

Security group

All traffic

Entra no ip

Criar um projeto

```
cd awx/project
mkdir opa
cd opa
ansible-galaxy init opa
```

