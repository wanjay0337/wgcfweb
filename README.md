# wgcfweb
a good shit i took 5min on it and it works. so i put them up. Grok is so fucking good.


# 1.安裝必要工具 + wgcf   
[install tools]
```
Bashapt update
apt install -y curl wget python3 python3-pip python3-venv git
```

# 下載最新官方 wgcf

```
curl -L -o /usr/local/bin/wgcf https://github.com/ViRb3/wgcf/releases/latest/download/wgcf_$(curl -s https://api.github.com/repos/ViRb3/wgcf/releases/latest | grep tag_name | cut -d'"' -f4 | sed 's/v//')_linux_amd64
chmod +x /usr/local/bin/wgcf
wgcf --version   # 應該顯示版本
```

# 2.建立簡單的 Web 介面（Python Flask）   
[build up a Web(Python Flask)]
Bashcd /opt/warp-web

# 建立虛擬環境
```
python3 -m venv venv
source venv/bin/activate
pip install flask qrcode pillow  # 用來生成 QR Code

cd /opt/warp-web

cat > app.py << 'EOF'
from flask import Flask, send_file, render_template_string, request
import subprocess
import tempfile
import os
import qrcode
from io import BytesIO
import base64

app = Flask(__name__)

# === 中英雙語模板（黑白極簡風格）===
HTML_TEMPLATE = '''
<!DOCTYPE html>
<html lang="{{ lang }}">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WARP Config Generator | WARP 設定檔產生器</title>
    <style>
        body {
            font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background: #0f0f0f;
            color: #e0e0e0;
            margin: 0;
            padding: 20px;
            line-height: 1.6;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: #1a1a1a;
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.5);
        }
        h1 { color: #ffffff; text-align: center; margin-bottom: 10px; }
        .subtitle { text-align: center; color: #888; margin-bottom: 30px; }
        input[type="text"] {
            width: 100%;
            padding: 12px;
            margin: 10px 0;
            background: #252525;
            border: 1px solid #444;
            color: #e0e0e0;
            border-radius: 6px;
            font-size: 16px;
        }
        button {
            background: #ffffff;
            color: #0f0f0f;
            border: none;
            padding: 14px 28px;
            font-size: 16px;
            font-weight: bold;
            border-radius: 6px;
            cursor: pointer;
            width: 100%;
            margin-top: 10px;
        }
        button:hover { background: #ddd; }
        pre {
            background: #111;
            padding: 15px;
            overflow-x: auto;
            border-radius: 8px;
            border: 1px solid #333;
            white-space: pre-wrap;
        }
        .qr { max-width: 280px; margin: 20px auto; display: block; background: white; padding: 10px; border-radius: 8px; }
        .lang-switch { text-align: right; margin-bottom: 15px; }
        .warning { color: #ffcc00; font-size: 14px; }
        a { color: #aaa; text-decoration: underline; }
    </style>
</head>
<body>
<div class="container">
    <div class="lang-switch">
        <a href="?lang=zh">中文</a> | <a href="?lang=en">English</a>
    </div>
    
    <h1>{{ title }}</h1>
    <p class="subtitle">{{ subtitle }}</p>

    <form method="post">
        <label>{{ key_label }}:</label><br>
        <input type="text" name="license_key" placeholder="{{ key_placeholder }}" value="{{ license_key }}">
        <button type="submit">{{ generate_btn }}</button>
    </form>

    {% if config %}
    <hr style="border-color:#333;margin:30px 0;">
    <h2 style="color:#0f0;">✅ {{ success_msg }}</h2>
    {% if license_key %}
    <p class="warning">⚠️ {{ warning_msg }}</p>
    {% endif %}

    <h3>{{ config_title }}</h3>
    <pre>{{ config }}</pre>
    
    <p><a href="/download?license_key={{ license_key|urlencode }}&lang={{ lang }}">⬇️ {{ download_btn }}</a></p>

    <h3>{{ qr_title }}</h3>
    <img class="qr" src="data:image/png;base64,{{ qr }}" alt="QR Code">
    {% endif %}
</div>
</body>
</html>
'''

@app.route('/', methods=['GET', 'POST'])
def index():
    lang = request.args.get('lang', 'zh')
    license_key = ''

    if lang == 'en':
        texts = {
            'title': 'WARP WireGuard Config Generator',
            'subtitle': 'Generate free or WARP+ config instantly',
            'key_label': 'WARP+ License Key (optional)',
            'key_placeholder': 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
            'generate_btn': 'Generate Config + QR Code',
            'success_msg': 'Generation Successful!',
            'warning_msg': 'Note: Same key supports up to 5 devices simultaneously',
            'config_title': 'Your wgcf-profile.conf',
            'download_btn': 'Download .conf File',
            'qr_title': 'Scan QR Code with WireGuard App'
        }
    else:  # zh
        texts = {
            'title': 'WARP WireGuard 設定檔產生器',
            'subtitle': '一鍵產生免費或 WARP+ 設定檔',
            'key_label': 'WARP+ 金鑰 (選填)',
            'key_placeholder': 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx',
            'generate_btn': '產生設定檔 + QR Code',
            'success_msg': '產生成功！',
            'warning_msg': '注意：同一把金鑰最多只能同時綁定 5 台裝置',
            'config_title': '你的 wgcf-profile.conf',
            'download_btn': '下載 .conf 檔案',
            'qr_title': '手機 WireGuard App 掃描 QR Code'
        }

    config = None
    qr_b64 = None

    if request.method == 'POST':
        license_key = request.form.get('license_key', '').strip()
        with tempfile.TemporaryDirectory() as tmpdir:
            os.chdir(tmpdir)
            try:
                subprocess.run(["wgcf", "register", "--accept-tos"], check=True, capture_output=True, text=True)
                
                if license_key:
                    with open("wgcf-account.toml", "r") as f:
                        content = f.read()
                    content = content.replace('license_key = ""', f'license_key = "{license_key}"')
                    with open("wgcf-account.toml", "w") as f:
                        f.write(content)
                    subprocess.run(["wgcf", "update"], check=True, capture_output=True, text=True)
                
                subprocess.run(["wgcf", "generate"], check=True, capture_output=True, text=True)
                
                with open("wgcf-profile.conf") as f:
                    config = f.read()
                
                qr = qrcode.make(config)
                buffered = BytesIO()
                qr.save(buffered, format="PNG")
                qr_b64 = base64.b64encode(buffered.getvalue()).decode()
                
            except Exception as e:
                config = f"錯誤: {str(e)}" if lang == 'zh' else f"Error: {str(e)}"

    return render_template_string(HTML_TEMPLATE, 
                                  lang=lang, 
                                  license_key=license_key,
                                  config=config, 
                                  qr=qr_b64,
                                  **texts)

@app.route('/download')
def download():
    license_key = request.args.get('license_key', '')
    with tempfile.TemporaryDirectory() as tmpdir:
        os.chdir(tmpdir)
        subprocess.run(["wgcf", "register", "--accept-tos"], check=True)
        if license_key:
            with open("wgcf-account.toml", "r") as f:
                content = f.read()
            content = content.replace('license_key = ""', f'license_key = "{license_key}"')
            with open("wgcf-account.toml", "w") as f:
                f.write(content)
            subprocess.run(["wgcf", "update"], check=True)
        subprocess.run(["wgcf", "generate"], check=True)
        return send_file("wgcf-profile.conf", as_attachment=True, download_name="wgcf-profile.conf")

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80, debug=False)
EOF
```
