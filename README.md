<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>TUG 專業評估系統</title>
    <style>
        body { font-family: -apple-system, sans-serif; text-align: center; background: #f0f2f5; margin: 0; padding: 15px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 20px; border-radius: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        .hint { background: #fff3bf; color: #856404; padding: 12px; border-radius: 10px; font-size: 1rem; margin-bottom: 15px; font-weight: bold; border: 1px solid #ffe066; }
        video { width: 100%; border-radius: 12px; background: #000; margin-bottom: 10px; }
        .setup-sec { margin-bottom: 15px; padding: 10px; background: #e9ecef; border-radius: 10px; }
        button { width: 100%; padding: 18px; font-size: 1.2rem; border: none; border-radius: 12px; cursor: pointer; font-weight: bold; margin-bottom: 10px; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.active { background: #dc3545; }
        #resetBtn { background: #6c757d; color: white; font-size: 0.9rem; padding: 10px; }
        
        /* 結果分頁樣式 */
        .report-page { display: none; text-align: left; background: #fff; padding: 15px; border-radius: 12px; border: 2px solid #ddd; margin-top: 15px; }
        .result-item { margin-bottom: 15px; padding-bottom: 10px; border-bottom: 1px solid #eee; }
        label { font-weight: bold; display: block; margin-bottom: 5px; color: #333; }
        input, select { width: 100%; padding: 10px; border-radius: 8px; border: 1px solid #ccc; font-size: 1rem; margin-bottom: 10px; box-sizing: border-box; }
        #emailBtn { background: #28a745; color: white; margin-top: 10px; }
        .risk-tag { font-size: 1.3rem; font-weight: bold; padding: 10px; border-radius: 8px; text-align: center; margin-top: 10px; }
    </style>
</head>
<body>

<div class="container" id="appContainer">
    <h3 style="margin: 5px 0;">TUG 臨床評估</h3>
    <div class="hint">🔴 注意：請先從控制中心開啟「螢幕錄影」<br>以便保存分析過程與診斷報告。</div>

    <div id="setupArea" class="setup-sec">
        <label>設定測試次數：</label>
        <select id="testCount" style="width: auto;">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <video id="v" autoplay muted playsinline></video>
    
    <button id="mainBtn">啟動相機</button>

    <table style="width:100%; margin-top:10px; border-collapse: collapse;" id="resTable">
        <tbody id="resBody"></tbody>
    </table>

    <div id="reportPage" class="report-page">
        <div class="result-item">
            <label>📊 測試結果摘要：</label>
            <div style="font-size: 1.1rem;">平均耗時：<span id="avgDisplay" style="color:#007bff; font-weight:bold;">0</span> 秒</div>
            <div style="font-size: 0.9rem; color:#666;" id="rawTimes"></div>
        </div>

        <div class="result-item">
            <label>👤 病人資訊：</label>
            <input type="text" id="patientId" placeholder="輸入病歷號">
            <label>🎯 選擇參考族群 (常模)：</label>
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

        <div class="result-item" style="border-bottom:none;">
            <label>📧 傳送報告至 Email：</label>
            <input type="email" id="targetEmail" placeholder="收件者 Email">
            <button id="emailBtn">發送電子郵件報告</button>
        </div>
    </div>

    <button id="resetBtn" style="margin-top: 20px;">重新開始測試</button>
    <div style="font-size: 0.8rem; color: #999; margin-top: 10px;">Reference: Physiopedia</div>
</div>

<script>
    let start, results = [], stream;
    const btn = document.getElementById('mainBtn');
    const v = document.getElementById('v');
    const reportPage = document.getElementById('reportPage');

    btn.onclick = async () => {
        if (!stream) {
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } });
                v.srcObject = stream;
                btn.textContent = "開始第 1 次計時";
                document.getElementById('testCount').disabled = true;
            } catch (e) { alert("請確保使用 HTTPS 並開啟相機權限"); }
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
            row.style.borderBottom = "1px solid #eee";
            row.innerHTML = `<td style="padding:10px;">第 ${results.length} 次</td><td style="padding:10px; font-weight:bold;">${time} s</td>`;
            
            if (results.length < max) {
                btn.textContent = `開始第 ${results.length + 1} 次計時`;
                btn.classList.remove('active');
            } else {
                btn.textContent = "測試完成";
                btn.disabled = true;
                btn.classList.remove('active');
                finishTest();
            }
        }
    };

    function finishTest() {
        const avg = (results.reduce((s, r) => s + r) / results.length).toFixed(2);
        document.getElementById('avgDisplay').textContent = avg;
        document.getElementById('rawTimes').textContent = `各次紀錄：${results.join('s, ')}s`;
        reportPage.style.display = 'block';
        updateRisk();
        // 自動捲動到結果區
        reportPage.scrollIntoView({ behavior: 'smooth' });
    }

    function updateRisk() {
        const avg = parseFloat(document.getElementById('avgDisplay').textContent);
        const cutOff = parseFloat(document.getElementById('popType').value);
        const riskDiv = document.getElementById('riskDisplay');
        
        if (avg >= cutOff) {
            riskDiv.innerHTML = `⚠️ 評估：高跌倒風險<br><small>(標準切點: ${cutOff}s)</small>`;
            riskDiv.style.background = "#fff5f5";
            riskDiv.style.color = "#c92a2a";
        } else {
            riskDiv.innerHTML = `✅ 評估：行動力良好<br><small>(標準切點: ${cutOff}s)</small>`;
            riskDiv.style.background = "#f8f9fa";
            riskDiv.style.color = "#2b8a3e";
        }
    }

    document.getElementById('emailBtn').onclick = () => {
        const pId = document.getElementById('patientId').value || "未填寫";
        const email = document.getElementById('targetEmail').value;
        const avg = document.getElementById('avgDisplay').textContent;
        const popName = document.getElementById('popType').options[document.getElementById('popType').selectedIndex].text;
        const riskText = document.getElementById('riskDisplay').innerText.split('\n')[0];

        if (!email) { alert("請輸入 Email"); return; }

        const subject = encodeURIComponent(`TUG 評估報告 - ID: ${pId}`);
        const body = encodeURIComponent(
            `Timed Up and Go (TUG) 測試報告\r\n` +
            `----------------------------------\r\n` +
            `病歷號: ${pId}\r\n` +
            `受測族群: ${popName}\r\n` +
            `測試紀錄: ${results.join('s, ')}s\r\n` +
            `平均耗時: ${avg}s\r\n` +
            `臨床結論: ${riskText}\r\n` +
            `----------------------------------\r\n` +
            `Ref: Physiopedia`
        );
        window.location.href = `mailto:${email}?subject=${subject}&body=${body}`;
    };

    document.getElementById('resetBtn').onclick = () => location.reload();
</script>
</body>
</html>
