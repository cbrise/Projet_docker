# Projet_docker

INTRODUCTION

Mise en place d'un docker swarm de 3 noeuds ainsi que une VIP (Virtual IP) avec Keepalived et Glusterfs pour avoir une haute disponibilité de nos applications.
Nos applications sont un wordpress avec MySQL avec un Prometheus associé à un Grafana pour avoir des tableaux visuels de nos métriques. Nous devons avoir les alertes qui remontent sur un serveur Discord.



# 1 Mise en place d'un Docker Swarm

root@sc28:~# docker swarm init --advertise-addr {172.20.107.140} --data-path-port=7789

root@sc28:~# docker swarm join-token manager

root@sc29:~# docker swarm join --token SWMTKN-1-38g9iwl49s1c8i8wdtyxlbq2825zmhervpnlscar1std5uyd7g-6g6wyguylbbg1y05ttzljsbgo 172.20.107.140:2377

root@sc30:~# docker swarm join --token SWMTKN-1-38g9iwl49s1c8i8wdtyxlbq2825zmhervpnlscar1std5uyd7g-6g6wyguylbbg1y05ttzljsbgo 172.20.107.140:2377

root@sc29:~# docker node promote d3mnprtz7r0vhs865z5ginlnc

root@sc30:~# docker node promote 0n1mekkny0iyn8zksu9xk5089

root@sc28:~# docker node ls 

ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION

k7a1xfbvi5nzfygzdjeh1am3z *   sc28       Ready     Active         Leader           27.0.3

0n1mekkny0iyn8zksu9xk5089     sc29       Ready     Active         Reachable        20.10.24+dfsg1

d3mnprtz7r0vhs865z5ginlnc     sc30       Ready     Active         Reachable        20.10.24+dfsg1



# 2 Mise en place du Keepalived

root@sc28:~# apt-get install keepalived

root@sc28:~# nano /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {
    state MASTER
    
    interface ens192
    
    virtual_router_id 51
    
    priority 100
    
    advert_int 1
    
    authentication {
    
        auth_type PASS
        
        auth_pass 12345
        
    }
    
    virtual_ipaddress {
    
        172.20.107.143
        
    } 
}

root@sc29:~# apt-get install keepalived

root@sc29:~# nano /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {
    state BACKUP
    
    interface ens192
    
    virtual_router_id 51
    
    priority 90
    
    advert_int 1
    
    authentication {
    
        auth_type PASS
        
        auth_pass 12345
        
    }
    
    virtual_ipaddress {
    
        172.20.107.143
        
    }
}

root@sc30:~# apt-get install keepalived

root@sc30:~# nano /etc/keepalived/keepalived.conf

vrrp_instance VI_1 {
    state BACKUP
    
    interface ens192
    
    virtual_router_id 51
    
    priority 90
    
    advert_int 1
    
    authentication {
    
        auth_type PASS
        
        auth_pass 12345
        
    }
    
    virtual_ipaddress {
    
        172.20.107.143
        
    }
}



# 3 Mise en place de Glusterfs

Prérequis :

    - Avoir le paquet xfsprogs d'installer
    - Avoir un second disque physique monté 
    
Il faut ensuite déclarer le DNS des serveurs du cluster : 

root@sc28:~# cat /etc/hosts

127.0.0.1       localhost
127.0.1.1       sc28
172.20.107.141  sc29
172.20.107.142  sc30

root@sc29:~# cat /etc/hosts

127.0.0.1       localhost
127.0.1.1       sc29
172.20.107.140  sc28
172.20.107.142  sc30


root@sc30:~# cat /etc/hosts

127.0.0.1       localhost
127.0.1.1       sc30
172.20.107.140  sc28
172.20.107.141  sc29

Sur chaque serveur faire cette commande (adapté le nom du disque au besoin):
mkfs.xfs /dev/sdb

Faire ces commandes :

1er serveur:
root@sc28:~# mkfs.xfs /dev/sdb

root@sc28:~# mkdir -p /gluster/bricks/1

root@sc28:~# echo '/dev/sdb /gluster/bricks/1 xfs defaults 0 0' >> /etc/fstab

root@sc28:~# mount -a

montage : (astuce) votre fstab a été modifié mais systemd utilise encore
       l'ancienne version ; utilisez « systemctl daemon-reload » pour recharger.
root@sc28:~# systemctl daemon-reload

root@sc28:~# mkdir /gluster/bricks/1/brick

2e serveur:
root@sc29:~# mkdir -p /gluster/bricks/2

root@sc29:~# echo '/dev/sdb /gluster/bricks/2 xfs defaults 0 0' >> /etc/fstab

root@sc29:~# mount -a
montage : (astuce) votre fstab a été modifié mais systemd utilise encore
       l'ancienne version ; utilisez « systemctl daemon-reload » pour recharger.
       
root@sc29:~# systemctl daemon-reload

root@sc29:~# mount -a

root@sc29:~# mkdir /gluster/bricks/2/brick

root@sc29:~#

3e serveur:
root@sc30:~# mkdir -p /gluster/bricks/3

root@sc30:~# echo '/dev/sdb /gluster/bricks/3 xfs defaults 0 0' >> /etc/fstab

root@sc30:~# mount -a
montage : (astuce) votre fstab a été modifié mais systemd utilise encore
       l'ancienne version ; utilisez « systemctl daemon-reload » pour recharger.
       
root@sc30:~# systemctl daemon-reload

root@sc30:~# mkdir /gluster/bricks/3/brick

root@sc30:~#

Sur chaque serveurs :
apt install glusterfs-server
systemctl enable glusterd
systemctl start glusterd
systemctl status glusterd


Ajouter les deux membres du cluster, voici un exemple:
root@sc29:~# gluster peer probe sc30
peer probe: success

root@sc29:~# gluster peer probe sc28
peer probe: success

Pour vérifier le nombre de membres :
root@sc29:~# gluster peer status
Number of Peers: 2

Hostname: sc30
Uuid: 3ae89ad4-1196-4fa8-89a0-8dcefb56cc02
State: Peer in Cluster (Connected)

Hostname: sc28
Uuid: 00fe50ba-7715-4c95-b24c-81f0e35ddc6a
State: Peer in Cluster (Connected)


Création du volume Glusterfs:
root@sc29:~# gluster volume create gfs \
replica 3 \
sc28:/gluster/bricks/1/brick \
sc29:/gluster/bricks/2/brick \
sc30:/gluster/bricks/3/brick

root@sc29:~# gluster volume start gfs
volume start: gfs: success

root@sc29:~# gluster volume status gfs
Status of volume: gfs
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick sc28:/gluster/bricks/1/brick          50393     0          Y       11618
Brick sc29:/gluster/bricks/2/brick          49472     0          Y       15689
Brick sc30:/gluster/bricks/3/brick          52203     0          Y       15043
Self-heal Daemon on localhost               N/A       N/A        Y       15706
Self-heal Daemon on sc28                    N/A       N/A        Y       11635
Self-heal Daemon on sc30                    N/A       N/A        Y       15060

Task Status of Volume gfs
------------------------------------------------------------------------------
There are no active volume tasks


Autorisation des membres du cluster a acceder au volume :
root@sc29:~# gluster volume set gfs auth.allow 172.20.107.140,172.20.107.141,172.20.107.142
volume set: success


Déclaration des points de montage pour le boot des serveurs :
root@sc28:~# echo 'localhost:/gfs /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab

root@sc28:~# mount.glusterfs localhost:/gfs /mnt

root@sc29:~# echo 'localhost:/gfs /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab

root@sc29:~# mount.glusterfs localhost:/gfs /mnt

root@sc30:~# echo 'localhost:/gfs /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab

root@sc30:~# mount.glusterfs localhost:/gfs /mnt



# 4 Mise en place d'un Wordpress avec MySQL
Il faut coller le répertoire /mnt/wordpress sur le volume Glusterfs, puis dans ce répertoire, créer le répertoire mysql et le répertoire wp-content

Pour  lancer le serveur wordpress, aller dans le répertoire wordpress (celui qui est à la base du projet) et faire cette commande: 
     docker stack deploy -c docker-compose.yml wordpress



# 5 Mise en place d'un Prometheus avec des notifications Discord

Prérequis :

    - Il faut coller le répertoire /mnt/prometheus dans le volume Glusterfs
    - Dans le fichier /mnt/prometheus/alertmanager.yml : Il faut modifier la valeur de la variable webhook_url par celle de son propre webhook.
    - Dans le fichier /mnt/prometheus/prometheus.yml : Il faut modifier les variables "targets" par les IP des hôtes du cluster swarm ( Rajouter des - si plus de membres)
      - job_name: 'cadvisor'
    static_configs:
      - targets:
        - 'IP_Hôte_1:8080'
        - 'IP_Hôte_2:8080'
        - 'IP_Hôte_3:8080'
 
Pour lancer la récolte des données des membres de la stack et des conteneurs, aller dans le répertoire Prometheus (celui qui est à la base du projet) et faire cette commande:
    - docker stack deploy -c cadvisor-stack.yml cadvisor

Pour lancer les conteneurs permettant la supervision et l'alerting, il faut lancer cette commande :
docker stack deploy -c docker-compose.yml prometheus





