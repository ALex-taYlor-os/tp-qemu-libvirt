# KVM - Metasploitable - Kali linux
## Sommaire
1. [Mise en place des vm](#1--mise-en-place-des-vm)
2. [Preparation des environnements](#2-preparation-des-environnements)
3. [Commandes avec virsh](#3-commandes-pratiques-avec-virsh)
4. [Cloud init](#4-cloud-init)
5. [Installation de metasploitable 2](#5-installation-metasploitable-2)
6. [Mise en place des vm](#2-preparation-des-environnements)



## 1.  Mise en place des vm
### 1.1 Concept fondamentaux
KVM : laisser les instructions de la machine virtuelle s'exécuter directement sur le processeur physique, Utilise les instructions CPU (Intel VT-x / AMD-V) pour exécuter les VMs à vitesse quasi-native, Chaque VM est isolée des autres grâce au matériel, Les instructions de la VM s'exécutent directement sur le CPU, Pas d'émulation coûteuse

QEMU (Quick Emulator) : émulateur et virtualiseur. Crée réellement les machines virtuelles.
Peut émuler différentes architectures (x86, ARM, etc.) Gère les périphériques : disques virtuels, cartes réseau, USB....

>En pratique, on utilise presque toujours QEMU + KVM pour les VMs Linux/Windows sur serveur x86.

libvirt : couche de gestion qui simplifie l'utilisation de QEMU/KVM (et d'autres hyperviseurs)
API unifiée -> même interface pour KVM, QEMU, Xen, VirtualBox, VMware, Daemon : libvirtd tourne en arrière-plan et gère les VMs, Outils : virsh (CLI), virt-manager (GUI), virt-install (création) ,Stocke les définitions de VMs en XML
````
virsh	           CLI	Gestion en ligne de commande
virt-manager	   GUI	Interface graphique desktop
virt-install	   CLI	Création de VMs scriptable
cockpit-machines   Web	Interface web (moderne)
````
### 1.2 Déploiment d'une Vm attaquante et d'une Vm cible
Contenant un serveur Web, un serveur ftp tout deux avec un certificat TLS. Ainsi que les serveurs DNS, DHCP qui auront pour domaine starfleet.lan, LDAP, MAriaDB, l'utilisation de PHP pour le serveur web avec ngnix

### 1.3 Plan 

- Installation de qemu,KVM
- Preparation des images de metasploitable et kali linux
- Utilisation de cloud init
- Trouver une faille à exploiter 
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
````
````
sudo apt install -y \
  qemu-system-x86 \
  libvirt-daemon-system \
  libvirt-clients \
  virtinst \
  virt-manager
````

````
Paquet	                Rôle
qemu-system-x86:    	Émulateur QEMU pour x86, avec accélération KVM
libvirt-daemon-system:	Daemon libvirtd + configuration système
libvirt-clients: 	    Outils CLI (virsh)
virtinst :	            Outil virt-install pour créer des VMs
virt-manager :	        Interface graphique (optionnel sur serveur)
````
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
## 3. Commandes pratiques avec virsh 

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


## 4. Cloud-init
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

## 5. Installation Metasploitable 2

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


## Snapshot

Lister, créer et lister les snapshot d'une vm
````
virsh domstate linux2024

virsh snapshot-create-as linux2024 --name "before_attack" --description "Etat avant l'utilisation d'une faille avec kali-linux"

virsh snapshot-list linux2024
````

faire un dump au format xml d'un snapsot
````
virsh snapshot-dumpxml linux2024 --snapshotname before_attack
````
````
<domainsnapshot>
  <name>before_attack</name>
  <description>Etat avant l&apos;utilisation d&apos;une faille avec kali-linux</description>
  <state>running</state>
  <creationTime>1794870655</creationTime>
  <memory snapshot='internal'/>
  <disks>
    <disk name='sda' snapshot='internal'/>
  </disks>
  <domain type='kvm'>
    <name>linux2024</name>
    <uuid>856cfc8c-01a5-4e8d-ae44-71da3f2c8cf1</uuid>
    <metadata>
      <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
        <libosinfo:os id="http://libosinfo.org/linux/2024"/>
      </libosinfo:libosinfo>
    </metadata>
    <memory unit='KiB'>1048576</memory>
    <currentMemory unit='KiB'>1048576</currentMemory>
    <vcpu placement='static'>2</vcpu>
    <resource>
      <partition>/machine</partition>
    </resource>
    <os>
      <type arch='x86_64' machine='pc-i440fx-10.0'>hvm</type>
      <bootmenu enable='no'/>
    </os>
    <features>
      <acpi/>
      <apic/>
      <vmport state='off'/>
    </features>
    <cpu mode='host-passthrough' check='none' migratable='on'/>
    <clock offset='utc'>
      <timer name='rtc' tickpolicy='catchup'/>
      <timer name='pit' tickpolicy='delay'/>
      <timer name='hpet' present='no'/>
    </clock>
    <on_poweroff>destroy</on_poweroff>
    <on_reboot>restart</on_reboot>
    <on_crash>destroy</on_crash>
    <pm>
      <suspend-to-mem enabled='no'/>
      <suspend-to-disk enabled='no'/>
    </pm>
    <devices>
      <emulator>/usr/bin/qemu-system-x86_64</emulator>
      <disk type='file' device='disk'>
        <driver name='qemu' type='qcow2'/>
        <source file='/var/lib/libvirt/images/Metasploitable2-Linux/metasploitable.qcow2'/>
        <target dev='sda' bus='sata'/>
        <boot order='1'/>
        <address type='drive' controller='0' bus='0' target='0' unit='0'/>
      </disk>
      <controller type='usb' index='0' model='qemu-xhci' ports='15'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
      </controller>
      <controller type='sata' index='0'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
      </controller>
      <controller type='virtio-serial' index='0'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
      </controller>
      <controller type='pci' index='0' model='pci-root'/>
      <interface type='network'>
        <mac address='52:54:00:2f:61:a3'/>
        <source network='labisolated'/>
        <model type='e1000'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
      </interface>
      <serial type='pty'>
        <target type='isa-serial' port='0'>
          <model name='isa-serial'/>
        </target>
      </serial>
      <console type='pty'>
        <target type='serial' port='0'/>
      </console>
      <channel type='unix'>
        <target type='virtio' name='org.qemu.guest_agent.0'/>
        <address type='virtio-serial' controller='0' bus='0' port='1'/>
      </channel>
      <channel type='spicevmc'>
        <target type='virtio' name='com.redhat.spice.0'/>
        <address type='virtio-serial' controller='0' bus='0' port='2'/>
      </channel>
      <input type='tablet' bus='usb'>
        <address type='usb' bus='0' port='1'/>
      </input>
      <input type='mouse' bus='ps2'/>
      <input type='keyboard' bus='ps2'/>
      <graphics type='spice' autoport='yes'>
        <listen type='address'/>
        <image compression='off'/>
      </graphics>
      <sound model='ich9'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
      </sound>
      <audio id='1' type='spice'/>
      <video>
        <model type='virtio' heads='1' primary='yes'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
      </video>
      <redirdev bus='usb' type='spicevmc'>
        <address type='usb' bus='0' port='2'/>
      </redirdev>
      <redirdev bus='usb' type='spicevmc'>
        <address type='usb' bus='0' port='3'/>
      </redirdev>
      <memballoon model='virtio'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
      </memballoon>
      <rng model='virtio'>
        <backend model='random'>/dev/urandom</backend>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
      </rng>
    </devices>
    <seclabel type='dynamic' model='apparmor' relabel='yes'/>
  </domain>
  <inactiveDomain type='kvm'>
    <name>linux2024</name>
    <uuid>856cfc8c-01a5-4e8d-ae44-71da3f2c8cf1</uuid>
    <metadata>
      <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
        <libosinfo:os id="http://libosinfo.org/linux/2024"/>
      </libosinfo:libosinfo>
    </metadata>
    <memory unit='KiB'>1048576</memory>
    <currentMemory unit='KiB'>1048576</currentMemory>
    <vcpu placement='static'>2</vcpu>
    <os>
      <type arch='x86_64' machine='pc-i440fx-10.0'>hvm</type>
      <bootmenu enable='no'/>
    </os>
    <features>
      <acpi/>
      <apic/>
      <vmport state='off'/>
    </features>
    <cpu mode='host-passthrough' check='none' migratable='on'/>
    <clock offset='utc'>
      <timer name='rtc' tickpolicy='catchup'/>
      <timer name='pit' tickpolicy='delay'/>
      <timer name='hpet' present='no'/>
    </clock>
    <on_poweroff>destroy</on_poweroff>
    <on_reboot>restart</on_reboot>
    <on_crash>destroy</on_crash>
    <pm>
      <suspend-to-mem enabled='no'/>
      <suspend-to-disk enabled='no'/>
    </pm>
    <devices>
      <emulator>/usr/bin/qemu-system-x86_64</emulator>
      <disk type='file' device='disk'>
        <driver name='qemu' type='qcow2'/>
        <source file='/var/lib/libvirt/images/Metasploitable2-Linux/metasploitable.qcow2'/>
        <target dev='sda' bus='sata'/>
        <boot order='1'/>
        <address type='drive' controller='0' bus='0' target='0' unit='0'/>
      </disk>
      <controller type='usb' index='0' model='qemu-xhci' ports='15'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
      </controller>
      <controller type='pci' index='0' model='pci-root'/>
      <controller type='sata' index='0'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
      </controller>
      <controller type='virtio-serial' index='0'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
      </controller>
      <interface type='network'>
        <mac address='52:54:00:2f:61:a3'/>
        <source network='labisolated'/>
        <model type='e1000'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
      </interface>
      <serial type='pty'>
        <target type='isa-serial' port='0'>
          <model name='isa-serial'/>
        </target>
      </serial>
      <console type='pty'>
        <target type='serial' port='0'/>
      </console>
      <channel type='unix'>
        <target type='virtio' name='org.qemu.guest_agent.0'/>
        <address type='virtio-serial' controller='0' bus='0' port='1'/>
      </channel>
      <channel type='spicevmc'>
        <target type='virtio' name='com.redhat.spice.0'/>
        <address type='virtio-serial' controller='0' bus='0' port='2'/>
      </channel>
      <input type='tablet' bus='usb'>
        <address type='usb' bus='0' port='1'/>
      </input>
      <input type='mouse' bus='ps2'/>
      <input type='keyboard' bus='ps2'/>
      <graphics type='spice' autoport='yes'>
        <listen type='address'/>
        <image compression='off'/>
      </graphics>
      <sound model='ich9'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
      </sound>
      <audio id='1' type='spice'/>
      <video>
        <model type='virtio' heads='1' primary='yes'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
      </video>
      <redirdev bus='usb' type='spicevmc'>
        <address type='usb' bus='0' port='2'/>
      </redirdev>
      <redirdev bus='usb' type='spicevmc'>
        <address type='usb' bus='0' port='3'/>
      </redirdev>
      <memballoon model='virtio'>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
      </memballoon>
      <rng model='virtio'>
        <backend model='random'>/dev/urandom</backend>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x09' function='0x0'/>
      </rng>
    </devices>
  </inactiveDomain>
  <cookie>
    <cpu mode='host-passthrough' check='none' migratable='on'/>
    <slirpHelper/>
  </cookie>
</domainsnapshot>
````
````
virsh snapshot-revert linux2024 --snapshotname before_attack
````
Le vrai coût d'une chaîne longue est ailleurs, et il est bien réel : l'espace que personne ne récupère, la complexité de savoir quelle couche porte quoi, et le risque de manipulation au moment de consolider. La règle de conduite ne change donc pas : créez un snapshot, testez, puis supprimez-le ou consolidez la chaîne. Elle se justifie par l'hygiène, pas par un seuil de performance.

Pour importer une vm au format xml : 
````
virsh define /path/to/xmlfile.xml
````
>Bien faire attention à ce que le chemin source dans la balise disk, pointe bien vers une image qcow2 !

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

Enfin se connecter depuis la machine hôte à l'adresse :
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