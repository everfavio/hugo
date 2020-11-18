---
title: 'Instalación de Kubernetes On Premise'
date: 2020-02-23T13:41:16-04:00
hero: "/images/kubernetes.jpg"
excerpt: Pasos para instalar kubernetes y sus dependencias en BareMetal.
authors:
  - Ever Favio Argollo Ticona
categories: [Kubernetes]
---

Se describen los siguientes pasos para instalar una instancia kubernetes y sus nodos preparada para ambientes de producción.

### Preinstalación: Paquetes necesarios y Registro de llaves

Actualización de repositorio
```shell
    $ sudo apt-get update
```

Instalación de requisitos previos que permitan usar paquetes a través de HTTPS:

```shell
  $ sudo apt install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```
Agregando la llave GPG desde el repositorio oficial de Docker al sistema:

```shell
  $ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
Agregando el repositorio docker a los recursos locales APT
```shell
  $ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
```
Actualizando recursos locales
```shell
  $ sudo apt-get update
```
Instalamos iptables arptables y ebtables
```shell
  $ sudo apt-get install -y iptables arptables ebtables
```
Verificamos que el MAC address y el product_uuid son unicos para cada nodo (servidor) que usaremos, 

Podemos verificar que el MAC sea unico para cada nodo ejecutando en la terminal de cada servidor
```shell
  $ ip link
```
o usamos el comando:
```shell
  $ ifconfig -a
```
El product_uuid puede ser verificado utilizando el comando:
```shell
  $ sudo cat /sys/class/dmi/id/product_uuid
```
Agregamos la llave GPG de kubernetes:

```shell
  $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
Agregamos el repositorio kubernetes:
```shell
  $ cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
Actualizando recursos locales:
```shell
  $ sudo apt-get update
```
### Instalación: instalando kubernetes
Instalamos Docker, Kubelet, Kubeadm y Kubectl:
```shell
  $ sudo apt-get install -y docker-ce kubelet kubeadm kubectl
```
Aseguramos que siempre se mantengan en la versión instalada:
```shell
  $ sudo apt-mark hold docker-ce kubelet kubeadm kubectl
```
Agregamos al usuario de ejecución al grupo docker
```shell
  $ sudo usermod -aG docker $USER
```
Agregamos una regla ipTables a sysctl.conf:
```shell
  $ echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
```
Habilitamos las reglas inmediatamente:
```shell
  $ sudo sysctl -p
```
Deshabilitamos el modo swap del k8s master y los nodos
```shell
  $ swapoff -a
```
Iniciamos el cluster(paso ejecutado en el k8s master)
```shell
  $ sudo kubeadm init --pod-network-cidr=<ip-k8s-master>/16
```
Guardamos el token que nos genera,
Luego ejecutamos los siguientes comandos (Solo en el nodo master)

```shell
  $ sudo cp /etc/kubernetes/admin.conf $HOME/
  $ sudo chown $(id -u):$(id -g) $HOME/admin.conf
  $ export KUBECONFIG=$HOME/admin.conf
```
Agregamos el módulo flannel CNI
```shell
  $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
Agregamos los nodos a nuestro k8s ejecutando el comando
```shell
  $ sudo kubeadm join <ip-master>:6443 --token <token generado en pasos anteriores> --discovery-token-unsafe-skip-ca-verification 
```
### Creando un cluster de alta disponibilidad en k8s

Visualizamos los pods en el namespace por default con una vista customizada
```shell
  $ kubectl get pods -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```
Visualizamos el kube-schenduler YAML
```shell
  $ kubectl get endpoints kube-scheduler -n kube-system -o yaml
```
Creamos una topología etcd apilada usando kubeadm
```shell
  $ kubeadm init --config=kubeadm-config.yaml
```
