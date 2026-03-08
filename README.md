<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TUG 專業臨床評估工具</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; text-align: center; background: #f4f7f9; margin: 0; padding: 15px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 20px; border-radius: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); }
        video { width: 100%; border-radius: 12px; background: #222; margin-bottom: 10px; transform: scaleX(1); }
        .setup-sec { margin-bottom: 15px; padding: 10px; background: #e9ecef; border-radius: 10px; }
        .controls { display: flex; flex-direction: column; gap: 12px; }
        button { padding: 18px; font-size: 1.2rem; border: none; border-radius: 12px; cursor: pointer; transition: 0.2s; font-weight: bold; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.recording { background: #dc3545; animation: pulse 1.5s infinite; }
        @keyframes pulse { 0% { opacity: 1; } 50% { opacity: 0.8; } 100% { opacity: 1; } }
        .results-table { width: 100%; border-collapse: collapse; margin: 15px 0; font-size: 0.9rem; }
        .results-table th, .results-table td { border-bottom: 1px solid #eee; padding: 12px; }
        .report-box { padding: 20px; border-radius: 12px; margin-top: 15px; line-height: 1.6; }
        .high-risk { background: #fff5f5; color: #c92a2a; border: 2px solid #ffc9c9; }
        .low-risk { background: #f8f9fa; color: #2b8a3e; border: 2px solid #b2f2bb; }
        .download-link { display: block; margin-top: 8px; color: #007bff; text-decoration: underline; font-size: 0.8rem; }
    </style>
</head>
<body>

<div class="container">
    <h2 style="color: #333;">TUG 測試評估</h2>
    
    <div id="setupArea" class="setup-sec">
        <label>選擇測試總次數：</label>
        <select id="testCountSelect" style="font-size: 1rem; padding: 5px;">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <video id="preview" autoplay muted playsinline></video>
    
    <div class="controls">
        <button id="mainBtn">啟動相機</button>
        <button id="resetBtn" style="background: #dee2e6; color: #495057; font-size: 0.9rem; padding: 10px;">重新開始</button>
    </div>

    <table class="results-table">
        <thead>
            <tr>
                <th>次數</th>
                <th>時間 (s)</th>
                <th>錄影檔</th>
            </tr>
        </thead>
        <tbody id="resultsBody"></tbody>
    </table>

    <div id="report" class="report-box" style="display: none;">
        <div style="font-size: 1.1rem;">本次平均時間：<span id="avgTime">0</span> 秒</div>
        <div style="font-size: 0.9rem; margin: 5px 0; color: #666;">( 正常標準參考值：< 12 秒 )</div>
        <hr style="border: 0.5px solid #ccc;">
        <div id="riskLevel" style="font-size: 1.2rem;"></div>
    </div>
</div>

<script>
    let mediaRecorder;
    let recordedChunks = [];
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

    // 啟動相機 (後置鏡頭)
    async function startCamera() {
        try {
            stream = await navigator.mediaDevices.getUserMedia({ 
                video: { facingMode: "environment" }, // 強制使用後置相機
                audio: true 
            });
            preview.srcObject = stream;
            mainBtn.textContent = "開始測試 (第 1 次)";
            document.getElementById('setupArea').style.opacity = "0.5";
            testCountSelect.disabled = true;
        } catch (err) {
            alert("無法開啟後置相機，請檢查權限設定。");
        }
    }

    mainBtn.addEventListener('click', () => {
        if (!stream) {
            startCamera();
            return;
        }

        maxTests = parseInt(testCountSelect.value);
        if (results.length >= maxTests) return;

        if (mainBtn.textContent.includes('開始')) {
            // --- 開始測試 ---
            recordedChunks = [];
            mediaRecorder = new MediaRecorder(stream);
            mediaRecorder.ondataavailable = (e) => { if (e.data.size > 0) recordedChunks.push(e.data); };
            
            // 當錄影停止時，產生下載連結
            mediaRecorder.onstop = () => {
                const blob = new Blob(recordedChunks, { type: 'video/webm' });
                const url = URL.createObjectURL(blob);
                const lastRow = resultsBody.lastChild;
                const downloadLink = document.createElement('a');
                downloadLink.href = url;
                downloadLink.download = `TUG_Test_${results.length}.webm`;
                downloadLink.className = "download-link";
                downloadLink.textContent = "下載影片";
                lastRow.cells[2].appendChild(downloadLink);
            };

            mediaRecorder.start();
            startTime = performance.now();
            mainBtn.textContent = '停止計時 (結束)';
            mainBtn.classList.add('recording');
        } else {
            // --- 結束測試 ---
            mediaRecorder.stop();
            const duration = ((performance.now() - startTime) / 1000).toFixed(2);
            results.push(parseFloat(duration));
            
            updateTable(duration);
            
            if (results.length < maxTests) {
                mainBtn.textContent = `開始測試 (第 ${results.length + 1} 次)`;
                mainBtn.classList.remove('recording');
            } else {
                mainBtn.textContent = '測試全部完成';
                mainBtn.disabled = true;
                mainBtn.classList.remove('recording');
                calculateFinalReport();
            }
        }
    });

    function updateTable(time) {
        const row = resultsBody.insertRow();
        row.insertCell(0).innerText = `第 ${results.length} 次`;
        row.insertCell(1).innerText = `${time} s`;
        row.insertCell(2).innerText = ""; // 留給影片下載連結
    }

    function calculateFinalReport() {
        const avg = (results.reduce((a, b) => a + b) / results.length).toFixed(2);
        avgTimeDisplay.textContent = avg;
        report.style.display = 'block';

        if (avg >= 12) {
            riskLevelDisplay.innerHTML = "評估：<span style='color:red;'>高跌倒風險</span><br><small>平均時間大於標準值 12秒</small>";
            report.className = "report-box high-risk";
        } else {
            riskLevelDisplay.innerHTML = "評估：<span style='color:green;'>行動力良好</span><br><small>平均時間優於標準值 12秒</small>";
            report.className = "report-box low-risk";
        }
    }

    resetBtn.addEventListener('click', () => {
        location.reload(); // 簡單直接的重置方式
    });
</script>

</body>
</html>
