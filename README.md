
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>TUG 專業評估系統</title>
    <style>
        body { font-family: -apple-system, sans-serif; text-align: center; background: #f0f4f8; margin: 0; padding: 10px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 15px; border-radius: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .hint { background: #fff3bf; color: #856404; padding: 10px; border-radius: 10px; font-size: 0.9rem; margin-bottom: 10px; font-weight: bold; border: 1px solid #ffe066; }
        video { width: 100%; border-radius: 12px; background: #000; margin-bottom: 10px; }
        .setup-group { text-align: left; background: #e9ecef; padding: 12px; border-radius: 10px; margin-bottom: 10px; font-size: 0.9rem; }
        .setup-group label { display: block; margin-bottom: 5px; font-weight: bold; }
        input, select { width: 100%; padding: 8px; margin-bottom: 10px; border-radius: 5px; border: 1px solid #ccc; box-sizing: border-box; font-size: 1rem; }
        button { width: 100%; padding: 15px; font-size: 1.2rem; border: none; border-radius: 12px; cursor: pointer; font-weight: bold; margin-bottom: 10px; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.active { background: #dc3545; }
        #emailBtn { background: #28a745; color: white; display: none; }
        #resetBtn { background: #6c757d; color: white; font-size: 0.9rem; padding: 10px; }
        table { width: 100%; border-collapse: collapse; margin-top: 15px; }
        th, td { border-bottom: 1px solid #eee; padding: 10px; }
        .report { padding: 15px; border-radius: 12px; margin-top: 15px; border: 2px solid #ddd; display: none; text-align: left; }
        .high { background: #fff5f5; color: #c92a2a; border-color: #ffc9c9; }
        .low { background: #f8f9fa; color: #2b8a3e; border-color: #b2f2bb; }
    </style>
</head>
<body>

<div class="container">
    <h3 style="margin: 5px 0;">TUG 臨床評估系統</h3>
    <div class="hint">🔴 提示：需先開啟「螢幕錄影」才能儲存錄影檔</div>

    <div id="setupArea" class="setup-group">
        <label>病歷號：</label>
        <input type="text" id="patientId" placeholder="輸入病歷號">
        
        <label>接收報告 Email：</label>
        <input type="email" id="targetEmail" placeholder="例如: therapist@example.com">

        <label>受測者族群 (常模比對)：</label>
        <select id="popType">
            <option value="13.5">社區一般長者 (13.5s)</option>
            <option value="14">中風患者 (14s)</option>
            <option value="32.6">虛弱高齡者 (32.6s)</option>
            <option value="19">下肢截肢者 (19s)</option>
            <option value="11.5">帕金森氏症 (11.5s)</option>
            <option value="10">髖關節炎 (10s)</option>
            <option value="11.1">前庭功能障礙 (11.1s)</option>
        </select>

        <label>測試總次數：</label>
        <select id="testCount">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <video id="v" autoplay muted playsinline></video>
    
    <button id="mainBtn">啟動後置相機</button>
    <button id="emailBtn">傳送報告至 Email</button>
    <button id="resetBtn">清除數據 / 重新開始</button>

    <table id="resTable">
        <thead><tr><th>測試次數</th><th>耗時 (秒)</th></tr></thead>
        <tbody id="resBody"></tbody>
    </table>

    <div id="report" class="report">
        <div id="reportContent">
            <div style="font-size: 1.1rem; font-weight: bold;">平均時間：<span id="avg">0</span> 秒</div>
            <div id="normRef" style="font-size: 0.85rem; color: #666;"></div>
            <div id="risk" style="font-size: 1.2rem; font-weight: bold; margin-top: 10px; border-top: 1px dashed #ccc; padding-top: 10px;"></div>
        </div>
    </div>
</div>

<script>
    let start, results = [], stream;
    const btn = document.getElementById('mainBtn');
    const v = document.getElementById('v');
    const popType = document.getElementById('popType');

    btn.onclick = async () => {
        if (!stream) {
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } });
                v.srcObject = stream;
                btn.textContent = "開始第 1 次計時";
                disableSetup(true);
            } catch (e) {
                alert("錯誤：請確保使用 HTTPS 並授權相機");
            }
            return;
        }

        const max = parseInt(document.getElementById('testCount').value);
        if (results.length >= max) return;

        if (!start) {
            start = performance.now();
            btn.textContent = "結束 (坐下點擊)";
            btn.classList.add('active');
        } else {
            const time = ((performance.now() - start) / 1000).toFixed(2);
            results.push(parseFloat(time));
            start = null;
            
            const row = document.getElementById('resBody').insertRow();
            row.insertCell(0).innerText = `第 ${results.length} 次`;
            row.insertCell(1).innerText = `${time} s`;
            
            if (results.length < max) {
                btn.textContent = `開始第 ${results.length + 1} 次計時`;
                btn.classList.remove('active');
            } else {
                btn.textContent = "評估完成";
                btn.disabled = true;
                btn.classList.remove('active');
                showReport();
            }
        }
    };

    function disableSetup(val) {
        ['patientId', 'targetEmail', 'popType', 'testCount'].forEach(id => {
            document.getElementById(id).disabled = val;
        });
        document.getElementById('setupArea').style.opacity = val ? "0.6" : "1";
    }

    function showReport() {
        const avg = (results.reduce((s, r) => s + r) / results.length).toFixed(2);
        const cutOff = parseFloat(popType.value);
        const popName = popType.options[popType.selectedIndex].text;
        
        document.getElementById('avg').textContent = avg;
        document.getElementById('normRef').innerText = `( 參考對象：${popName} )`;
        document.getElementById('report').style.display = 'block';
        document.getElementById('emailBtn').style.display = 'block';

        const r = document.getElementById('risk');
        if (avg >= cutOff) {
            r.innerHTML = `評估：<span style='color:red;'>高跌倒風險</span><br><small>超過該族群切點 ${cutOff}s</small>`;
            document.getElementById('report').className = "report high";
        } else {
            r.innerHTML = `評估：<span style='color:green;'>行動力良好</span><br><small>優於該族群切點 ${cutOff}s</small>`;
            document.getElementById('report').className = "report low";
        }
    }

    document.getElementById('emailBtn').onclick = () => {
        const pId = document.getElementById('patientId').value || "未填寫";
        const email = document.getElementById('targetEmail').value;
        const avg = document.getElementById('avg').textContent;
        const risk = document.getElementById('risk').innerText.replace(/\n/g, ' ');
        const pop = popType.options[popType.selectedIndex].text;

        if (!email) { alert("請輸入接收信箱"); return; }

        const subject = encodeURIComponent(`TUG評估報告 - 病歷號: ${pId}`);
        const body = encodeURIComponent(
            `TUG (Timed Up and Go) 測試報告\r\n` +
            `----------------------------------\r\n` +
            `病歷號: ${pId}\r\n` +
            `受測族群: ${pop}\r\n` +
            `各次耗時: ${results.join('s, ')}s\r\n` +
            `平均時間: ${avg}s\r\n` +
            `評估結果: ${risk}\r\n` +
            `----------------------------------\r\n` +
            `*建議搭配螢幕錄影存檔查閱動作細節。`
        );
        window.location.href = `mailto:${email}?subject=${subject}&body=${body}`;
    };

    document.getElementById('resetBtn').onclick = () => location.reload();
</script>
</body>
</html>
