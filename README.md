<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TUG 專業評估工具</title>
    <style>
        body { font-family: -apple-system, sans-serif; text-align: center; background: #f8f9fa; margin: 0; padding: 15px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 20px; border-radius: 20px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); }
        
        /* 螢幕錄影提示區 */
        .record-hint { background: #fff3bf; color: #856404; padding: 12px; border-radius: 10px; font-size: 1rem; margin-bottom: 15px; border: 1px solid #ffe066; font-weight: bold; line-height: 1.4; }
        
        video { width: 100%; border-radius: 12px; background: #000; margin-bottom: 10px; }
        .setup-sec { margin-bottom: 15px; padding: 12px; background: #e9ecef; border-radius: 10px; }
        .controls { display: flex; flex-direction: column; gap: 12px; }
        
        button { padding: 20px; font-size: 1.3rem; border: none; border-radius: 12px; cursor: pointer; font-weight: bold; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.active { background: #dc3545; }
        
        .results-table { width: 100%; border-collapse: collapse; margin: 15px 0; font-size: 1.1rem; }
        .results-table th, .results-table td { border-bottom: 1px solid #eee; padding: 12px; }
        
        .report-box { padding: 20px; border-radius: 12px; margin-top: 15px; line-height: 1.6; border: 2px solid #ddd; }
        .high-risk { background: #fff5f5; color: #c92a2a; border-color: #ffc9c9; }
        .low-risk { background: #f8f9fa; color: #2b8a3e; border-color: #b2f2bb; }
    </style>
</head>
<body>

<div class="container">
    <h2 style="margin-bottom: 5px;">TUG 臨床評估</h2>
    
    <div class="record-hint">
        🔴 重要提示：<br>請先「打開螢幕錄影」再開始測試，<br>即可儲存含時間數據的評估影片。
    </div>

    <div id="setupArea" class="setup-sec">
        <label>設定測試總次數：</label>
        <select id="testCountSelect" style="font-size: 1.1rem; padding: 5px;">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <video id="preview" autoplay muted playsinline></video>
    
    <div class="controls">
        <button id="mainBtn">啟動後置相機</button>
        <button id="resetBtn" style="background: #dee2e6; color: #495057; font-size: 0.9rem; padding: 10px; margin-top: 10px;">清除數據 / 重新開始</button>
    </div>

    <table class="results-table">
        <thead>
            <tr>
                <th>測試序號</th>
                <th>耗時 (秒)</th>
            </tr>
        </thead>
        <tbody id="resultsBody"></tbody>
    </table>

    <div id="report" class="report-box" style="display: none;">
        <div style="font-size: 1.2rem; font-weight: bold;">平均時間：<span id="avgTime">0</span> 秒</div>
        <div style="font-size: 1rem; color: #666; margin-bottom: 10px;">( 正常標準時間：< 12 秒 )</div>
        <div id="riskLevel" style="font-size: 1.3rem; font-weight: bold; border-top: 1px solid #ccc; pt: 10px;"></div>
    </div>
</div>

<script>
    let startTime;
    let results = [];
    let stream;

    const mainBtn = document.getElementById('mainBtn');
    const resetBtn = document.getElementById('resetBtn');
    const resultsBody = document.getElementById('resultsBody');
    const report = document.getElementById('report');
    const avgTimeDisplay = document.getElementById('avgTime');
    const riskLevelDisplay = document.getElementById('riskLevel');
    const preview = document.getElementById('preview');
    const testCountSelect = document.getElementById('testCountSelect');

    async function startCamera() {
        try {
            stream = await navigator.mediaDevices.getUserMedia({ 
                video: { facingMode: "environment" }, 
                audio: false 
            });
            preview.srcObject = stream;
            mainBtn.textContent = "開始第 1 次計時";
            testCountSelect.disabled = true;
            document.getElementById('setupArea').style.opacity = "0.6";
        } catch (err) {
            alert("無法開啟相機，請確認已授權。");
        }
    }

    mainBtn.addEventListener('click', () => {
        if (!stream) {
            startCamera();
            return;
        }

        const maxTests = parseInt(testCountSelect.value);
        if (results.length >= maxTests) return;

        if (!startTime) {
            startTime = performance.now();
            mainBtn.textContent = '結束測試 (受測者坐下)';
            mainBtn.classList.add('active');
        } else {
            const duration = ((performance.now() - startTime) / 1000).toFixed(2);
            results.push(parseFloat(duration));
            startTime = null;
            
            const row = resultsBody.insertRow();
            row.insertCell(0).innerText = `第 ${results.length} 次`;
            row.insertCell(1).innerText = `${duration} s`;
            
            if (results.length < maxTests) {
                mainBtn.textContent = `開始第 ${results.length + 1} 次計時`;
                mainBtn.classList.remove('active');
            } else {
                mainBtn.textContent = '評估完成';
                mainBtn.disabled = true;
                mainBtn.classList.remove('active');
                
                const avg = (results.reduce((a, b) => a + b) / results.length).toFixed(2);
                avgTimeDisplay.textContent = avg;
                report.style.display = 'block';

                if (avg >= 12) {
                    riskLevelDisplay.innerHTML = "評估結果：<span style='color:#c92a2a;'>高跌倒風險</span>";
                    report.className = "report-box high-risk";
