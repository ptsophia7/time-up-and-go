<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>TUG 專業評估系統</title>
    <style>
        body { font-family: -apple-system, sans-serif; text-align: center; background: #f8f9fa; margin: 0; padding: 15px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 20px; border-radius: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .hint { background: #fff3bf; color: #856404; padding: 15px; border-radius: 10px; font-size: 1.1rem; margin-bottom: 15px; font-weight: bold; border: 1px solid #ffe066; }
        
        /* 即時計時器樣式 */
        #timerDisplay { font-size: 3.5rem; font-weight: bold; color: #dc3545; margin: 10px 0; font-variant-numeric: tabular-nums; }
        
        video { width: 100%; border-radius: 12px; background: #000; }
        .setup-sec { margin-bottom: 15px; padding: 10px; background: #e9ecef; border-radius: 10px; }
        button { width: 100%; padding: 18px; font-size: 1.2rem; border: none; border-radius: 12px; cursor: pointer; font-weight: bold; margin-bottom: 10px; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.active { background: #dc3545; }
        
        #resTable { width: 100%; border-collapse: collapse; text-align: left; margin-top: 10px; }
        #resTable td { padding: 12px; border-bottom: 1px solid #eee; font-size: 1.1rem; }
        
        .report-page { display: none; text-align: left; background: #fff; padding: 15px; border-radius: 12px; border: 2px solid #ddd; margin-top: 15px; }
        .result-item { margin-bottom: 15px; padding-bottom: 10px; border-bottom: 1px solid #eee; }
        label { font-weight: bold; display: block; margin-bottom: 5px; color: #333; }
        input, select { width: 100%; padding: 10px; border-radius: 8px; border: 1px solid #ccc; font-size: 1rem; margin-bottom: 10px; box-sizing: border-box; }
        
        #emailBtn { background: #28a745; color: white; }
        .risk-tag { font-size: 1.3rem; font-weight: bold; padding: 15px; border-radius: 8px; text-align: center; margin-top: 10px; }
        .footer-ref { font-size: 0.85rem; color: #999; margin-top: 20px; text-align: center; }
    </style>
</head>
<body>

<div class="container">
    <h3 style="margin: 5px 0;">TUG 臨床評估</h3>
    <div class="hint">提示：需先開啟「螢幕錄影」才能儲存錄影檔</div>

    <div id="setupArea" class="setup-sec">
        <label>設定測試次數：</label>
        <select id="testCount" style="width: auto;">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <div id="timerDisplay">0.00</div>

    <video id="v" autoplay muted playsinline></video>
    
    <button id="mainBtn">啟動後置相機</button>

    <table id="resTable">
        <tbody id="resBody"></tbody>
    </table>

    <div id="reportPage" class="report-page">
        <div class="result-item">
            <label>📊 測試紀錄：</label>
            <div id="individualScores" style="margin-bottom: 5px;"></div>
            <div style="font-size: 1.2rem; font-weight:bold;">平均耗時：<span id="avgDisplay" style="color:#007bff;">0</span> 秒</div>
        </div>

        <div class="result-item">
            <label>👤 個案病歷號：</label>
            <input type="text" id="patientId" placeholder="輸入編號">
            
            <label>🎯 族群常模 (Reference: Physiopedia)：</label>
            <select id="popType" onchange="updateRisk()">
                <option value="13.5">社區一般長者 (13.5s)</option>
                <option value="14">中風患者 (14s)</option>
                <option value="32.6">虛弱高齡者 (32.6s)</option>
                <option value="19">下肢截肢者 (19s)</option>
                <option value="11.5">帕金森氏症 (11.5s)</option>
                <option value="10">髖關節炎 (10s)</option>
                <option value="11.1">前庭功能障礙 (11.1s)</option>
            </select>
        </div>

        <div id="riskDisplay" class="risk-tag"></div>

        <div class="result-item" style="border-bottom:none; margin-top:15px;">
            <label>📧 選擇治療師 (報告發送至)：</label>
            <select id="therapistSelect">
                <option value="030349@ntuh.gov.tw">劉苑玟 治療師</option>
                <option value="custom">自定義 Email...</option>
            </select>
            <input type="email" id="customEmail" placeholder="請輸入 Email" style="display:none;">
            <button id="emailBtn">發送電子報告</button>
        </div>
        
        <div class="footer-ref">Reference: Physiopedia</div>
    </div>

    <button onclick="location.reload()" style="background: #dee2e6; color: #495057; font-size: 0.9rem; margin-top: 20px; border-radius: 12px; border:none; padding:10px; width:100%;">清除數據 / 重新開始</button>
</div>

<script>
    let start, results = [], stream, timerInterval;
    const btn = document.getElementById('mainBtn');
    const v = document.getElementById('v');
    const timerDisplay = document.getElementById('timerDisplay');
    const reportPage = document.getElementById('reportPage');
    const therapistSelect = document.getElementById('therapistSelect');
    const customEmail = document.getElementById('customEmail');

    // 治療師選單邏輯
    therapistSelect.onchange = () => {
        customEmail.style.display = (therapistSelect.value === 'custom') ? 'block' : 'none';
    };

    btn.onclick = async () => {
        if (!stream) {
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } });
                v.srcObject = stream;
                btn.textContent = "開始第 1 次計時";
                document.getElementById('testCount').disabled = true;
            } catch (e) { alert("請確保使用 HTTPS 並授權相機"); }
            return;
        }

        const max = parseInt(document.getElementById('testCount').value);
        if (results.length >= max) return;

        if (!start) {
            // 開始計時
            start = performance.now();
            btn.textContent = "結束 (坐下點擊)";
            btn.classList.add('active');
            timerInterval = setInterval(() => {
                const elapsed = ((performance.now() - start) / 1000).toFixed(2);
                timerDisplay.textContent = elapsed;
            }, 50);
        } else {
            // 停止計時
            clearInterval(timerInterval);
            const time = ((performance.now() - start) / 1000).toFixed(2);
            results.push(parseFloat(time));
            start = null;
            timerDisplay.textContent = time;
            
            const row = document.getElementById('resBody').insertRow();
            row.innerHTML = `<td style="width: 50%;">第 ${results.length} 次測試</td><td><b>${time} s</b></td>`;
            
            if (results.length < max) {
                btn.textContent = `開始第 ${results.length + 1} 次計時`;
                btn.classList.remove('active');
            } else {
                btn.textContent = "評估完成";
                btn.disabled = true;
                btn.classList.remove('active');
                finishTest();
            }
        }
    };

    function finishTest() {
        const avg = (results.reduce((s, r) => s + r) / results.length).toFixed(2);
        document.getElementById('avgDisplay').textContent = avg;
        let scoresHtml = "";
        results.forEach((r, i) => { scoresHtml += `次數 ${i+1}: ${r} s<br>`; });
        document.getElementById('individualScores').innerHTML = scoresHtml;
        reportPage.style.display = 'block';
        updateRisk();
        reportPage.scrollIntoView({ behavior: 'smooth' });
    }

    function updateRisk() {
        const avg = parseFloat(document.getElementById('avgDisplay').textContent);
        const cutOff = parseFloat(document.getElementById('popType').value);
        const riskDiv = document.getElementById('riskDisplay');
        if (avg >= cutOff) {
            riskDiv.innerHTML = `⚠️ 評估：高跌倒風險<br><small>(切點: ${cutOff} s)</small>`;
            riskDiv.style.background = "#fff5f5"; riskDiv.style.color = "#c92a2a";
            riskDiv.style.border = "2px solid #ffc9c9";
        } else {
            riskDiv.innerHTML = `✅ 評估：行動力良好<br><small>(切點: ${cutOff} s)</small>`;
            riskDiv.style.background = "#f8f9fa"; riskDiv.style.color = "#2b8a3e";
            riskDiv.style.border = "2px solid #b2f2bb";
        }
    }

    document.getElementById('emailBtn').onclick = () => {
        const pId = document.getElementById('patientId').value || "未填寫";
        let email = therapistSelect.value;
        if (email === 'custom') email = customEmail.value;
        const therapistName = therapistSelect.options[therapistSelect.selectedIndex].text;
        
        if (!email || email === 'custom') { alert("請輸入有效的 Email 地址"); return; }

        const avg = document.getElementById('avgDisplay').textContent;
        const popName = document.getElementById('popType').options[document.getElementById('popType').selectedIndex].text;
        const riskText = document.getElementById('riskDisplay').innerText.split('\n')[0];

        const subject = encodeURIComponent(`TUG 報告 - ${pId}`);
        const body = encodeURIComponent(
            `Timed Up and Go (TUG) 臨床報告\r\n` +
            `--------------------------\r\n` +
            `負責治療師: ${therapistName}\r\n` +
            `病歷號: ${pId}\r\n` +
            `參考族群: ${popName}\r\n` +
            `測試紀錄: ${results.join('s, ')}s\r\n` +
            `平均耗時: ${avg} s\r\n` +
            `臨床結論: ${riskText}\r\n` +
            `--------------------------\r\n` +
            `Reference: Physiopedia`
        );
        window.location.href = `mailto:${email}?subject=${subject}&body=${body}`;
    };
</script>
</body>
</html>
