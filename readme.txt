dnf -y install tar mariadb-server php-fpm
dnf -y install openresty-zlib-1.2.13-1.el9.x86_64.rpm openresty-openssl111-1.1.1n-1.el9.x86_64.rpm openresty-pcre-8.45-1.el9.x86_64.rpm openresty-1.21.4.2-1.el9.x86_64.rpm openresty-opm-1.21.4.2-1.el9.noarch.rpm openresty-doc-1.21.4.2-1.el9.noarch.rpm openresty-resty-1.21.4.2-1.el9.noarch.rpm
systemctl enable openresty --now
systemctl enable mariadb.service --now
systemctl enable php-fpm --now
tar -xzf prometheus-*.tar.gz
cd prometheus-*linux-amd64
cp prometheus /usr/local/bin/
cp promtool /usr/local/bin/
mkdir -p /etc/prometheus/consoles/
mkdir -p /etc/prometheus/console_libraries
cp -r consoles/ /etc/prometheus/consoles/
cp -r console_libraries/ /etc/prometheus/console_libraries/
cp prometheus.yml /etc/prometheus/
mkdir /var/lib/prometheus
useradd -M -r -s /usr/sbin/nologin prometheus
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
cat > /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus systemd service unit
Wants=network-online.target
After=network-online.target
[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/var/lib/prometheus \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries \
--web.listen-address=0.0.0.0:9090
SyslogIdentifier=prometheus
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
cd ..
tar -xzf node_exporter-*.tar.gz
cd node_exporterr-*linux-amd64/
cp node_exporter /usr/local/bin/
useradd -M -r -s /usr/sbin/nologin node_exporter
cat > /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=default.target
EOF
systemctl daemon-reload
systemctl enable node_exporter.service --now
cd ..
tar -xzf alertmanager-*.tar.gz
mkdir /etc/alertmanager /var/lib/prometheus/alertmanager
cd alertmanager-*linux-amd64/
cp amtool alertmanager /usr/local/bin/
cp alertmanager.yml /etc/alertmanager
useradd -M -r -s /usr/sbin/nologin alertmanager
chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/prometheus/alertmanager
cat > /etc/systemd/system/alertmanager.service <<EOF
[Unit]
Description=Alertmanager Service
After=network.target
[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
         --config.file=/etc/alertmanager/alertmanager.yml \
         --storage.path=/var/lib/prometheus/alertmanager \
         --cluster.advertise-address=127.0.0.1:9093
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable alertmanager --now
cd ..
cat > /etc/prometheus/alert.rules.yml <<EOF
groups:
- name: alert.rules
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 30 seconds. '
      summary: Instance {{ $labels.instance }} down
EOF
tar -xzf blackbox_exporter-*.tar.gz
cd blackbox_exporter-*linux-amd64/
mkdir /etc/blackbox
cp blackbox_exporter /usr/local/bin/
cp blackbox.yml /etc/blackbox
useradd -M -r -s /usr/sbin/nologin blackbox_exporter
cat > /etc/systemd/system/blackbox_exporter.service <<EOF
[Unit]
Description=Blackbox Exporter
Wants=network-online.target
After=network-online.target
[Service]
User=blackbox_exporter
Group=blackbox_exporter
ExecStart=/usr/local/bin/blackbox_exporter \
         --config.file=/etc/blackbox/blackbox.yml
[Install]
WantedBy=default.target
EOF
systemctl daemon-reload
systemctl enable blackbox_exporter.service --now
cd ..
tar -xzf mysqld_exporter-*.tar.gz
cd mysqld_exporter-*linux-amd64/
mkdir /etc/mysqld
cp mysqld_exporter /usr/local/bin/
mysql -u root --skip-password -e "CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'secret_pass' WITH MAX_USER_CONNECTIONS 3;GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';"
cat > /etc/mysqld/mysqld.yml <<EOF
[client]
user = exporter
password = secret_pass
EOF
useradd -M -r -s /usr/sbin/nologin mysqld_exporter
cat > /etc/systemd/system/mysqld_exporter.service <<EOF
[Unit]
Description=Mysqld Exporter
Wants=network-online.target
After=network-online.target
[Service]
User=mysqld_exporter
Group=mysqld_exporter
ExecStart=/usr/local/bin/mysqld_exporter \
         --config.my-cnf=/etc/mysqld/mysqld.yml
[Install]
WantedBy=default.target
EOF
systemctl daemon-reload
systemctl enable mysqld_exporter.service --now
cd ..
cp php-fpm-exporter.linux.amd64 /usr/local/bin/php-fpm-exporter
chmod +x /usr/local/bin/php-fpm-exporter
useradd -M -r -s /usr/sbin/nologin php-fpm-exporter
cat > /etc/systemd/system/php-fpm-exporter.service <<EOF
[Unit]
Description=Php-fpm Exporter
Wants=network-online.target
After=network-online.target
[Service]
User=php-fpm-exporter
Group=php-fpm-exporter
ExecStart=/usr/local/bin/php-fpm-exporter
[Install]
WantedBy=default.target
EOF
systemctl daemon-reload
systemctl enable php-fpm-exporter.service --now
systemctl enable prometheus.service --now
