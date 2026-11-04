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
sudo wget https://cdimage.kali.org/kali-2026.3/kali-linux-2026.3-qemu-amd64.7z
````

On doit localiser où se trouve initrd et le kernel pour le boot :

````
isoinfo -J -i <ISO image> -l | grep -e '(initrd|vmlinuz)'
````

Puis on lance l'installation avec les paramètres qui suit (installation sans interface) graphique  

````
virt-install \
  --name kali-vm2 \
  --memory 3072 \
  --vcpus 2 \
  --disk /var/lib/libvirt/images/kali-linux-2026.3-installer-amd64.iso,device=disk,bus=virtio,size=8,pool=default2 \
  --network network=default \
  --location /var/lib/libvirt/images/kali-linux-2026.3-installer-amd64.iso,initrd=install.amd/initrd.gz,kernel=install.amd/vmlinuz \
  --os-type Linux \
  --os-variant ubuntu24.04 \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args 'console=ttyS0,115200n8' \
  --debug 

````


### 3 Commandes pratiques avec virsh 

Lister les vm présentes
````
virsh list --all
````

Lancer, Démarrer, arrêter une vm, forcer l'arrêt 
````
virsh start MaVM

virsh console MaVM

virsh shutdown MaVM

virsh destroy MaVM
````

Supprimer une vm de la liste
````
virsh undefine MaVM
````

##### Lancer son système service

## 3. Industrialisation et supply chain




## Ressources 

Autoriser sur windows la virtualisation (https://stackoverflow.com/questions/70418275/vmware-workstation-unrecoverable-error-vcpu-0)