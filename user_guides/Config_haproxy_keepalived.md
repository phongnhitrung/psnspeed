---
# Hướng dẫn cấu hình High Available Loadblancer

---

## 1. Cấu hình LB Layer7
### 1.1 Cấu hình HAPROXY làm LB layer7 cho IPv4
```
global
        daemon #running haproxy at deamon mode
        maxconn 200000 #maximum connections haproxy can support
        nbproc 1 #threads of haproxy
        cpu-map 1 0 #pin thread 1 of haproxy to cpu 0 of machine 

    defaults
        mode http #default haproxy operate at layer7 (http)
        option http-keep-alive #haproxy keepalive connections => one connection can server multiple requests
        timeout connect 5000ms #Wait 5s for client init connection to haproxy. Drop connections if exceed timeout
        timeout client 50000ms #Wait 50s for client to finish sending a request. Drop connections if exceed timeout
        timeout server 600000ms #Wait 60s for backend to finish sending a repsonse. Drop connections if exceed timeout
        stats uri /haproxy_stats #Access to http://<IP address of haproxy>/haproxy_stats to get statistic information

    frontend http_in_port_80
        bind 0.0.0.0:80 #haproxy listen on port 80 for all active IPv4 address on haproxy machine
        default_backend http_out_port_80 #input traffic will be routed (forwarded) to backends of group http_out_port_80

    backend http_out_ddoswaf_80
        balance source
        hash-type consistent #Use source-hash algorithm for LB
        server backend1 10.10.10.2:80 maxconn 200000 check #Connect and healthcheck to backend1
        server backend2 10.10.10.3:80 maxconn 200000 check #Connect and healthcheck to backend2
```
### 1.2 Cấu hình HAPROXY làm LB Layer7 cho IPv6
```
    frontend http_ipv6_in_port_80
        bind :::80 #haproxy listen on port 80 for all active IPv6 address on haproxy machine
        default_backend http_ipv6_out_port_80

    backend http_ipv6_out_port_80
        balance source
        hash-type consistent
        server backend1 fe80::24a0:6cff:fe4b:9ce1:80 maxconn 200000 check
        server backend2 fe80::24a0:6cff:fe4b:9ce2:80 maxconn 200000 check
```
### 1.3 Thay đổi giải thuật LB
- Giải thuật RoundRobin:
```
    backend http_out_ddoswaf_80
        balance roundrobin
        server backend1 10.10.10.2:80 maxconn 200000 check #Connect and healthcheck to backend1
        server backend2 10.10.10.3:80 maxconn 200000 check #Connect and healthcheck to backend2
```
- Giải thuật Least connection:
```
    backend http_out_ddoswaf_80
        balance leastconn
        server backend1 10.10.10.2:80 maxconn 200000 check #Connect and healthcheck to backend1
        server backend2 10.10.10.3:80 maxconn 200000 check #Connect and healthcheck to backend2 
```

### 1.4 Running
```
haproxy -f <path to haproxy configuration file>
```

---
## 2. Cấu hình LB Layer4
### 2.1 Cấu hình HAPROXY làm LB layer4 cho IPv4
```
global
        daemon #running haproxy at deamon mode
        maxconn 200000 #maximum connections haproxy can support
        nbproc 1 #threads of haproxy
        cpu-map 1 0 #pin thread 1 of haproxy to cpu 0 of machine 

    defaults
        mode http #default haproxy operate at layer7 (http)
        timeout connect 5000ms #Wait 5s for client init connection to haproxy. Drop connections if exceed timeout
        timeout client 50000ms #Wait 50s for client to finish sending a request. Drop connections if exceed timeout
        timeout server 600000ms #Wait 60s for backend to finish sending a repsonse. Drop connections if exceed timeout
        stats uri /haproxy_stats #Access to http://<IP address of haproxy>/haproxy_stats to get statistic information

    frontend http_in_port_80
        mode tcp #haproxy operate at layer4 (tcp)
        bind 0.0.0.0:80 transparent #haproxy listen on port 80 for all active IPv4 address on haproxy machine. Transpartent for source IP of clients
        default_backend http_out_port_80

    backend http_out_port_80
        mode tcp #haproxy operate at layer4 (tcp)
        source 0.0.0.0 usesrc clientip
        balance source
        hash-type consistent #Use source-hash algorithm for LB
        server backend1 10.10.10.2:80 maxconn 200000 check #Connect and healthcheck to backend1
        server backend2 10.10.10.3:80 maxconn 200000 check #Connect and healthcheck to backend2
```
### 2.2 Cấu hình HAPROXY làm LB layer4 cho IPv6
```
    frontend http_ipv6_in_port_80
        mode tcp
        bind :::80 transparent
        default_backend http_ipv6_out_port_80

    backend http_ipv6_out_port_80
        mode tcp
        source 0:0:0:0:0:0:0:0:0 usesrc clientip
        balance source
        hash-type consistent
        server backend1 fe80::24a0:6cff:fe4b:9ce1:80 maxconn 200000 check
        server backend2 fe80::24a0:6cff:fe4b:9ce2:80 maxconn 200000 check
```
### 2.3 Tuning OS và cấu hình iptables, route để transparent source IP trên HAPROXY
- Tuning OS:
```
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
sysctl -p /etc/sysctl.conf
```

- Cấu hình iptables:
```
iptables -t mangle -N DIVERT
iptables -t mangle -A PREROUTING -p tcp -m socket -j DIVERT
iptables -t mangle -A DIVERT -j MARK --set-mark 1
iptables -t mangle -A DIVERT -j ACCEPT
ip6tables -t mangle -N DIVERT
ip6tables -t mangle -A PREROUTING -p tcp -m socket -j DIVERT
ip6tables -t mangle -A DIVERT -j MARK --set-mark 1
ip6tables -t mangle -A DIVERT -j ACCEPT
```

- Cấu hình route:
```
ip rule add fwmark 1 lookup 100
ip -6 rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
ip -6 route add local default dev lo table 100
```

---
## 3. Cấu hình High Available
### 3.1 Pre-setup
Đảm bảo iptables mở kết nối cho giao thức VRRP:
```
iptables -t filter -A INPUT -p vrrp -j ACCEPT
iptables -t filter -A OUTPUT -p vrrp -j ACCEPT
```
### 3.2 Cấu hình High Available sử dụng keepalived
- Cấu hình trên Node Master
```
vrrp_script track1 {
    script "pidof haproxy"
    interval 1 #healthcheck pid haproxy with cycle 1s
    weight -40 #when node master down, its priortiy will be decrease 40 points 
    fall 2 #Healthcheck 2 times, if can not see pid of haproxy => node down
    rise 2 #Healthcheck 2 times, if can see pid of haproxy => node up
}
vrrp_instance INST1 {
    state MASTER
    interface eth4
    virtual_router_id 22
    priority 150 #default for node master is 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LB_WAF_1111
    }
    virtual_ipaddress {
        10.10.10.21/27
    }
    track_script {
        track1
    }
}
```
- Cấu hình trên Node backup
```
vrrp_script track2 {
    script "pidof haproxy"
    interval 1 #healthcheck pid haproxy with cycle 1s
    weight -40 #when node master down, its priortiy will be decrease 40 points 
    fall 2 #Healthcheck 2 times, if can not see pid of haproxy => node down
    rise 2 #Healthcheck 2 times, if can see pid of haproxy => node up
}
vrrp_instance INST1 {
    state BACKUP
    interface eth4
    virtual_router_id 22 #virtual_router_id of node backup must equal to virtual_router_id of node master
    priority 120 #default priority for node master is 120 < defaule priority of node master
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass LB_WAF_1111
    }
    virtual_ipaddress {
        10.10.10.21/27
    }
    track_script {
        track1
    }
}
```

### 3.3 Running keepalived
```
service keepalived start
```
