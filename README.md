# Rajdeep
<!DOCTYPE html>
<html>
<head>
    <title>3-Phase Power Analysis</title>
    <script src="https://cdn.plot.ly/plotly-2.20.0.min.js"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .phase-container { display: flex; gap: 20px; margin-bottom: 20px; }
        .phase { border: 1px solid #ccc; padding: 15px; border-radius: 5px; }
        .input-group { margin: 10px 0; }
        label { display: inline-block; width: 60px; }
        input { width: 80px; padding: 5px; }
        #plot { width: 100%; height: 1500px; }
        button { padding: 10px 20px; margin: 10px 0; }
    </style>
</head>
<body>
    <h1>3-Phase Load Analysis</h1>
    
    <div class="phase-container">
        <div class="phase" style="border-color: red;">
            <h3>Phase R</h3>
            <div class="input-group">
                <label>R (Ω):</label>
                <input type="number" id="R-R" value="10" step="0.1">
            </div>
            <div class="input-group">
                <label>L (H):</label>
                <input type="number" id="L-R" value="0.05" step="0.01">
            </div>
            <div class="input-group">
                <label>C (F):</label>
                <input type="number" id="C-R" value="0.001" step="0.001">
            </div>
        </div>
        
        <div class="phase" style="border-color: green;">
            <h3>Phase Y</h3>
            <div class="input-group">
                <label>R (Ω):</label>
                <input type="number" id="R-Y" value="10" step="0.1">
            </div>
            <div class="input-group">
                <label>L (H):</label>
                <input type="number" id="L-Y" value="0.05" step="0.01">
            </div>
            <div class="input-group">
                <label>C (F):</label>
                <input type="number" id="C-Y" value="0.001" step="0.001">
            </div>
        </div>
        
        <div class="phase" style="border-color: blue;">
            <h3>Phase B</h3>
            <div class="input-group">
                <label>R (Ω):</label>
                <input type="number" id="R-B" value="10" step="0.1">
            </div>
            <div class="input-group">
                <label>L (H):</label>
                <input type="number" id="L-B" value="0.05" step="0.01">
            </div>
            <div class="input-group">
                <label>C (F):</label>
                <input type="number" id="C-B" value="0.001" step="0.001">
            </div>
        </div>
    </div>

    <div class="input-group">
        <label>ω (rad/s):</label>
        <input type="number" id="omega" value="314" step="1">
    </div>
    
    <div class="input-group">
        <label>Balanced Load:</label>
        <input type="checkbox" id="balanced" onchange="syncBalanced()">
    </div>
    
    <button onclick="plot()">Plot Results</button>
    <div id="plot"></div>

    <script>
        const Vm = 325;  // Peak voltage (230V RMS)
        
        function syncBalanced() {
            const balanced = document.getElementById('balanced').checked;
            ['Y', 'B'].forEach(phase => {
                ['R', 'L', 'C'].forEach(param => {
                    const element = document.getElementById(`${param}-${phase}`);
                    element.disabled = balanced;
                });
            });

            if (balanced) {
                const R = document.getElementById('R-R').value;
                const L = document.getElementById('L-R').value;
                const C = document.getElementById('C-R').value;
                
                ['Y', 'B'].forEach(phase => {
                    document.getElementById(`R-${phase}`).value = R;
                    document.getElementById(`L-${phase}`).value = L;
                    document.getElementById(`C-${phase}`).value = C;
                });
            }
        }

        function getLoadValues(phase) {
            return {
                R: parseFloat(document.getElementById(`R-${phase}`).value) || 0,
                L: parseFloat(document.getElementById(`L-${phase}`).value) || 0,
                C: parseFloat(document.getElementById(`C-${phase}`).value) || 0
            };
        }

        function plot() {
            const omega = parseFloat(document.getElementById('omega').value);
            if (!omega) {
                alert('Please enter a valid ω value');
                return;
            }

            const phases = ['R', 'Y', 'B'];
            const data = phases.map(phase => {
                const load = getLoadValues(phase);
                const X = omega * load.L - 1/(omega * load.C);
                const Z = Math.sqrt(load.R**2 + X**2);
                const phi = Math.atan2(X, load.R);
                return { load, Z, phi };
            });

            const tMax = (5 * 2 * Math.PI) / omega;
            const time = Array.from({length: 500}, (_, i) => tMax * i / 500);

            const traces = [];
            const colors = { R: 'red', Y: 'green', B: 'blue' };
            const pTotal = new Array(time.length).fill(0);

            phases.forEach((phase, i) => {
                const phaseShift = i * -120 * Math.PI/180;
                const { Z, phi } = data[i];
                
                // Voltage waveform
                const V = time.map(t => Vm * Math.sin(omega * t + phaseShift));
                
                // Current waveform
                const I = time.map(t => (Vm / Z) * Math.sin(omega * t + phaseShift - phi));
                
                // Instantaneous power
                const p = V.map((v, idx) => v * I[idx]);
                p.forEach((val, idx) => pTotal[idx] += val);

                traces.push({
                    x: time,
                    y: V,
                    name: `V_${phase}`,
                    line: { color: colors[phase] },
                    xaxis: 'x1',
                    yaxis: 'y1'
                });

                traces.push({
                    x: time,
                    y: I,
                    name: `I_${phase}`,
                    line: { color: colors[phase], dash: 'dot' },
                    xaxis: 'x2',
                    yaxis: 'y2'
                });

                traces.push({
                    x: time,
                    y: p,
                    name: `P_inst_${phase}`,
                    line: { color: colors[phase] },
                    xaxis: 'x3',
                    yaxis: 'y3'
                });
            });

            // Total power trace
            traces.push({
                x: time,
                y: pTotal,
                name: 'Total Power',
                line: { color: 'black' },
                xaxis: 'x3',
                yaxis: 'y3'
            });

            const layout = {
                grid: {
                    rows: 3,
                    columns: 1,
                    pattern: 'independent'
                },
                title: '3-Phase Power Analysis',
                showlegend: true,
                height: 1500,
                xaxis1: { title: 'Time (s)', anchor: 'y1' },
                yaxis1: { title: 'Voltage (V)', domain: [0.66, 1] },
                xaxis2: { title: 'Time (s)', anchor: 'y2' },
                yaxis2: { title: 'Current (A)', domain: [0.33, 0.66] },
                xaxis3: { title: 'Time (s)', anchor: 'y3' },
                yaxis3: { title: 'Power (W)', domain: [0, 0.33] },
            };

            Plotly.newPlot('plot', traces, layout);
        }

        // Initialize balanced load sync
        document.querySelectorAll('#R-R, #L-R, #C-R').forEach(input => {
            input.addEventListener('input', () => {
                if (document.getElementById('balanced').checked) syncBalanced();
            });
        });
        syncBalanced();
    </script>
</body>
</html>
