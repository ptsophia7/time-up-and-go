<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TUG 3米起立行走測試計時器</title>
    <style>
        body { font-family: sans-serif; text-align: center; background: #f0f2f5; margin: 0; padding: 20px; }
        .container { max-width: 600px; margin: auto; background: white; padding: 20px; border-radius: 15px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        video { width: 100%; border-radius: 10px; background: #000; margin-bottom: 10px; }
        .controls { display: flex; flex-direction: column; gap: 15px; }
        button { padding: 15px; font-size: 1.2rem; border: none; border-radius: 8px; cursor: pointer; transition: 0.3s; }
        #mainBtn { background: #007bff; color: white; font-weight: bold; }
        #mainBtn.recording { background: #dc3545; }
        #resetBtn { background: #6c757d; color: white; }
        .results-table { width: 100%; border-collapse: collapse; margin: 20px 0; }
        .results-table th, .results-table td { border: 1px solid #ddd; padding: 10px; }
        .report-box { padding: 15px; border-radius: 8px; font-weight: bold; margin-top: 15px; }
        .high-risk { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
        .low-risk { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
    </style>
</head>
<body>

<div class="container">
    <h2>TUG 臨床評估工具</h2>
    
    <video id="preview" autoplay muted playsinline></video>
    
    <div class="controls">
        <button id="mainBtn">開始測試 (第 1 次)</button>
        <button id="resetBtn">重置所有紀錄</button>
    </div>

    <table class="results-table">
        <thead>
            <tr>
                <th>測試次數</th>
                <th>時間 (秒)</th>
            </tr>
        </thead>
        <tbody id="resultsBody">
            </tbody>
    </table>

    <div id="report" class="report-box" style="display: none;">
        <div>平均時間：<span id="avgTime">0</span> 秒</div>
        <div id="riskLevel">評估結果：--</div>
    </div>
</div>

<script>
    let mediaRecorder;
    let startTime;
    let results = [];
    const maxTests = 3;
    
    const mainBtn = document.getElementById('mainBtn');
    const resetBtn = document.getElementById('resetBtn');
    const resultsBody = document.getElementById('resultsBody');
    const report = document.getElementById('report');
    const avgTimeDisplay = document.getElementById('avgTime');
    const riskLevelDisplay = document.getElementById('riskLevel');
    const preview = document.getElementById('preview');

    // 初始化相機錄影
    navigator.mediaDevices.getUserMedia({ video: true, audio: false })
        .then(stream => { preview.srcObject = stream; })
        .catch(err => alert("請允許開啟相機權限：" + err));

    mainBtn.addEventListener('click', () => {
        if (results.length >= maxTests) return;

        if (mainBtn.textContent.includes('開始')) {
            // 開始計時與錄影
            startTime = performance.now();
            mainBtn.textContent = '結束測試';
            mainBtn.classList.add('recording');
            // 這裡可加入 mediaRecorder.start() 來實際儲存影片
        } else {
            // 結束計時
            const endTime = performance.now();
            const duration = ((endTime - startTime) / 1000).toFixed(2);
            results.push(parseFloat(duration));
            
            updateUI();
            
            if (results.length < maxTests) {
                mainBtn.textContent = `開始測試 (第 ${results.length + 1} 次)`;
                mainBtn.classList.remove('recording');
            } else {
                mainBtn.textContent = '測試完成';
                mainBtn.disabled = true;
                mainBtn.classList.remove('recording');
                calculateFinalReport();
            }
        }
    });

    function updateUI() {
        resultsBody.innerHTML = results.map((time, index) => 
            `<tr><td>第 ${index + 1} 次</td><td>${time} s</td></tr>`
        ).join('');
    }

    function calculateFinalReport() {
        const avg = (results.reduce((a, b) => a + b) / results.length).toFixed(2);
        avgTimeDisplay.textContent = avg;
        report.style.display = 'block';

        // 臨床常模邏輯
        if (avg >= 12) {
            riskLevelDisplay.textContent = "評估結果：高跌倒風險 (>12秒)";
            report.className = "report-box high-risk";
        } else {
            riskLevelDisplay.textContent = "評估結果：低跌倒風險 (<12秒)";
            report.className = "report-box low-risk";
        }
    }

    resetBtn.addEventListener('click', () => {
        results = [];
        resultsBody.innerHTML = '';
        report.style.display = 'none';
        mainBtn.disabled = false;
        mainBtn.textContent = '開始測試 (第 1 次)';
        mainBtn.classList.remove('recording');
    });
</script>

</body>
</html>
