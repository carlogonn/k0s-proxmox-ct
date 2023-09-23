**------------- bajar template de rocky linux en proxmox --------------**

pveam list local

pveam available | grep 'rocky'

pveam download local rockylinux-8-default_20210929_amd64.tar.xz

**------------- Usar el template para crear una plantilla --------------**

Pasos en la creacion de contenedor

Pestaña 

General: Asignar Hostname / Password / Confirm Password / SSH public key / Destiladar Unprivileged / Tildar Advanced

Template: Seleccionar rocky Linux 8

Root Disk : Storage: 20 Gb o superior

CPU: 1024 Gb o superior

Network: IPv4: 192.168.211.<IP-CT-PLANTILLA>/24  / Gateway:  192.168.211.1 
Nota: las direcciones IPs usadas son de la red local personal

DNS: 8.8.8.8

En ultima pestañan hacer que no se inicie
-------------------------------------------------------------------------

En proxmox luego de creado el contenedor debemos ingresar a consola del proxmox y editar

/etc/pve/lxc/<container_id>.conf

agregando a lo que posee el archivo el contenido
´´´
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop:
lxc.mount.auto: "proc:rw sys:rw"
´´´
y modificando el valor de swap a valor 0 (cero)

swap: 0

Luego de modificado esto desde la consolo podemos enceder el contenedor 

pct start <container id>

***********************************************************************************************************************************************
A continuación, debemos publicar la configuración de arranque del kernel en el contenedor. Normalmente, el contenedor no necesita esto, 
ya que se ejecuta con el kernel del host, pero Kubelet usa la configuración para determinar varias configuraciones para el tiempo de ejecución, 
por lo que debemos copiarlo en el contenedor. Para hacer esto, primero inicie el contenedor , 
luego ejecute el siguiente comando en el host de Proxmox:
***********************************************************************************************************************************************

pct push <container id> /boot/config-$(uname -r) /boot/config-$(uname -r)

Ingresamos al contenedor, si estamos en consola de proxmox podemos hacerlo con

pct enter <container id>

Estando dentro creamos al carpeta /dev/kmsg 
(Kubelet usa esto para algunas funciones de registro y no existe en los contenedores de forma predeterminada. )

creamos el archivo /usr/local/bin/conf-kmsg.sh con el siguiente contenido
´´´
.......................................
#!/bin/sh -e
if [ ! -e /dev/kmsg ]; then
	ln -s /dev/console /dev/kmsg
fi
mount --make-rshared /
.......................................
´´´
luego creamos el archivo /etc/systemd/system/conf-kmsg.service con el contenido
´´´
.......................................
[Unit]
Description=Make sure /dev/kmsg exists
[Service]
Type=simple
RemainAfterExit=yes
ExecStart=/usr/local/bin/conf-kmsg.sh
TimeoutStartSec=0
[Install]
WantedBy=default.target
.......................................
´´´
Finalmente ejecutamos los comandos

chmod +x /usr/local/bin/conf-kmsg.sh
systemctl daemon-reload
systemctl enable --now conf-kmsg

Terminado esto apagamos el contenedor para convertirlo en plantilla

Convertido en plantilla vamos a usar la misma para crear 2 contenedores para kubernetes

Contenedor 1 - k0s-master1
Contenedor 2 - K0s-worker1

-------------------------------------------------------------------------

En mi equipo tengo que instalar k0sctl, en el link https://docs.k0sproject.io/v1.21.2+k0s.0/k0sctl-install/ 

se observan los pasos para su instalacion

brew install k0sproject/tap/k0sctl

Finalmente vamos desplegar k0s utilizando el archivo de configuracion k0sctl.yaml siguiente
´´´
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
spec:
  hosts:
  - ssh:
      address: 192.168.211.IP-MASTER
      user: root
      port: 22
      keyPath: ~/.ssh/id_rsa
    role: controller
  - ssh:
      address: 192.168.211.IP-WORKER
      user: root
      port: 22
      keyPath: ~/.ssh/id_rsa
    role: worker
  k0s:
    version: 1.27.2+k0s.0
    dynamicConfig: false
    config:
      apiVersion: k0s.k0sproject.io/v1beta1
      kind: Cluster
      metadata:
        name: k0s
      spec:
        images:
          calico:
            cni:
              image: calico/cni
              version: v3.18.1
        api:
          k0sApiPort: 9443
          port: 6443
        installConfig:
          users:
            etcdUser: etcd
            kineUser: kube-apiserver
            konnectivityUser: konnectivity-server
            kubeAPIserverUser: kube-apiserver
            kubeSchedulerUser: kube-scheduler
        konnectivity:
          adminPort: 8133
          agentPort: 8132
        network:
          kubeProxy:
            disabled: false
            mode: iptables
          kuberouter:
            autoMTU: true
            mtu: 0
            peerRouterASNs: ""
            peerRouterIPs: ""
          podCIDR: 10.244.0.0/16
          provider: kuberouter
          serviceCIDR: 10.96.0.0/12
        podSecurityPolicy:
          defaultPolicy: 00-k0s-privileged
        storage:
          type: etcd
        telemetry:
          enabled: true
    ´´´
