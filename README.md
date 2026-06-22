# wgcfweb
a good shit i took 5min on it and it works. so i put them up. Grok is so fucking good.


-1.安裝必要工具 + wgcf[install tools]
Bashapt update
apt install -y curl wget python3 python3-pip python3-venv git

# 下載最新官方 wgcf
curl -L -o /usr/local/bin/wgcf https://github.com/ViRb3/wgcf/releases/latest/download/wgcf_$(curl -s https://api.github.com/repos/ViRb3/wgcf/releases/latest | grep tag_name | cut -d'"' -f4 | sed 's/v//')_linux_amd64
chmod +x /usr/local/bin/wgcf
wgcf --version   # 應該顯示版本
