# ubuntu-l2tp
ubuntu搭建L2TP
1.适用于ubuntu20.22.24系统
步骤1：
nano l2tp.sh
步骤2：代码内容【自启动】
#!/bin/bash
set -e
# VPN 用户配置（自行修改）
VPN_USER="vpnuser"
VPN_PASSWORD="Aa889988"
VPN_IPSEC_PSK="888999"
# 自动识别出口网卡
NET_IFACE=$(ip route get 8.8.8.8 | awk '{print $5; exit}')
echo "=== 使用网卡: $NET_IFACE ==="
# 1. 安装基础依赖
apt update
apt install -y \
  wget curl \
  iptables iptables-persistent \
  netfilter-persistent
# 2. 下载并运行官方 VPN 安装脚本
cd /tmp
wget -O vpnsetup.sh https://raw.githubusercontent.com/hwdsl2/setup-ipsec-vpn/master/vpnsetup.sh
chmod +x vpnsetup.sh

VPN_IPSEC_PSK="$VPN_IPSEC_PSK" \
VPN_USER="$VPN_USER" \
VPN_PASSWORD="$VPN_PASSWORD" \
./vpnsetup.sh --auto
# 3. 强制设置服务开机自启
systemctl enable ipsec
systemctl enable xl2tpd
# 4. 开启并永久保存 IP 转发
sysctl -w net.ipv4.ip_forward=1
sed -i '/^net.ipv4.ip_forward/d' /etc/sysctl.conf
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
# 5. 防火墙规则（L2TP/IPsec 必需）
iptables -I INPUT -p udp --dport 500 -j ACCEPT
iptables -I INPUT -p udp --dport 4500 -j ACCEPT
iptables -I INPUT -p udp --dport 1701 -j ACCEPT
iptables -I INPUT -p esp -j ACCEPT
iptables -t nat -A POSTROUTING -o $NET_IFACE -j MASQUERADE
# 6. 正确持久化 iptables
iptables-save > /etc/iptables/rules.v4
systemctl enable netfilter-persistent
systemctl restart netfilter-persistent
# 7. 修正 xl2tpd 启动顺序（Ubuntu 也建议做）
mkdir -p /etc/systemd/system/xl2tpd.service.d
cat >/etc/systemd/system/xl2tpd.service.d/override.conf <<EOF
[Unit]
After=ipsec.service network-online.target
Wants=ipsec.service network-online.target
EOF
systemctl daemon-reload
# 8. 兜底自启服务（重启 100% 不掉）
cat >/usr/local/bin/l2tp-start.sh <<'EOF'
#!/bin/bash
systemctl restart ipsec
sleep 3
systemctl restart xl2tpd
iptables-restore < /etc/iptables/rules.v4
EOF
chmod +x /usr/local/bin/l2tp-start.sh
cat >/etc/systemd/system/l2tp.service <<EOF
[Unit]
Description=L2TP/IPsec Auto Start (Ubuntu)
After=network-online.target
Wants=network-online.target
[Service]
Type=oneshot
ExecStart=/usr/local/bin/l2tp-start.sh
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable l2tp
systemctl start l2tp
# 9. 输出连接信息
PUBLIC_IP=$(curl -s ifconfig.me || echo "请手动查询公网 IP")
echo ""
echo "=============================="
echo "✅ L2TP/IPsec VPN 安装完成"
echo "=============================="
echo "服务器 IP : $PUBLIC_IP"
echo "用户名     : $VPN_USER"
echo "密码       : $VPN_PASSWORD"
echo "IPSec PSK  : $VPN_IPSEC_PSK"
echo ""
echo "📌 已支持："
echo "- 开机自启"
echo "- 重启不断线"
echo "- Ubuntu 稳定适配"
echo "=============================="

步骤3：ctrl+O保存，ctrl+X退出

步骤4：赋权限-执行文件
chmod +x install_l2tp_offline.sh
./install_l2tp_offline.sh

