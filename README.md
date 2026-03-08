<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>TUG 臨床評估工具</title>
    <style>
        body { font-family: -apple-system, sans-serif; text-align: center; background: #f0f2f5; margin: 0; padding: 10px; }
        .container { max-width: 500px; margin: auto; background: white; padding: 15px; border-radius: 20px; box-shadow: 0 4px 10px rgba(0,0,0,0.1); }
        .hint { background: #fff3bf; color: #856404; padding: 10px; border-radius: 10px; font-size: 0.9rem; margin-bottom: 15px; font-weight: bold; border: 1px solid #ffe066; }
        video { width: 100%; border-radius: 12px; background: #000; margin-bottom: 10px; }
        .setup { margin-bottom: 15px; padding: 10px; background: #e9ecef; border-radius: 10px; }
        button { width: 100%; padding: 18px; font-size: 1.2rem; border: none; border-radius: 12px; cursor: pointer; font-weight: bold; margin-bottom: 10px; }
        #mainBtn { background: #007bff; color: white; }
        #mainBtn.active { background: #dc3545; }
        #resetBtn { background: #6c757d; color: white; font-size: 0.9rem; padding: 10px; }
        table { width: 100%; border-collapse: collapse; margin-top: 15px; }
        th, td { border-bottom: 1px solid #eee; padding: 10px; }
        .report { padding: 15px; border-radius: 12px; margin-top: 15px; border: 2px solid #ddd; display: none; }
        .high { background: #fff5f5; color: #c92a2a; border-color: #ffc9c9; }
        .low { background: #f8f9fa; color: #2b8a3e; border-color: #b2f2bb; }
    </style>
</head>
<body>

<div class="container">
    <h3 style="margin: 10px 0;">TUG 評估 (3米起步走)</h3>
    <div class="hint">🔴 提示：請先開啟「螢幕錄影」<br>再點擊按鈕開始測試</div>

    <div id="setupArea" class="setup">
        <label>測試總次數：</label>
        <select id="testCount" style="font-size: 1rem;">
            <option value="1">1 次</option>
            <option value="2">2 次</option>
            <option value="3" selected>3 次</option>
        </select>
    </div>

    <video id="v" autoplay muted playsinline></video>
    
    <button id="mainBtn">啟動相機</button>
    <button id="resetBtn">重新開始</button>

    <table id="resTable">
        <thead><tr><th>測試</th><th>耗時 (秒)</th></tr></thead>
        <tbody id="resBody"></tbody>
    </table>

    <div id="report" class="report">
        <div style="font-size: 1.1rem; font-weight: bold;">平均時間：<span id="avg">0</span> 秒</div>
        <div style="font-size: 0.85rem; color: #666;">( 正常標準時間：< 12 秒 )</div>
        <div id="risk" style="font-size: 1.2rem; font-weight: bold; margin-top: 10px;"></div>
    </div>
</div>

<script>
    let start, results = [], stream;
    const btn = document.getElementById('mainBtn');
    const v = document.getElementById('v');

    btn.onclick = async () => {
        if (!stream) {
            try {
                stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } });
                v.srcObject = stream;
                btn.textContent = "開始第 1 次計時";
                document.getElementById('testCount').disabled = true;
            } catch (e) {
                alert("錯誤：請確認網址為 HTTPS 且已允許相機權限");
            }
            return;
        }

        const max = parseInt(document.getElementById('testCount').value);
        if (results.length >= max) return;

        if (!start) {
            start = performance.now();
            btn.textContent = "結束 (坐下時點擊)";
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

    function showReport() {
        const a = (results.reduce((s, r) => s + r) / results.length).toFixed(2);
        document.getElementById('avg').textContent = a;
        document.getElementById('report').style.display = 'block';
        const r = document.getElementById('risk');
        if (a >= 12) {
            r.innerHTML = "結果：<span style='color:red;'>高跌倒風險</span>";
            document.getElementById('report').className = "report high";
        } else {
            r.innerHTML = "結果：<span style='color:green;'>行動良好</span>";
            document.getElementById('report').className = "report low";
        }
    }

    document.getElementById('resetBtn').onclick = () => location.reload();
</script>
</body>
</html>
