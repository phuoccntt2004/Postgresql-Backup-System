## Hệ thống backup CSDL

### Mục lục

[I. Setup](#setup)
- [1. Cài posgreSQL cho các node](#installedpostgresql)
- [2. Cài patroni và cấu hình các node liên kết với nhau](#configpatroni)
- [3. Cấu hình keepalived](#keepalived)

[II. Setup minio cluster](#minio)
- [1. Phân vùng ổ đĩa](#disk)
- [2. Cài minio và cấu hình](#configminio)
- [3. Cấu hình loadblance và keepalived](#configproxy)

[II. Monitor](#monitor)
- [1. Setup các export để lấy metrics từ HĐH, PosgreSQL, Patroni, Minio](#metrics)
- [2. Cài grafana và prometheus](#grafanaprometheus)

===========================

| Ip  | Node | username 
|--------------|-------|------
| 192.168.0.181 | postgres1 | pgprimary |  
| 192.168.0.182 | postgres2 | pgreplica1 | 
| 192.168.0.183 | postgres3 | pgreplica2 |
| 192.168.0.184 | minio1 | minio1 | 
| 192.168.0.185 | minio2 | minio2 | 
| 192.168.0.186 | minio3 | minio3 | 
| 192.168.0.187 | minio4 | minio4 | 
| 192.168.0.188 | monitor | monitor | 

[Mô hình các node](https://drive.google.com/file/d/1lOnaO0WrTvPQIZ2Fn4rRJNmQgQ9r7-pO/view?usp=sharing)

🚀 Hệ thống được thiết kế với khả năng dự phòng toàn diện cho PostgreSQL.
Khi primary gặp sự cố, Patroni sẽ tự động promote replica lên thành primary mới. Replica còn lại được dùng chuyên biệt cho việc backup lên MinIO và giám sát hệ điều hành, PostgreSQL, Patroni và MinIO.

🛡️ PostgreSQL + Keepalived (VIP High Availability)
Hai node PostgreSQL sử dụng Keepalived để cung cấp một địa chỉ VIP dùng chung.
- Khi một node lỗi → VIP tự động chuyển sang node còn lại.
- Keepalived được cấu hình thêm kiểm tra role nhằm tránh tình huống node hồi sinh với ưu tiên cao hơn nhưng không còn là primary (vì Patroni khi node sống lại chỉ cho phép nó trở thành replica).
➡️ Điều này đảm bảo VIP luôn trỏ đúng vào node primary thực sự.

📦 MinIO High Availability
Hệ thống MinIO được triển khai kèm load balancer và Keepalived để đảm bảo tính dự phòng và khả dụng cao.

💾 Backup định kỳ
Dữ liệu được thực hiện backup từ replica chuyên dụng và được upload lên MinIO theo lịch crontab đã cấu hình sẵn.
<a name="setup"></a>
## I. Setup

<a name="installedpostgresql"></a>
### 1. Cài posgreSQL cho các node

```
  sudo apt update
  sudo apt install postgresql postgresql-contrib
```
Sau đó xem trạng thái serivce của postgreSQL của 3 node
```
  sudo systemctl status postgresql
```
<a name="configpatroni"></a>
### 2. Cài patroni và cấu hình các node liên kết với nhau

Cài đặt các gói python để hỗ trợ patroni và etcd
```
  sudo apt install python3-pip python3-dev binutils -y
```

Cài đặt patroni, etcd 
```
  sudo apt install patroni  -y
```
Cài đặt etcd từ source, chuyển các thư mục `~/etcd/bin` vào `/usr/local/bin/` và phân quyền root các thư mục đó
```
  sudo apt install golang-go -y
  git clone -b v3.5.21 https://github.com/etcd-io/etcd.git
  cd etcd
  ./build.sh
  export PATH="$PATH:`pwd`/bin"
  sudo cp ~/etcd/bin/etcd /usr/local/bin/
  sudo cp ~/etcd/bin/etcdctl /usr/local/bin/
  sudo cp ~/etcd/bin/etcdutl /usr/local/bin/
  sudo chmod 755 /usr/local/bin/etcd /usr/local/bin/etcdctl /usr/local/bin/etcdutl
  sudo chown root:root /usr/local/bin/etcd /usr/local/bin/etcdctl /usr/local/bin/etcdutl
```
- Patroni trong posgreSQL để quản lý primary và replica và tự động bầu chọn primary mới từ replica khi primary chết
- etcd là hệ thống lưu trữ thông tin primary, giúp các node biết ai là primary và xử lý failover (chuyển đổi dự phòng) an toàn

Xoá thư mục cài sẵn của postgreSQL vì  patroni tự tạo lại để patroni dễ quản lí và cần trống để tránh xung đột
```
  sudo rm -rf /var/lib/postgresql/16/main/
```
Tạo thư mục và phân quyền `/data/etcd/postgresql/` để etcd lưu trữ các dữ liệu cluster
```
  sudo mkdir -p /data/etcd/postgresql
  sudo chown root:root /data/etcd/postgresql
  sudo chmod 700 /data/etcd/postgresql
```
Tạo file systemd cho service `etcd`
```
  sudo vim /etc/systemd/system/etcd.service
```
```
  [Unit]
  Description=etcd key-value store
  Documentation=https://github.com/etcd-io/etcd
  After=network.target

  [Service]
  Type=notify
  EnvironmentFile=/etc/default/etcd
  User=root
  ExecStart=/usr/local/bin/etcd
  Restart=always
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
```
#### Configure postgres1 (leader/primary)
Tạo file environment phù hợp cho `posgres1`
```
  sudo vim /etc/default/etcd
```
```
  ETCD_NAME=postgres1          # Tên node
  ETCD_INITIAL_CLUSTER="postgres1=http://192.168.0.181:2380"  # Danh sách node cluster
  ETCD_INITIAL_CLUSTER_TOKEN="PostgreSQL_HA_Cluster"          # Token cluster
  ETCD_INITIAL_CLUSTER_STATE="new"         # Trạng thái cluster (new/existing)
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.181:2380"  # URL peer quảng bá
  ETCD_DATA_DIR="/data/etcd/postgresql"    # Thư mục dữ liệu
  ETCD_LISTEN_PEER_URLS="http://192.168.0.181:2380"          # URL peer lắng nghe
  ETCD_LISTEN_CLIENT_URLS="http://192.168.0.181:2379,http://localhost:2379"  # URL client lắng nghe
  ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.181:2379"    # URL client quảng bá
```

Khởi động lại dịch vụ etcd để áp dụng các thay đổi

```
  sudo systemctl daemon-reload
  sudo systemctl enable --now etcd
  sudo systemctl start etcd
  sudo systemctl status etcd
```

Tiếp theo kiểm tra leader(primary) và status

```
  ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379 endpoint status -w table
  ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379 endpoint health

```
<img src="./assets/1.PNG">

#### Configure postgres2 (replica1)

Tạo file environment phù hợp cho `posgres2`
```
  sudo vim /etc/default/etcd
```
```
  ETCD_NAME=postgres2                     # Tên node
  ETCD_INITIAL_CLUSTER="postgres1=http://192.168.0.181:2380,postgres2=http://192.168.0.182:2380"  # Danh sách tất cả node cluster
  ETCD_INITIAL_CLUSTER_TOKEN="PostgreSQL_HA_Cluster"          # Token cluster
  ETCD_INITIAL_CLUSTER_STATE="existing"   # Trạng thái cluster (new/existing)
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.182:2380"  # URL peer quảng bá
  ETCD_DATA_DIR="/data/etcd/postgresql"   # Thư mục dữ liệu node
  ETCD_LISTEN_PEER_URLS="http://192.168.0.182:2380"          # URL peer lắng nghe
  ETCD_LISTEN_CLIENT_URLS="http://192.168.0.182:2379,http://localhost:2379"  # URL client lắng nghe
  ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.182:2379"    # URL client quảng bá
```
Khởi động lại dịch vụ etcd để áp dụng các thay đổi

```
  sudo systemctl daemon-reload
  sudo systemctl enable --now etcd
  sudo systemctl start etcd
```

Tiếp theo quay lại máy `postgreSQL1(leader/primary)` để thêm member
```
  ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379 member add postgres2 --peer-urls=http://192.168.0.182:2380
```
Sau đó kiểm tra member đã thêm vào chưa
```
  ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379 member list
```
Sau khi thêm thành công quay lại `posgres2` để restart lại service và kiểm tra
```
  sudo systemctl restart etcd
  sudo systemctl status etcd
```
#### Configure postgres3 (replica2)

Tạo file environment phù hợp cho `posgres3`
```
  sudo vim /etc/default/etcd
```
```
  ETCD_NAME=postgres3                     # Tên node
  ETCD_INITIAL_CLUSTER="postgres1=http://192.168.0.181:2380,postgres2=http://192.168.0.182:2380,postgres3=http://192.168.0.183:2380"  # Danh sách tất cả node cluster
  ETCD_INITIAL_CLUSTER_TOKEN="PostgreSQL_HA_Cluster"          # Token cluster
  ETCD_INITIAL_CLUSTER_STATE="existing"   # Trạng thái cluster (new/existing)
  ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.183:2380"  # URL peer quảng bá
  ETCD_DATA_DIR="/data/etcd/postgresql"   # Thư mục dữ liệu node
  ETCD_LISTEN_PEER_URLS="http://192.168.0.183:2380"          # URL peer lắng nghe
  ETCD_LISTEN_CLIENT_URLS="http://192.168.0.183:2379,http://localhost:2379"  # URL client lắng nghe
  ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.183:2379"    # URL client quảng bá
```
Khởi động lại dịch vụ etcd để áp dụng các thay đổi

```
  sudo systemctl daemon-reload
  sudo systemctl enable --now etcd
  sudo systemctl start etcd
```
Tiếp theo quay lại máy `postgreSQL1(leader/primary)` để thêm member
```
  ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379 member add postgres3 --peer-urls=http://192.168.0.183:2380
```
Sau đó kiểm tra member đã thêm vào chưa
```
  ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379 member list
```
Sau khi thêm thành công quay lại `posgres3` để restart lại service và kiểm tra
```
  sudo systemctl restart etcd
  sudo systemctl status etcd
```
##### posgres1(leader/primary)
Kiểm tra ai là leader bên `postgres1`, nếu ai có `IS LEADER = true` thì đó là leader
```
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.181:2379,http://192.168.0.182:2379,http://192.168.0.183:2379 endpoint status --write-out=table
```
<img src="./assets/2.PNG" alt="1">
Nếu leader sai không đúng với node thì chuyển `24ac94b43c79eb69` này là id của member cần lên leader
```
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.182:2379 move-leader 24ac94b43c79eb69
```

#### Configure patroni trên 3 node
Tạo thư mục và phân quyền để patroni quản lý cluster
```
  sudo mkdir -p /data/postgresql/16/main
  sudo chown -R postgres /data/postgresql
  sudo chmod -R 700 /data/postgresql/16/main
```
Tiếp thêm các biến môi trường để tạo file config patroni
```
  DATA_DIR="/data/postgresql/16/main"
  PG_BIN_DIR="/usr/lib/postgresql/16/bin"
  NAMESPACE="prod_env"
  SCOPE="cluster1"
  NODE_NAME=postgres1     #(postgres1/postgres3)cái này thì đặt name giống name của etcd đã cấu hình trên node nào thì name đó
  NODE_IP=192.168.0.181   #(192.168.0.182/192.168.0.183) ip này cũng vậy đặt theo ip của node đó
```
Tiếp theo file `secret` cho posgreSQL quản lý các key bí mật

```
  sudo mkdir -p /opt/secret
  sudo chmod -R 700 /opt/secret
  sudo chown -R postgres /opt/secret
```

Tiếp theo dùng `echo` để tạo file có cấu hình cho patroni

```
echo "
namespace: ${NAMESPACE}
scope: ${SCOPE}
name: ${NODE_NAME}

restapi:
    listen: 0.0.0.0:8008
    connect_address: ${NODE_IP}:8008

etcd3:
    host: ${NODE_IP}:2379

bootstrap:
  # this section will be written into Etcd:/
<namespace>/<scope>/config after initializing new cluster
  dcs:
      ttl: 30
      loop_wait: 10
      retry_timeout: 10
      maximum_lag_on_failover: 1048576
      slots:
          repl_slot:
            type: physical

      postgresql:
          use_pg_rewind: true
          use_slots: true
          parameters:
              wal_level: replica
              hot_standby: "on"
              wal_keep_segments: 10
              max_wal_senders: 5
              max_replication_slots: 10
              wal_log_hints: "on"
              logging_collector: 'on'
              max_connections: 512

  # some desired options for 'initdb'
  initdb: # Note: It needs to be a list (some options need values, others are switches)
      - encoding: UTF8
      - data-checksums

  pg_hba: # Add following lines to pg_hba.conf after running 'initdb'
      - host replication replicator 127.0.0.1/32 trust
      - host replication replicator 0.0.0.0/0 md5
      - host all all 0.0.0.0/0 md5
      - host all all ::0/0 md5

postgresql:
    cluster_name: cluster_1
    listen: 0.0.0.0:5432
    connect_address: ${NODE_IP}:5432
    data_dir: ${DATA_DIR}
    bin_dir: ${PG_BIN_DIR}
    pgpass: /opt/secret/pgpass
    authentication:
        replication:
            username: replicator
            password: replPasswd
        superuser:
            username: postgres
            password: aMu3c9QKKUGYQZK
    parameters:
        unix_socket_directories: "/var/run/postgresql/"
    create_replica_methods:
        - basebackup
    basebackup:
        checkpoint: 'fast'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
" | sudo tee -a /etc/patroni/config.yml
```
Chọn 1 node riêng chỉ để backup sửa không lên primary 
```
  nofailover: true
  noloadbalance: true
```
Sau đó cài enable và restart dịch vụ patroni
```
  sudo systemctl enable --now patroni
  sudo systemctl restart patroni
  sudo systemctl status patroni
```
Tiếp theo kiểm tra patroni cluster
```
  patronictl -c /etc/patroni/config.yml list $SCOPE
```
<img src="./assets/3.PNG" alt="3">

Tiếp theo kiểm tra replica của postgres
```
  sudo psql -U postgres
  SELECT * FROM pg_replication_slots;
```
<img src="./assets/4.PNG" alt="4">

<a name="keepalived"></a>
### 3. Cấu hình keepalived

Tiếp theo cài keepalived lên `postgres1 (primary)` và `postgres2 (replica)` để có thể dự phòng nếu primary chết, vì do có 2 node nên không làm nginx làm keepalived vì patroni có thể tự động chuyển đổi replica thành primary nếu primary chết, nên chỉ cần setup keepalive thêm 1 script để nếu primary sống lại VIP cũng không thể ưu tiên được exit luôn trừ khi nó role master thì nó mới chạy tiếp. Mặc định VIP nếu nó chạy lại được thì nó sẽ ưu tiên theo điểm nên buộc phải có thêm script để khi nó chạy lại primary kia không còn master thì nó cũng không được ưu tiên theo điểm
Đoạn script để kiểm tra role làm trên 2 node
```
  sudo vim /usr/local/bin/check_patroni.sh
```
```
/usr/local/bin/check_patroni.sh
```
Cài keepalived trên 2 node
```
  sudo apt -y install keepalived
```
```
  sudo vim /etc/keepalived/keepalived.conf
```
###### Cấu hình keepalived cho node postgres1(primary)
```
vrrp_script chk_patroni {
    script "/usr/local/bin/check_patroni.sh"
    interval 2
    fall 2
    rise 2
}

vrrp_instance VI_WRITE {
    state MASTER
    interface enp0s3
    virtual_router_id 51
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        192.168.0.200/24
    }

    track_script {
        chk_patroni
    }
}


```
###### Cấu hình keepalived cho node postgres2(replica)
```
vrrp_script chk_patroni {
    script "/usr/local/bin/check_patroni.sh"
    interval 2
    fall 2
    rise 2
}

vrrp_instance VI_WRITE {
    state BACKUP
    interface enp0s3
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1234
    }

    virtual_ipaddress {
        192.168.0.180/24
    }

    track_script {
        chk_patroni
    }
}
```
##### PostgreSQL
`psql -h 192.168.0.180 -U postgres`
`username: postgres`
`password: aMu3c9QKKUGYQZK`

<a name="minio"></a>
## III. Setup minio cluster (4 node)

<a name="disk"></a>
### 1. Phân vùng ổ đĩa
##### Thiết lập các cài đặt trên 4 node
- cTrước tiên tạo các ổ đĩa ảo cho các máy ảo

<img src="./assets/5.PNG" alt="5">

- Tạo folder chung để lưu trữ của minio
```
  sudo mkdir -p /mnt/data
```
- Phân quyền folder đó
```
  sudo chown -R root:root /mnt/data
  sudo chmod -R 777 /mnt/data
```
- Tạo phần vùng ổ đĩa 
```
  sudo fdisk /dev/sdb
```
- Nhấn n (new partition).
- Nhấn p (primary partition)
- Nhấn w (write) để lưu các thay đổi và thoát.
- Định dạng ổ đĩa
```
  sudo mkfs.ext4 /dev/sdb1
```
- Mount ổ đĩa đó lại với thư mục đó
```
  sudo mount /dev/sdb1 /mnt/data
```
<a name="configminio"></a>
### 2. Cài minio và cấu hình
##### Cài đặt minio trên 4 node

```
  wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio.RELEASE.2024-12-18T13-15-44Z
  mv minio.RELEASE.2024-12-18T13-15-44Z minio
  chmod +x minio
  sudo mv minio /usr/local/bin/
```
Tiếp theo tạo file chưacs các biến môi trường để chạy minio trên 4 node
```
  sudo vim /etc/default/minio
```
```
MINIO_VOLUMES="http://192.168.0.184:9000/mnt/data http://192.168.0.185:9000/mnt/data http://192.168.0.186:9000/mnt/data http://192.168.0.187:9000/mnt/data"
MINIO_OPTS="--address :9000 --console-address :40835"
MINIO_ROOT_USER="admin"
MINIO_ROOT_PASSWORD="phuocdaica624"
MINIO_SERVER_URL="http://192.168.0.184:9000"
MINIO_PROMETHEUS_AUTH_TYPE="public"
```
Tiếp theo tạo file systemd để chạy minio bằng lệnh systemctl và kết nối file môi trường ổ đĩa mnt/data  trên 4 node
```
[Unit]
Description=MinIO
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio
RequiresMountsFor=/mnt/data

[Service]
User=root
Group=root
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
Sau đó reload service khởi động dịch vụ và enable minio
```
  sudo systemctl daemon-reload
  sudo systemctl enable minio
  sudo systemctl start minio
  sudo systemctl status minio
```
Dùng mc để kiểm tra thử đủ 4 node minio chưa
Cài đặt mc trên node nào cũng được
```
  wget https://dl.min.io/client/mc/release/linux-amd64/mc -O mc
  chmod +x mc
  sudo mv mc /usr/local/bin/mc
```
Sau đó kiểm tra
```
  mc alias set myminio http://192.168.0.184:9000 admin phuocdaica624
```
```
  mc admin info myminio
```
<img src="./assets/6.PNG">

<a name="configproxy"></a>
### 3. Cấu hình loadblance và keepalived
Vì tài nguyên có hạn nên làm luôn 2 loadblance và keepalived lên node `minio1` và `minio2`
Trước tiên cài nginx và keepalived cài trên 2 node

```
  sudo apt install nginx -y
  sudo apt -y install keepalived
```

Cấu hình nginx trên 2 node `minio1` và `minio2`
```
  sudo vim /etc/nginx/sites-enabled/default
```

```
upstream minio{
        server 192.168.0.184:9000;
        server 192.168.0.185:9000;
        server 192.168.0.186:9000;
        server 192.168.0.187:9000;

}
```

```
location / {
        # First attempt to serve request as file, then
         as directoiry, then fall back to displaying a 404.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://minio;
}
```

<img src="./assets/7.PNG" alt="7">

##### Tiếp theo cấu hình keepalived trên `minio1`

```
  sudo vim /etc/keepalived/keepalived.conf
```

```
global_defs {
    router_id node1
}

vrrp_script chk_nginx {              # Định nghĩa script dùng để kiểm tra dịch vụ nginx
    script "/usr/bin/pgrep nginx"    # Chạy lệnh pgrep để xem nginx có đang chạy không (0 = chạy, 1 = chết)
    interval 2                        # Cứ mỗi 2 giây chạy script một lần
    weight -80                        # Nếu script fail thì trừ 80 điểm priority của node
    fall 2                            # Nếu script fail 2 lần liên tiếp thì coi như nginx đã chết
    rise 2                            # Nếu script success 2 lần liên tiếp thì coi nginx đã sống lại
}

vrrp_instance VRRP1 {
    state MASTER
    interface enp0s3
    virtual_router_id 101
    priority 200
    advert_int 1

    virtual_ipaddress {
        192.168.0.190/24
    }

    track_script {
        chk_nginx
    }
}
```

Khởi động lại dịch vụ keepalived
```
  sudo systemctl restart keepalived.service
  sudo systemctl status keepalived.service
```

##### Tiếp theo cấu hình keepalived trên `minio2`

```
global_defs {
    router_id node2
}

vrrp_script chk_nginx {              # Định nghĩa script dùng để kiểm tra dịch vụ nginx
    script "/usr/bin/pgrep nginx"    # Chạy lệnh pgrep để xem nginx có đang chạy không (0 = chạy, 1 = chết)
    interval 2                        # Cứ mỗi 2 giây chạy script một lần
    weight -80                        # Nếu script fail thì trừ 80 điểm priority của node
    fall 2                            # Nếu script fail 2 lần liên tiếp thì coi như nginx đã chết
    rise 2                            # Nếu script success 2 lần liên tiếp thì coi nginx đã sống lại
}

vrrp_instance VRRP1 {
    state BACKUP
    interface enp0s3
    virtual_router_id 101
    priority 150
    advert_int 1

    virtual_ipaddress {
        192.168.0.190/24
    }

    track_script {
        chk_nginx
    }
}
```

Khởi động lại dịch vụ keepalived
```
  sudo systemctl restart keepalived.service
  sudo systemctl status keepalived.service
```
Có cấu hình 2 con keepalived nếu nginx chết trừ 80 điểm và nó kiểm tra liên tục 2 giây khi nào fail 2 lần nó sẽ trừ điểm giúp nếu nginx chết sẽ chuyển sang con còn lại

Tiếp theo backup postgreSQL lên minio
Tạo bucket để lưu file backup posgreSQL.sql
Tạo access key trên minio và cấp phép policy cho key

```
{
 "Version": "2012-10-17",
 "Statement": [
  {
   "Effect": "Allow",
   "Action": [
    "s3:*"
   ],
   "Resource": [
    "arn:aws:s3:::posgresql",
    "arn:aws:s3:::posgresql/*"
   ]
  }
 ]
}
```
##### Thực hiện trên node postgres3(pgreplica2/backup)
Trước tiên test upload file lên minio bằng python

```
  sudo apt install python3-pip
  pip3 install minio --break-system-packages
```


Tạo file python chứa đoạn mã
```
# file_uploader.py MinIO Python SDK example
from minio import Minio
from minio.error import S3Error

def main():
    # Create a client with the MinIO server playground, its access key
    # and secret key.
    client = Minio(
        endpoint="192.168.0.190:9000",
        access_key="nrFeyC4XBPq3MaIO8mlv",
        secret_key="1CGUr7R5KZ4h57PmJFapZQKDD2OICwIZsOGYHpPq",
        secure=False
    )

    # The file to upload, change this path if needed
    source_file = "/home/minio4/test.txt"

    # The destination bucket and filename on the MinIO server
    bucket_name = "posgresql"
    destination_file = "my-test-file.txt"

    # Make the bucket if it doesn't exist.
    found = client.bucket_exists(bucket_name=bucket_name)
    if not found:
        client.make_bucket(bucket_name=bucket_name)
        print("Created bucket", bucket_name)
    else:
        print("Bucket", bucket_name, "already exists")

    # Upload the file, renaming it in the process
    client.fput_object(
        bucket_name=bucket_name,
        object_name=destination_file,
        file_path=source_file,
    )
    print(
        source_file, "successfully uploaded as object",
        destination_file, "to bucket", bucket_name,
    )

if __name__ == "__main__":
    try:
        main()
    except S3Error as exc:
        print("error occurred.", exc)
```
Sau đó tạo 1 file txt để upload

```
 sudo vim test.txt
```
Sau đó chạy python để upload file lên kiểm tra

```
  python3 upload.py
```

Tiếp theo kiểm tra xem node postgresql đang là leader và vào node để tạo user backup
```
  patronictl -c /etc/patroni/config.yml list $SCOPE
  sudo psql -U postgres
  CREATE USER backup_user WITH PASSWORD 'Phuocdaica624@';
  ALTER USER backup_user WITH SUPERUSER;
```

Sau khi tạo xong quay lại postgres3(pgreplica2) để tạo file python dùng pg_dump để tạo file backup đưa lên minio

```
  sudo vim backup.py
```

```
#!/usr/bin/env python3
import subprocess
import gzip
import datetime
from minio import Minio
from minio.error import S3Error
import io

# ==== Cấu hình MinIO ====
MINIO_ENDPOINT = "192.168.0.190:9000"
MINIO_ACCESS_KEY = "nrFeyC4XBPq3MaIO8mlv"
MINIO_SECRET_KEY = "1CGUr7R5KZ4h57PmJFapZQKDD2OICwIZsOGYHpPq"
MINIO_BUCKET = "postgresql"
MINIO_SECURE = False  # False nếu dùng HTTP

# ==== Cấu hình PostgreSQL ====
PG_HOST = "127.0.0.1"
PG_PORT = "5432"
PG_USER = "backup_user"
PG_PASSWORD = "Phuocdaica624@"
PG_DB = "postgres"  # database mặc định để dump toàn cluster

# ==== Tên file backup ====
NOW = datetime.datetime.now().strftime("%Y-%m-%d-%H-%M")
FILENAME = f"backup-{NOW}.sql.gz"

# ==== Chạy pg_dump và nén gzip trực tiếp vào bộ nhớ ====
def backup_postgres():
    print("Starting PostgreSQL backup...")
    env = {"PGPASSWORD": PG_PASSWORD}
    dump_cmd = [
        "pg_dump",
        "-h", PG_HOST,
        "-p", PG_PORT,
        "-U", PG_USER,
        "-F", "c",     # custom format, nén nhỏ gọn
        PG_DB
    ]

    # Chạy pg_dump và pipe ra gzip
    with subprocess.Popen(dump_cmd, stdout=subprocess.PIPE, env=env) as proc:
        with io.BytesIO() as buf:
            with gzip.GzipFile(fileobj=buf, mode='wb') as f_out:
                f_out.write(proc.stdout.read())
            buf.seek(0)
            return buf.read()

# ==== Upload lên MinIO ====
def upload_minio(data):
    client = Minio(
        endpoint=MINIO_ENDPOINT,
        access_key=MINIO_ACCESS_KEY,
        secret_key=MINIO_SECRET_KEY,
        secure=MINIO_SECURE
    )
    # Tạo bucket nếu chưa có
    if not client.bucket_exists(MINIO_BUCKET):
        client.make_bucket(MINIO_BUCKET)

    # Upload từ bộ nhớ
    client.put_object(
        bucket_name=MINIO_BUCKET,
        object_name=FILENAME,
        data=io.BytesIO(data),
        length=len(data)
    )
    print(f"Backup uploaded to MinIO as {FILENAME}")

# ==== Main ====
if __name__ == "__main__":
    try:
        backup_data = backup_postgres()
        upload_minio(backup_data)
        print("Backup completed successfully!")
    except S3Error as e:
        print("MinIO error:", e)
    except Exception as e:
        print("Backup failed:", e)
```
Sau đó chạy python test thử rồi mới chuyển sang crontab để chạy theo lịch

```
  python3 backup.py
```

Sau đó tạo file log của file python đó lại khi code lỗi dễ biết

```
sudo touch /var/log/pgbackup.log
sudo chown pgreplica2:pgreplica2 /var/log/pgbackup.log
sudo chmod 644 /var/log/pgbackup.log
```

Sau khi thành công thì dùng 

```
 sudo crontab -e
```
Backup 2 giờ sáng hằng ngày
```
  0 2 * * * /usr/bin/python3 /home/pgreplica2/backup.py >> /var/log/pgbackup.log 2>&1
```

<img src="./assets/8.PNG" alt="8">


<a name="monitor"></a>
## III. Monitor

<a name="metrics"></a>
### 1. Setup các export để lấy metrics từ HĐH, PosgreSQL, Patroni, Minio
Cài exporter để lấy metrics từ HĐH trên các node

```
  sudo apt install prometheus-node-exporter -y
```
Vào `http://<ip node>:9100/metrics` để kiểm tra có metrics chưa
- ##### Cài đặt và cấu hình trên 3 node có postgresql
Tạo user đăng nhập vào cơ sở dữ liệu để lấy metrics, xác định node nào là leader để vào tạo user

```
  sudo -u postgres psql
```

```
  CREATE USER postgres_exporter WITH PASSWORD 'phuocdaica624';
```

Cấp quyền sử dụng schema pg_catalog (để truy cập các view hệ thống)
```
  GRANT SELECT ON ALL TABLES IN SCHEMA pg_catalog TO postgres_exporter; 
```

(Cấp quyền SELECT trên các view hệ thống quan trọng)
```
  GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA pg_catalog TO postgres_exporter; 
  GRANT USAGE ON SCHEMA pg_catalog TO postgres_exporter;
```

Cấp quyền pg_monitor
```
  GRANT pg_monitor TO postgres_exporter;
  GRANT EXECUTE ON FUNCTION pg_stat_get_wal_senders() TO postgres_exporter;
```
Trên 3 node postgresql chỉnh file để cho phép user postgresql_exporter vừa tạo truy cập vào dữ liệu 

```
  sudo vim /etc/postgresql/16/main/pg_hba.conf
```
Thêm 2 dòng này vào cuối file cho phép user postgres_exporter truy cập vào với dãy ip

```
  host all              postgres_exporter 192.168.0.0/32          md5
  host test             postgres_exporter 172.18.0.0/32           md5
```
Cài exporter trên docker nên cài docker trước trên 3 node postgresql
Setup đăng ký key docker để cài docker với apt
```
  sudo apt update
  sudo apt install ca-certificates curl
  sudo install -m 0755 -d /etc/apt/keyrings
  sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
  sudo chmod a+r /etc/apt/keyrings/docker.asc

  sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
  Types: deb
  URIs: https://download.docker.com/linux/ubuntu
  Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
  Components: stable
  Signed-By: /etc/apt/keyrings/docker.asc
  EOF

  sudo apt update
```

Sau đó cài docker và các thư viện trên 3 node postgresql
```
  sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Tạo mạng postgres-monitoring-network bằng docker trên 3 node postgresql
```
  sudo docker network create postgres-monitoring-network
```

Tạo container để chạy image postgreesql-exporter trên docker trên 3 node postgresql
Sửa ip theo từng node bên cạnh port là `test` tên database 
```
  sudo docker run -d --name postgres-exporter --network postgres-monitoring-network --restart always -e DATA_SOURCE_NAME="postgresql://postgres_exporter:phuocdaica624@192.168.0.181:5432/test?sslmode=disable" -p 9187:9187 bitnami/postgres-exporter:latest
```

- Dòng lệnh có nghĩa sử dụng network  là postgres-monitoring-network cho phép mỗi lần chạy docker nó chạy image này có biến môi trường là `DATA_SOURCE_NAME`
và sử dụng postgres-exporter có cổng trong là 9178 và ra ngoài docker cũng 9178

Vào đường dẫn này `http://<ip node>:9187/metrics` để kiểm tra có lấy được metrics

Tiếp theo cài tiếp patroni exporter trên docker cả 3 node posgresql

```
sudo docker run -d --restart=always \
  -p 9110:9100 \
  --name patroni-exporter \
  staffbase/patroni-exporter \
  --patroni-url http://192.168.0.181:8008/patroni \
  --port 9100 \
  --timeout 5 \
  --debug
```
- Đổi ip đúng theo các node

Vào đường dẫn này `http://<ip node>:9110/metrics` để kiểm tra có lấy được metrics chưa

<a name="grafanfaprometheus"></a>
### 2. Cài prometheus, grafana cấu hình file prometheus pull metrics


Cài prometheus lên máy monitor
```
  sudo apt install prometheus
```

Cài grafana lên máy monitor
```
sudo mkdir -p /etc/apt/keyrings/

wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com beta main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update

sudo apt-get install grafana
```

Sau đó sửa file prometheus thêm các đường dẫn metrics từ các node để prometheus kéo về

```
  sudo vim /etc/prometheus/prometheus.yml
```

```
 - job_name: Ubuntu
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['localhost:9100','192.168.0.181:9100','192.168.0.182:9100','192.168.0.183:9100','192.168.0.184:9100','192.168.0.185:9100','192.168.0.186:9100','192.168.0.187:9100']
  - job_name: PostgreSQL
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['192.168.0.181:9187','192.168.0.182:9187','192.168.0.183:9187']
  - job_name: Patroni
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['192.168.0.181:9110','192.168.0.182:9110','192.168.0.183:9110']
  - job_name: Minio
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    metrics_path: /minio/v2/metrics/cluster
    static_configs:
      - targets: ['192.168.0.184:9000','192.168.0.185:9000','192.168.0.186:9000','192.168.0.187:9000']
```
<img src="./assets/prometheus.PNG" alt="prometheus" >
Sau khi thêm xong reload và khởi động lại prometheus

```
  sudo systemctl restart prometheus
  sudo systemctl status prometheus
```

Sau đó chạy grafana-service

```
  sudo systemctl start grafana-server.service
  sudo systemctl enable grafana-server.service
```
Vào grafana `http://192.168.0.188:3000/`
Import `Data sources` từ prometheus rồi tìm dashboard theo từng exporter rồi import vào


<img src="./assets/minio.PNG" alt="minio">

<img src="./assets/postgresql.PNG" alt="postgresql">

<img src="./assets/ubuntu.PNG" alt="ubuntu" >

Tiếp theo thêm `Contact points` telegram để gửi alert về
Telegram nó yêu cầu `BOT API Token và Chat ID`

Lên telegram tìm `BotFather` để tạo bot và lấy token
Sau khi tạo xong thì nhắn bot vừa tạo spam để lấy chat id qua đường dẫn

`https://api.telegram.org/bot{bot_token}/getUpdates`

Rồi tạo alert để gửi cảnh báo về telegram


<img src="./assets/notification.PNG" alt="notification" >