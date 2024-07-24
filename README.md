# Projet_docker

INTRODUCTION

Mise en place d'un docker swarm de 3 noeuds ainsi que une VIP (Virtual IP) avec Keepalived pour avoir une haute disponibilité de nos applications.
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

# 3 Mise en place d'un Wordpress avec MySQL

# 4 Mise en place d'un Prometheus avec Grafana

# 5 Mise en place des alertes vers discord.


