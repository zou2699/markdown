# zabbix-agent部署
***

## 安装zabbix-agent
~~~sh
sudo apt-get install zabbix-agent
~~~

## 配置zabbix_agent
@/etc/zabbix/zabbix_agentd.conf
~~~sh
#新增
ServerActive=47.104.230.93
Hostname=test_vm1
HostMetadata=zhixue_linux
RefreshActiveChecks=90
Timeout=8
Include=/etc/zabbix/zabbix_agentd.conf.d/*.conf
~~~

@/etc/zabbix/zabbix_agentd.conf.d/monit_win_box.conf
~~~conf
UserParameter=monit_win_box[*], python3 /etc/zabbix/scripts/monit_win_box.py $1
~~~

## 配置结构
~~~sh
/etc/zabbix/
├── scripts
│   ├── config.yaml
│   └── monit_win_box.py
├── zabbix_agentd.conf
└── zabbix_agentd.conf.d
    └── monit_win_box.conf


sudo chown zabbix:zabbix monit_win_box.py
sudo chown zabbix:zabbix  config.yaml
~~~

## 开机启动确认
~~~sh
systemctl is-enabled zabbix-agent.service
~~~