# KVM - Metasploitable - Kali linux
## Sommaire
1. [Objectifs et contexte](#1--mise-en-place-des-vm)
2. [Etapes d'installation](#2-Preparation-des-environnements)
3. [](#3-Industrialisation-et-supply-chain)


## 1.  Mise en place des vm
### 1.1 Déploiment d'une Vm attaquante et d'une Vm cible.

Contenant un serveur Web, un serveur ftp tout deux avec un certificat TLS. Ainsi que les serveurs DNS, DHCP qui auront pour domaine starfleet.lan, LDAP, MAriaDB, l'utilisation de PHP pour le serveur web avec ngnix

## Plan 

- Installation de qemu,KVM
- Preparation des images de metasploitable et kali linux
- Mettre tout ça dans cloud init (checker les paramètres à configurer)
- Trouver une faille à exploiter (scanner la machine avant l'attaque (snapshot))
- Intégration depuis le web (cockpit)

## 2. Preparation des environnements
### 2.1 Installation de qemu avec KVM

Vérifier si le processeur supporter la virtualisation dans vm (Résultat supérieur à 0 correspondant aux coeurs logiques)
````
grep -Ec '(vmx|svm)' /proc/cpuinfo
````


S'assurer de bien avoir les dépôt mis à jour dans le fichier /etc/apt/sources.list  

Ensuite installation de qemu 

````
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  qemu-system-x86 \
  libvirt-daemon-system \
  libvirt-clients \
  virtinst \
  virt-manager
````

Paquet	                Rôle
qemu-system-x86:    	Émulateur QEMU pour x86, avec accélération KVM
libvirt-daemon-system:	Daemon libvirtd + configuration système
libvirt-clients: 	    Outils CLI (virsh)
virtinst :	            Outil virt-install pour créer des VMs
virt-manager :	        Interface graphique (optionnel sur serveur)

Activation du service libvirt 
````
sudo systemctl enable --now libvirtd
````

Ajout de l'utilisateur courant au groupe kvm et libvirt pour qu'on puisse manipuler nos vm (domaines)
````
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER


````

On configure ensuite l'URI par défaut pour notre réseau 
````
export LIBVIRT_DEFAULT_URI="qemu:///system"
source ~/.bashrc
````
Vérification 
````
virsh uri
# Doit afficher : qemu:///system
````

Installer et vérifier le service libvirt
````
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
````
>>Résultat attendu : Active: active (running)


### 2.2 Verifications Réseau, Stockage 

#### 2.2.1 Réseau
````
virsh net-list --all
````
##### Résultat attendu :
````
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
````

Sinon faire : 

````
virsh net-start default
virsh net-autostart default
````

#### 2.2.2 Stockage
````
virsh pool-list --all

````
##### Résultat si le pool existe déjà  :
````
 Name      State    Autostart
-------------------------------
 default   active   yes
````

Sinon faire 

````
virsh pool-define-as default dir --target /var/lib/libvirt/images
sudo mkdir -p /var/lib/libvirt/images
virsh pool-start default
virsh pool-autostart default
````

Le pool default pointe vers /var/lib/libvirt/images/

### 2.3 Test général

Faire cette commande pour avoir un résumé de différents 
````
sudo virt-host-validate
````

cela doit afficher 
````
QEMU: Checking for hardware virtualization                  : PASS
QEMU: Checking if device /dev/kvm exists                    : PASS
QEMU: Checking if device /dev/kvm is accessible             : PASS
QEMU: Checking if device /dev/vhost-net exists              : PASS
QEMU: Checking if device /dev/net/tun exists                : PASS
QEMU: Checking for cgroup 'memory' controller support       : PASS
QEMU: Checking for device assignment IOMMU support          : WARN (...)
QEMU: Checking for secure guest support                     : WARN (...)
````

>> ATTENTION : hardware virtualization  et  /dev/kvm doivent afficher PASS !!


### 2.4 Installation de Kali linux individuellement en CLI 

On télécharge d'abord notre iso dans le dossier libvirt/image

````
cd /var/lib/libvirt/images
sudo wget https://cdimage.kali.org/kali-2026.3/kali-linux-2026.3-installer-amd64.iso
````

On doit localiser où se trouve initrd et le kernel pour le boot :

````
apt install genisoimage
````
````
isoinfo -J -i kali-linux-2026.3-installer-amd64.iso -l | grep -e '(initrd|vmlinuz)'
````

Puis on lance l'installation avec les paramètres qui suit (installation sans interface) graphique  

````
virt-install \
  --name kali-vm \
  --memory 3072 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/kali-linux-2026.3-installer-amd64.iso,device=disk,bus=virtio,size=8,pool=default \
  --network network=default \
  --location /var/lib/libvirt/images/kali-linux-2026.3-installer-amd64.iso,initrd=install.amd/initrd.gz,kernel=install.amd/vmlinuz \
  --os-type Linux \
  --os-variant ubuntu24.04 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args 'console=ttyS0,115200n8' \
  --debug 


````
Si problème avec le démarrage pendant le boot de GRUB. 

On va faire ensorte d'avoir le terminal sur la sortie  console
1) Au menu démarrage de kali linux, appuyer sur 'e'.
2) Ensuite se deplacer à la ligne linux /vmlinuz....
3) Pour se déplacer, Ctrl+n(ligne suivante) ou Ctrl+p(ligne précédente), Ctrl+f (caractère suivant)
4) Ajouter 'console=ttyS0' après le 'quiet'
5) Sur la vm hôte, mettre 'systemctl enable getty@ttyS0' pour ne pas à refaire toute la manipulation.


Installation avec interface graphique 
````
virt-install \
  --name kali-vm \
  --memory 3072 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/kali-linux-2026.3-installer-amd64.iso,device=disk,bus=virtio,size=8,pool=default \
  --network network=default \
  --location /var/lib/libvirt/images/kali-linux-2026.3-installer-amd64.iso,initrd=install.amd/initrd.gz,kernel=install.amd/vmlinuz \
  --os-type Linux \
  --os-variant ubuntu24.04 \
  --graphics vnc,listen=127.0.0.1 \
  --noautoconsole
````
### 3 Commandes pratiques avec virsh 

Lister les vm présentes
````
virsh list --all
````

Lancer, Démarrer, arrêter une vm, forcer l'arrêt ,supprimer une vm de A à Z  avec son stockage
````
virsh start MaVM

virsh console MaVM

virsh shutdown MaVM

virsh destroy MaVM

virsh undefine mavmDebianTest --remove-all-storage
````

Supprimer une vm de la liste
````
virsh undefine MaVM
````


## CLoud-init
````
sudo apt install -y cloud-init cloud-utils
````

Une fois installée, l'activer et le rendre persistant :
````
sudo systemctl enable cloud-init-local.service cloud-init.service cloud-config.service cloud-final.service
````

Dans le fichier meta-data.yml :
````
instance-id: debian_test-003
local-hostname: debian_test
````

user-data.yml
````
#cloud-config
hostname: debian_test
users:
  - name: alex
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: sudo
    shell: /bin/bash
    ssh_authorized_keys:
    - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIWGNsoqN0+gfvgD/502hQDKru77iOw7z64N1irdcX4Z
package_update: true
packages:
  - wget
  - curl
runcmd:
  - echo "Cloud-init terminé à $(date)" > /var/log/cloud-init-done.log
````

Pour valider la synthaxe du fichier user-data.yml
````
cloud-init schema -c user-data.yml --annotate
````
````
cloud-localds seed.iso user-data.yml meta-data.yml
sudo cp seed.iso /var/lib/libvirt/images/
````
````
sudo qemu-img create -f qcow2 -F qcow2 \
  -b /var/lib/libvirt/images/debian-13-generic-amd64-20261111-2659.qcow2\
  /var/lib/libvirt/images/debian_test.qcow2 8G
  ````
  
  ````
virt-install \
  --name mavmDebianTest \
  --memory 2048 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/debian_test.qcow2 \
  --disk /var/lib/libvirt/images/seed.iso,device=cdrom \
  --network network=default \
  --os-variant ubuntu24.04 \
  --import \
  --graphics vnc \
  --noautoconsole
````
Attention : CLoud-init ne fonctionne pas avec Metasploitable 2
## 3. Industrialisation et supply chain

## Installation Metasploitable 2

dans /var/lib/libvirt/images :
````
wget https://sourceforge.net/projects/metasploitable/files/latest/download

unzip Metasploitable2-Linux.zip

#Convertir l'image vmdk en qcow2 pour une meilleure installation 
qemu-img convert -O qcow2 Metasploitable.vmdk metasploitable.qcow2
````

Donner les permissions nécessaires

Lancer virt-manager

Regarder le fichier .vmx du zip de metasploit pour regarder les parametres de ram et de stockage

Lors de l'installation dans virt-manager
````
-> généric linux 
-> Personnaliser l'installation.
Ensuite dans les options de stockage; choisir SATA ou IDE
Dans les options du processeur :  i440FX et BIOS
````
Lancer Metasploitable

## Supervision avec Cockpit

Installation de cockpit 

````
sudo apt install cockpit-storaged cockpit-networkmanager cockpit-packagekit cockpit-machine cockpit-sosreport virt-viewer -y
````
Puis pour activer le stream des vms :
````
sudo mkdir -p /etc/systemd/system/cockpit.socket.d
sudo touch /etc/systemd/system/cockpit.socket.d/listen.conf
````
Dans ce fichier :
````
[Socket]
ListenStream=
ListenStream=9191
FreeBind=yes
````
On relance tout 
````
sudo systemctl daemon-reload
sudo systemctl stop cockpit cockpit.socket
sudo systemctl start cockpit cockpit.socket
````

Enfin se connecter depuis la machine hôte à l'adresse 
> https://IP_de_Ma_VM:9191


>D'abord fermer les vm de virt-manager et se connecter sur cockpit pour voir les VMs !!



## Ressources 

Autoriser sur windows la virtualisation
https://stackoverflow.com/questions/70418275/vmware-workstation-unrecoverable-error-vcpu-0

Autoriser à lancer Kali linux sur ma vm 
https://unix.stackexchange.com/questions/203768/debian-8-kvm-guest-loading-initial-ramdisk

attaque Kali linux DVWWA 
https://medium.com/@Linuxlearners/exploiting-dvwa-on-metasploitable-2-a-comprehensive-guide-998b6c6b7cc4


https://www.webhi.com/how-to/fr/comment-installer-configurer-kvm-ubuntu-linux/


https://www.stephane-robert.info/docs/virtualiser/type1/kvm/snapshots-clones/

https://blog.zencoffee.org/2016/06/easy-headless-kvm-deployment-virt-install/


https://unix.stackexchange.com/questions/4126/what-is-the-exact-difference-between-a-terminal-a-shell-a-tty-and-a-con

https://www.maravento.com/2022/11/cockpit.html