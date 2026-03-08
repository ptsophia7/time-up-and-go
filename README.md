<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TUG 專業臨床評估工具</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; text-align: center; background: #f4f7f9; margin: 0; padding: 15px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 20px; border-radius: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); }
        
        /* 螢幕錄影提示 */
        .record-hint { background: #fff3bf; color: #856404; padding: 10px; border-radius: 8px; font-size: 0.9rem; margin-bottom: 15px; border: 1px solid #ffe066; font-weight: bold; }
        
        video { width: 100%; border-radius: 12px; background: #222; margin-bottom: 10px; }
        .setup-sec { margin-bottom: 15px; padding: 10px; background: #e9ecef; border-radius: 10px; }
        .controls { display: flex; flex-direction: column; gap: 12px; }
        button { padding: 18px; font-size: 1.2rem; border: none; border-radius: 12px; cursor: pointer; transition: 0.2s; font-weight: bold; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.active { background: #dc3545; }
        
        .results-table { width: 100%; border-collapse: collapse; margin: 15px 0; font-size: 1rem; }
        .results-table th, .results-table td { border-bottom: 1px solid #eee; padding: 12px; }
        
        .report-box { padding: 20px; border-radius: 12px; margin-top: 15px; line-height: 1.6; }
        .high-risk { background: #fff5f5; color: #c92a2a; border: 2px solid #ffc9c9; }
        .low-risk { background: #f8f9fa; color: #2b8a3e; border: 2px solid #b2f2bb; }
        .norm-info { font-size: 0.85rem; color: #666; margin-top: 5px; }
    </style>
</head>
<body>

<div class="container">
    <h2 style="color: #333; margin-bottom: 5px;">TUG 測試評估</h2>
    <div class="norm-info">3公尺起立行走測試 (Timed Up and Go)</div>
    
    <div class="record-hint">
        💡 提示：請先打開手機「螢幕錄影」再開始測試，即可完整儲存評估過程。
    </div>

    <div id="setupArea" class="setup-sec">
        <label>設定測試總次數：</label>
        <select id="testCountSelect" style="font-size: 1rem; padding: 5px; border-radius: 5px;">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <video id="preview" autoplay muted playsinline></video>
    
    <div class="controls">
        <button id="mainBtn">啟動相機</button>
        <button id="resetBtn" style="background: #dee2e6; color: #495057; font-size: 0.9rem; padding: 10px;">清除數據重新開始</button>
    </div>

    <table class="results-table">
        <thead>
            <tr>
                <th>測試次數</th>
                <th>耗時 (秒)</th>
            </tr>
        </thead>
        <tbody id="resultsBody"></tbody>
    </table>

    <div id="report" class="report-box" style="display: none;">
        <div style="font-size: 1.1rem;">本次平均時間：<span id="avgTime">0</span> 秒</div>
        <div style="font-size: 0.9rem; margin: 5px 0; color: #666;">( 正常標準時間：< 12 秒 )</div>
        <hr style="border: 0.5px solid #ccc; margin: 10px 0;">
        <div id="riskLevel" style="font-size: 1.2rem;"></div>
    </div>
</div>

<script>
    let startTime;
    let results = [];
    let maxTests = 3;
    let stream;

    const mainBtn = document.getElementById('mainBtn');
    const resetBtn = document.getElementById('resetBtn');
    const resultsBody = document.getElementById('resultsBody');
    const report = document.getElementById('report');
    const avgTimeDisplay = document.getElementById('avgTime');
    const riskLevelDisplay = document.getElementById('riskLevel');
    const preview = document.getElementById('preview');
    const testCountSelect = document.getElementById('testCountSelect');

    // 啟動相機 (後置)
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

        maxTests = parseInt(testCountSelect.value);
        if (results.length >= maxTests) return;

        if (!startTime) {
            // --- 開始計時 ---
            startTime = performance.now();
            mainBtn.textContent = '點擊結束 (坐下)';
            mainBtn.classList.add('active');
        } else {
            // --- 結束計時 ---
            const duration = ((performance.now() - startTime) / 1000).toFixed(2);
            results.push(parseFloat(duration));
            startTime = null; // 重置起點
            
            updateTable(duration);
            
            if (results.length < maxTests) {
                mainBtn.textContent = `開始第 ${results.length + 1} 次計時`;
                mainBtn.classList.remove('active');
            } else {
                mainBtn.textContent = '測試全部完成';
                mainBtn.disabled = true;
                mainBtn.classList.remove('active');
                calculateFinalReport();
            }
        }
    });

    function updateTable(time) {
        const row = resultsBody.insertRow();
        row.insertCell(0).innerText = `第 ${results.length} 次`;
        row.insertCell(1).innerText = `${time} s`;
    }

    function calculateFinalReport() {
        const avg = (results.reduce((a, b) => a + b) / results.length).toFixed(2);
        avgTimeDisplay.textContent = avg;
        report.style.display = 'block';

        if (avg >= 12) {
            riskLevelDisplay.innerHTML = "評估結果：<span style='color:#c92a2a;'>高跌倒風險</span><br><small>平均時間 ≥ 12秒 標準值</small>";
            report.className = "report-box high-risk";
        } else {
            riskLevelDisplay.innerHTML = "評估結果：<span style='color:#2b8a3e;'>行動能力良好</span><br><small>平均時間 < 12秒 標準值</small>";
            report.className = "report-box low-risk";
        }
    }

    resetBtn.addEventListener('click', () => {
        location.reload(); 
    });
</script>

</body>
</html>
