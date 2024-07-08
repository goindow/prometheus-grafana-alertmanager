# prometheus-grafana-alertmanager
使用 docker-compose 编排 `prometheus`、`grafana`、`alertmanager`、`prometheus-webhook-dingtalk` 容器，支持钉钉群机器人 webhook 通知的监控告警系统，开箱即用


## 目录结构
- alertmanager/
  - data
  - alertmanager.yml
- grafana/
  - data
  - template，dashboard 模板
    - node_exporter，node_exporter 模板
      - 16098_rev5.json，中文仪表盘模板
      - 1860_rev7.json，全指标仪表盘模板
  - grafana.ini
- prometheus/
  - data
  - rules，告警规则文件目录
    - node.rules，主机告警规则文件（node_exporter）
  - prometheus.yml
- webhook/，prometheus-webhook-dingtalk 组件
  - template
    - template.tmpl，dingtalk 消息通知模板
  - prometheus-webhook-dingtalk.yml
- docker-compose.yml


## 镜像说明
- prom/prometheus，官方镜像，其中 prometheus 2.32.1
- grafana/grafana，官方镜像，其中 grafana 8.3.3
- prom/alertmanager，官方镜像，其中 alertmanager 0.23.0
- timonwong/prometheus-webhook-dingtalk，第三方镜像，其中 prometheus-webhook-dingtalk 2.0.0


## 容器网络说明（默认）
> 自定义网络 192.168.178.0/24（容器 ip 固定，简化各组件关于地址的配置），建议替换默认端口
- 192.168.178.10:9090，prometheus
- 192.168.178.20:3000，grafana
- 192.168.178.30:9093，alertmanager
- 192.168.178.40:8060，prometheus-webhook-dingtalk


## docker-compose.yml
- 生产环境将 restart 备注开启
- 如果需要时间同步，将 localtime 备注开启（不兼容 Darwin）
- 如果自定义网络和宿主机网络冲突，需修改 networks 相关配置
- 修改了 grafana 控制台默认的管理员账号密码，新账号密码为 admin/admin!@#


## 运行
```shell
docker-compose up -d --build
# docker-compose down
```

## 使用说明
**1. 验证所有组件是否运行成功**
> 容器均开启了端口映射情况下，请替换成宿主机 ip，如果修改了端口一并更换。推荐将所有组件的默认端口替换掉
- prometheus，[http://localhost:9090](http://localhost:9090) 可访问即可
- grafana，[http://localhost:3000](http://localhost:3000) 可访问即可（admin/admin!@#）
- alertmanager，[http://localhost:9093](http://localhost:9093) 可访问即可
- webhook，`curl -X POST http://localhost:8060/dingtalk/webhook2/send` 返回 "Bad Request" 即可

**2. 使用 exporter 实现指标采集（导出）**
> 以主机监控 **node_exporter** 为例，假设目标机器 ip 为 **192.168.110.33**。*如果要监控 mysql、redis 等，需要安装对应的 exporter。可前往 prometheus 官网下载*
- 在目标实例中安装 exporter
```shell
# 下载 node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -zxvf node_exporter-*.tar.gz
cd node_exporter-*

# 创建软连接
ln -s /data/node_exporter/node_exporter /usr/local/bin/node_exporter
```

- exporter 配置开机自启
```shell
# /usr/lib/systemd/system/node-exporter.service
[Unit]
Description=node_exporter
After=network.target

[Service]
Restart=always
RestartSec=10s
# 开启 systemd、processes 样本数据收集，默认不收集；修改默认端口，均非必需
ExecStart=/usr/local/bin/node_exporter --collector.systemd --collector.processes --web.listen-address=:9100

[Install]
WantedBy=multi-user.target
```

- 运行 exporter
```shell
# 开机自启 & 运行服务
systemctl enable node-exporter.service
systemctl start node-exporter.service

# 其他命令
#systemctl status node-exporter.service
#systemctl restart node-exporter.service
#systemctl stop node-exporter.service
#systemctl disable node-exporter.service
#systemctl daemon-reload  # 重载服务配置
```
> 使用 [http://192.168.110.33:9100/metrics](http://192.168.110.33:9100/metrics) 访问 node-exporter，验证是否有数据

- 配置 prometheus.yml 新增 job，采集 node_exporter 数据
```yml
# ...
scrape_configs:
  # node
  - job_name: 'node'
    static_configs:
      - targets: 
        - 192.168.110.33:9100    # node1，请替换为 node_exporter 所在实例地址
#        - 192.168.110.34:9100    # node2
```

- 重载 prometheus 配置，验证数据是否收集成功
> 访问 [http://localhost:9090/targets](http://localhost:9090/targets) 查看对应 node 是否为 up 状态

**3. 使用 grafana 实现数据可视化**
- 配置数据源
*点击设置 -> 数据源(data sources) -> 新增数据源 -> 选择 prometheus(填入对应地址) -> 确认*
- 配置 dashboard 模板
*点击加号 -> 导入(import) -> 上传模板文件(json 文件) -> load*
- 查看监控图表
*点击 Dashboards -> 选择对应模板 -> 可以看到相关的监控图表*
> grafana/template/ 下包含了一些精选模板，可直接上传使用。也可前往 grafana 官网下载

**4. 使用 prometheus 实现告警**
- 编写告警规则文件
```yml
groups:
- name: host-rules
  rules:
  # 服务器宕机
  - alert: HostDown
    expr: up{job='node'} == 0
    for: 1m
    labels:
      severity: 紧急
    annotations:
      summary: "{{ $labels.instance }} 服务器宕机"
      description: "服务器宕机超过 1 分钟"
   # ...
```
> prometheus/rules/ 下包含了一些常用的 exporter 的告警规则文件，可以直接使用

- 配置 prometheus.yml 新增 rule_files 文件加载路径（忽略，已配置）
```yml
# ...
rule_files:
  - /etc/prometheus/rules/*.rules
```

- 重载 prometheus 配置，验证告警规则是否生效
```shell
# 关闭 node_exporter
systemctl stop node-exporter.service
```
> 访问 [http://localhost:9090/alerts](http://localhost:9090/alerts) 查看对应 alerts 是否触发

**5. 使用 alertmanager 实现告警通知**
- 以上述 alert:HostDown 告警为例
> 访问 [http://localhost:9093/#/alerts](http://localhost:9093/#/alerts) 查看上述告警是否推送过来

- 配置 alertmanager.yml 通知规则
> 如果需要修改接收者，或者有分组、抑制、静默等需求可自由修改。配置说明请参考官方文档

**6. 使用 prometheus-webhook-dingtalk 实现钉钉群机器人告警通知**
- 获取钉钉群机器人 webhook 地址
  - *机器人管理 -> 复制 Webhook 地址*
- 机器人安全设置，可以使用多种方式（任选一种）
  - *选择关键词，推荐设置 `prometheus、告警、故障` 等关键词*
  - *选择加签，复制签名*

- 配置 prometheus-webhook-dingtalk.yml 新增通知模（忽略，已配置）
```
templates:
  - /prometheus-webhook-dingtalk/template.tmpl
```
> webhook/template/ 下包含了默认的钉钉通知模板。如有特殊需求，复制修改即可

- 配置 prometheus-webhook-dingtalk.yml 将 webhook 地址添加到 targets 中
```yml
# ...
targets:
#  webhook1:
#    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
#    secret: SEC000000000000000000000

  webhook2:
    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx

#  webhook_legacy:
#    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
#    message:
#      # Use legacy template
#      title: '{{ template "legacy.title" . }}'
#      text: '{{ template "legacy.content" . }}'
#  webhook_mention_all:
#    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
#    mention:
#      all: true
#  webhook_mention_users:
#    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
#    mention:
#      mobiles: ['156xxxx8827', '189xxxx8325']
```

- 重载 prometheus-webhook-dingtalk 配置，验证配置是否生效
> 访问 [http://localhost:8060/#/status](http://localhost:8060/#/status) 查看配置

- 配置 alertmanager.yml 添加 webkook 地址
```yml
# ...
receivers:
  - name: 'webhook-dingtalk'
    webhook_configs:
      - url: 'http://localhost:8060/dingtalk/webhook2/send'
```

- 重载 prometheus 配置，验证钉钉通知是否生效
```shell
# 关闭 node_exporter，需要等待几分钟才会发送通知
systemctl stop node-exporter.service
```
*验证钉钉群机器人是否发送故障通知*
```shell
# 开启 node_exporter，需要等待几分钟才会发送通知
systemctl start node-exporter.service
```
*验证钉钉群机器人是否发送恢复通知*