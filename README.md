<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PrintQuote - 3D Cost Calculator</title>
    
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Chart.js -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Font Awesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');
        
        body { 
            font-family: 'Inter', sans-serif; 
            background-color: #f1f5f9; 
            color: #0f172a;
            -webkit-font-smoothing: antialiased;
        }
        
        /* Smooth Transitions */
        .transition-all { transition-property: all; transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1); transition-duration: 200ms; }

        /* Range Slider Styling */
        input[type=range] { -webkit-appearance: none; background: transparent; }
        input[type=range]::-webkit-slider-thumb {
            -webkit-appearance: none; height: 18px; width: 18px; border-radius: 50%;
            background: #4f46e5; margin-top: -7px; cursor: pointer; box-shadow: 0 2px 6px rgba(79, 70, 229, 0.4);
        }
        input[type=range]::-webkit-slider-runnable-track {
            width: 100%; height: 4px; background: #cbd5e1; border-radius: 2px;
        }

        /* --- PRINT MODE STYLING --- */
        @media print {
            /* Hide everything generally */
            body * {
                visibility: hidden;
            }
            
            /* Make background white */
            body {
                background-color: white;
                margin: 0;
                padding: 0;
            }

            /* Target the specific invoice card to show */
            #quote-invoice, #quote-invoice * {
                visibility: visible;
            }

            /* Position the invoice at top-left of page */
            #quote-invoice {
                position: absolute;
                left: 0;
                top: 0;
                width: 100%;
                margin: 0;
                padding: 20px;
                box-shadow: none !important;
                border: 1px solid #ddd !important;
                border-radius: 0 !important;
            }

            /* Hide interactive elements inside the invoice during print */
            .no-print, button, input[type=range], select {
                display: none !important;
            }
            
            /* Ensure text inputs look like flat text */
            input {
                border: none !important;
                background: transparent !important;
                padding: 0 !important;
            }
            
            /* Hide the chart canvas to save ink/clean look (optional, can remove if you want graph) */
            canvas { display: none !important; }
            .chart-container { display: none !important; }
        }
    </style>
</head>
<body>

<div id="root"></div>

<script type="text/babel">
    const { useState, useEffect, useRef } = React;

    // --- CONSTANTS ---
    const CURRENCIES = { USD: "$", EUR: "€", GBP: "£" };

    // --- STL PARSER & ESTIMATOR ---
    const parseSTL = (buffer) => {
        try {
            const view = new DataView(buffer);
            const numTriangles = view.getUint32(80, true);
            let minX = Infinity, maxX = -Infinity, minY = Infinity, maxY = -Infinity, minZ = Infinity, maxZ = -Infinity;
            let totalVolume = 0;
            
            let offset = 84;
            for (let i = 0; i < numTriangles; i++) {
                // Vertices for Bounding Box
                const x1 = view.getFloat32(offset + 12, true); const y1 = view.getFloat32(offset + 16, true); const z1 = view.getFloat32(offset + 20, true);
                const x2 = view.getFloat32(offset + 24, true); const y2 = view.getFloat32(offset + 28, true); const z2 = view.getFloat32(offset + 32, true);
                const x3 = view.getFloat32(offset + 36, true); const y3 = view.getFloat32(offset + 40, true); const z3 = view.getFloat32(offset + 44, true);

                minX = Math.min(minX, x1, x2, x3); maxX = Math.max(maxX, x1, x2, x3);
                minY = Math.min(minY, y1, y2, y3); maxY = Math.max(maxY, y1, y2, y3);
                minZ = Math.min(minZ, z1, z2, z3); maxZ = Math.max(maxZ, z1, z2, z3);

                // Volume (Signed Tetrahedron)
                totalVolume += (-x3 * y2 * z1 + x2 * y3 * z1 + x3 * y1 * z2 - x1 * y3 * z2 - x2 * y1 * z3 + x1 * y2 * z3) / 6.0;
                offset += 50;
            }

            return { 
                x: (maxX - minX).toFixed(1), 
                y: (maxY - minY).toFixed(1), 
                z: (maxZ - minZ).toFixed(1),
                volumeCm3: Math.abs(totalVolume) / 1000
            };
        } catch (e) { console.error(e); return null; }
    };

    // --- COMPONENTS ---

    const InputGroup = ({ label, value, onChange, suffix, type="number", step="any", min }) => (
        <div className="group relative">
            <label className="input-group-label block text-[10px] font-bold text-slate-400 uppercase tracking-wider mb-1 group-focus-within:text-indigo-600 transition-colors">{label}</label>
            <div className="relative flex items-center">
                <input 
                    type={type}
                    step={step}
                    min={min}
                    value={value} 
                    onChange={e => onChange(e.target.value)} 
                    className="w-full bg-white text-slate-700 border-b border-slate-200 py-1.5 text-sm focus:outline-none focus:border-indigo-500 transition-colors font-semibold"
                />
                {suffix && <span className="absolute right-0 text-xs text-slate-400 font-medium pointer-events-none bg-white pl-1">{suffix}</span>}
            </div>
        </div>
    );

    const PriceOption = ({ title, price, symbol, selected, onClick, margin }) => (
        <div 
            onClick={onClick}
            className={`cursor-pointer rounded-lg p-3 border transition-all duration-200 ${selected 
                ? 'border-indigo-600 bg-indigo-50 ring-1 ring-indigo-600' 
                : 'border-slate-200 hover:border-indigo-300 hover:bg-white'}`}
        >
            <div className="flex justify-between items-center mb-1">
                <span className={`text-xs font-bold uppercase ${selected ? 'text-indigo-700' : 'text-slate-500'}`}>{title}</span>
                <span className="text-[10px] text-slate-400 bg-white px-1.5 py-0.5 rounded-full border border-slate-100">{margin}%</span>
            </div>
            <div className="text-lg font-bold text-slate-800">{symbol}{price}</div>
        </div>
    );

    // --- MAIN APP ---
    const App = () => {
        const [curr, setCurr] = useState('USD');
        const [showAdvanced, setShowAdvanced] = useState(true); 

        // State
        const [inputs, setInputs] = useState({
            partName: 'Untitled Project',
            filamentCost: 20.00,
            spoolWeight: 1000,
            printWeight: 257,
            printTimeHrs: 5,
            printTimeMin: 45,
            speed: 100, // mm/s
            
            watts: 155,
            kwhRate: 0.15,
            
            laborTime: 2,
            laborRate: 15,
            depreciation: 0.1,
            failureRate: 0,
            
            shipping: 0,
            handling: 5,
            customMargin: 14
        });

        const [dims, setDims] = useState(null);
        const chartRef = useRef(null);
        const chartInstanceRef = useRef(null);

        // Calculations
        const calculate = () => {
            const mat = (inputs.filamentCost / inputs.spoolWeight) * inputs.printWeight;
            const hrs = parseFloat(inputs.printTimeHrs) + (parseFloat(inputs.printTimeMin) / 60);
            const pwr = (inputs.watts / 1000) * hrs * inputs.kwhRate;
            const lab = (inputs.laborTime / 60) * inputs.laborRate;
            const dep = hrs * inputs.depreciation;
            
            const productionBase = mat + pwr + lab + dep;
            const failCost = productionBase * (inputs.failureRate / 100);
            const mfgCost = productionBase + failCost;
            const profit = mfgCost * (inputs.customMargin / 100);
            const extras = parseFloat(inputs.shipping) + parseFloat(inputs.handling);
            const final = mfgCost + profit + extras;
            
            return { mat, lab, pwr, dep, failCost, extras, mfgCost, final, profit };
        };

        const data = calculate();
        const sym = CURRENCIES[curr];

        // Chart Effect
        useEffect(() => {
            if (!chartRef.current) return;
            const ctx = chartRef.current.getContext('2d');
            if (chartInstanceRef.current) chartInstanceRef.current.destroy();

            chartInstanceRef.current = new Chart(ctx, {
                type: 'doughnut',
                data: {
                    labels: ['Material', 'Labor', 'Energy', 'Machine', 'Failure', 'Extras'],
                    datasets: [{
                        data: [data.mat, data.lab, data.pwr, data.dep, data.failCost, data.extras],
                        backgroundColor: ['#4f46e5', '#06b6d4', '#f59e0b', '#8b5cf6', '#ef4444', '#64748b'],
                        borderWidth: 0,
                        hoverOffset: 6
                    }]
                },
                options: {
                    cutout: '75%',
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: { legend: { display: false } }
                }
            });
            return () => { if (chartInstanceRef.current) chartInstanceRef.current.destroy(); };
        }, [inputs]);

        // File Handler & Estimation
        const handleFile = (e) => {
            const f = e.target.files[0];
            if(!f) return;
            const r = new FileReader();
            r.onload = (ev) => {
                const d = parseSTL(ev.target.result);
                if(d) {
                    setDims(d);
                    
                    // 1. Estimate Weight (PLA Density 1.24 g/cm3 * ~35% Infill/Walls/Tops)
                    const estWeight = (d.volumeCm3 * 1.24 * 0.35).toFixed(0);
                    
                    // 2. Estimate Time via Volumetric Flow Rate
                    // Assumptions: 0.4mm line width, 0.2mm layer height
                    // Flow (mm3/s) = Speed (mm/s) * 0.4 * 0.2
                    const nozzle = 0.4;
                    const layer = 0.2;
                    const volumetricFlow = inputs.speed * nozzle * layer; // mm3/s
                    
                    // Total Volume to print (Solids + Infill)
                    // We estimated weight based on 35% solidity, so let's use that volume
                    const printVolumeMm3 = (d.volumeCm3 * 1000) * 0.35;
                    
                    // Seconds = Volume / Flow
                    // Efficiency factor (0.8) accounts for acceleration, travel moves, heating
                    const efficiency = 0.8; 
                    const totalSeconds = printVolumeMm3 / (volumetricFlow * efficiency);
                    
                    const totalHours = totalSeconds / 3600;
                    
                    const estHrs = Math.floor(totalHours);
                    const estMins = Math.round((totalHours - estHrs) * 60);

                    setInputs(prev => ({
                        ...prev, 
                        partName: f.name.replace('.stl',''),
                        printWeight: estWeight,
                        printTimeHrs: estHrs,
                        printTimeMin: estMins
                    }));
                }
            };
            r.readAsArrayBuffer(f);
        };

        // JSON Handling
        const exportJSON = () => {
            const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(inputs));
            const node = document.createElement('a');
            node.setAttribute("href", dataStr);
            node.setAttribute("download", "print_settings.json");
            document.body.appendChild(node);
            node.click();
            node.remove();
        };
        const importJSON = (e) => {
            const f = e.target.files[0]; if(!f) return;
            const r = new FileReader();
            r.onload = (ev) => setInputs(JSON.parse(ev.target.result));
            r.readAsText(f);
        };

        return (
            <div className="min-h-screen py-10 px-4 md:px-8 max-w-6xl mx-auto">
                
                {/* Header */}
                <nav className="flex justify-between items-center mb-10 no-print">
                    <div className="flex items-center gap-3">
                        <div className="w-10 h-10 bg-indigo-600 rounded-xl flex items-center justify-center text-white shadow-lg shadow-indigo-200">
                            <i className="fas fa-cube text-lg"></i>
                        </div>
                        <div>
                            <h1 className="font-bold text-xl tracking-tight text-slate-800">PrintQuote</h1>
                            <p className="text-xs text-slate-400 font-medium">Professional Estimator</p>
                        </div>
                    </div>
                    <div className="flex items-center gap-4">
                         <label className="cursor-pointer text-xs font-bold text-slate-500 hover:text-indigo-600 transition-colors flex items-center gap-1">
                            <i className="fas fa-file-import"></i> Load
                            <input type="file" accept=".json" className="hidden" onChange={importJSON} />
                        </label>
                         <button onClick={exportJSON} className="text-xs font-bold text-slate-500 hover:text-indigo-600 transition-colors flex items-center gap-1">
                            <i className="fas fa-file-export"></i> Save
                        </button>
                        <select value={curr} onChange={e => setCurr(e.target.value)} className="text-sm font-bold text-slate-600 bg-transparent border-none focus:ring-0 cursor-pointer">
                            <option value="USD">USD ($)</option>
                            <option value="EUR">EUR (€)</option>
                            <option value="GBP">GBP (£)</option>
                        </select>
                        <button onClick={() => window.print()} className="bg-indigo-600 text-white px-4 py-2 rounded-lg text-sm font-bold shadow-md hover:bg-indigo-700 transition-all flex items-center gap-2">
                            <i className="fas fa-print"></i> Print Quote
                        </button>
                    </div>
                </nav>

                <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">

                    {/* LEFT: Configuration */}
                    <div className="lg:col-span-2 space-y-6 config-column no-print">
                        
                        {/* Project Info Section */}
                        <section className="bg-white rounded-2xl p-6 shadow-sm border border-slate-100">
                            <div className="flex justify-between items-start mb-6">
                                <h2 className="font-bold text-slate-800 flex items-center gap-2">
                                    <span className="w-1.5 h-1.5 rounded-full bg-indigo-500"></span> 1. Material & Specs
                                </h2>
                                <label className="cursor-pointer text-xs font-bold text-indigo-600 bg-indigo-50 px-3 py-1.5 rounded-full hover:bg-indigo-100 transition-colors no-print border border-indigo-100">
                                    <i className="fas fa-upload mr-1"></i> Analyze STL
                                    <input type="file" accept=".stl" className="hidden" onChange={handleFile} />
                                </label>
                            </div>
                            
                            <div className="grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-6">
                                <div className="md:col-span-2">
                                    <InputGroup label="Project Name" type="text" value={inputs.partName} onChange={v => setInputs({...inputs, partName: v})} />
                                    {dims && (
                                        <div className="mt-3 bg-slate-50 p-3 rounded-lg border border-slate-100 flex flex-wrap gap-4">
                                            <div>
                                                <div className="text-[10px] font-bold text-slate-400 uppercase">Dimensions</div>
                                                <div className="text-xs font-bold text-slate-700">{dims.x} × {dims.y} × {dims.z} mm</div>
                                            </div>
                                            <div>
                                                <div className="text-[10px] font-bold text-slate-400 uppercase">Suggested Box</div>
                                                <div className="text-xs font-bold text-indigo-600">
                                                    {(parseFloat(dims.x)+10).toFixed(0)} × {(parseFloat(dims.y)+10).toFixed(0)} × {(parseFloat(dims.z)+10).toFixed(0)} mm
                                                </div>
                                            </div>
                                        </div>
                                    )}
                                </div>

                                <InputGroup label="Filament Cost" suffix={sym} value={inputs.filamentCost} onChange={v => setInputs({...inputs, filamentCost: v})} />
                                <InputGroup label="Spool Weight" suffix="g" value={inputs.spoolWeight} onChange={v => setInputs({...inputs, spoolWeight: v})} />
                                
                                <InputGroup label="Estimated Weight" suffix="g" value={inputs.printWeight} onChange={v => setInputs({...inputs, printWeight: v})} />
                                
                                <div className="relative">
                                    <InputGroup label="Print Speed" suffix="mm/s" value={inputs.speed} onChange={v => setInputs({...inputs, speed: v})} />
                                    <div className="text-[9px] text-slate-400 mt-1 absolute -bottom-4 left-0 w-full">
                                        *Time est. based on 0.4mm nozzle / 0.2mm layer
                                    </div>
                                </div>
                                
                                <div className="grid grid-cols-2 gap-4 md:col-span-2">
                                    <InputGroup label="Est. Hours" suffix="h" value={inputs.printTimeHrs} onChange={v => setInputs({...inputs, printTimeHrs: v})} />
                                    <InputGroup label="Minutes" suffix="m" value={inputs.printTimeMin} onChange={v => setInputs({...inputs, printTimeMin: v})} />
                                </div>
                            </div>
                        </section>

                        {/* Electricity Section */}
                        <section className="bg-white rounded-2xl p-6 shadow-sm border border-slate-100">
                             <h2 className="font-bold text-slate-800 flex items-center gap-2 mb-6">
                                <span className="w-1.5 h-1.5 rounded-full bg-amber-500"></span> 2. Power Consumption
                            </h2>
                             <div className="grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-6">
                                <InputGroup label="Printer Wattage" suffix="W" value={inputs.watts} onChange={v => setInputs({...inputs, watts: v})} />
                                <InputGroup label="Electricity Rate" suffix="/kWh" value={inputs.kwhRate} onChange={v => setInputs({...inputs, kwhRate: v})} />
                             </div>
                        </section>

                        {/* Advanced / Overhead Section */}
                        <section className="bg-white rounded-2xl p-6 shadow-sm border border-slate-100">
                            <div className="flex justify-between items-center mb-6 cursor-pointer no-print" onClick={() => setShowAdvanced(!showAdvanced)}>
                                <h2 className="font-bold text-slate-800 flex items-center gap-2">
                                    <span className="w-1.5 h-1.5 rounded-full bg-cyan-500"></span> 3. Business & Extras
                                </h2>
                                <i className={`fas fa-chevron-down text-slate-300 transition-transform ${showAdvanced ? 'rotate-180' : ''}`}></i>
                            </div>
                            
                            <div className={`grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-6 ${showAdvanced ? 'block' : 'hidden md:grid'}`}>
                                <InputGroup label="Labor Time" suffix="min" value={inputs.laborTime} onChange={v => setInputs({...inputs, laborTime: v})} />
                                <InputGroup label="Labor Rate" suffix="/hr" value={inputs.laborRate} onChange={v => setInputs({...inputs, laborRate: v})} />
                                
                                <InputGroup label="Machine Wear" suffix="/hr" value={inputs.depreciation} onChange={v => setInputs({...inputs, depreciation: v})} />
                                <InputGroup label="Failure Rate" suffix="%" value={inputs.failureRate} onChange={v => setInputs({...inputs, failureRate: v})} />

                                <InputGroup label="Shipping Cost" suffix={sym} value={inputs.shipping} onChange={v => setInputs({...inputs, shipping: v})} />
                                <InputGroup label="Handling Fee" suffix={sym} value={inputs.handling} onChange={v => setInputs({...inputs, handling: v})} />
                            </div>
                        </section>
                    </div>

                    {/* RIGHT: Results / Invoice (THE ONLY PART THAT PRINTS) */}
                    <div className="lg:col-span-1 result-column" id="quote-invoice">
                        <div className="bg-white rounded-2xl shadow-xl shadow-slate-200/60 overflow-hidden border border-slate-100 print-container sticky top-6">
                            
                            {/* Total Banner */}
                            <div className="bg-slate-900 p-8 text-center relative overflow-hidden">
                                <div className="absolute top-0 left-0 w-full h-1.5 bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500"></div>
                                
                                {/* Print-Only Header */}
                                <div className="hidden print-only mb-4">
                                    <h1 className="text-3xl font-bold text-slate-900">Print Quotation</h1>
                                    <p className="text-sm text-slate-500">Project: {inputs.partName}</p>
                                    <div className="border-b border-slate-200 my-4"></div>
                                </div>

                                <h3 className="text-slate-400 text-[10px] font-bold uppercase tracking-widest mb-2 no-print">Estimated Quote</h3>
                                <div className="text-5xl font-bold text-white tracking-tight print:text-black print:text-6xl">{sym}{data.final.toFixed(2)}</div>
                                <div className="text-xs font-medium text-emerald-400 mt-3 bg-emerald-400/10 inline-block px-3 py-1 rounded-full no-print">
                                    Profit: +{sym}{data.profit.toFixed(2)}
                                </div>
                            </div>

                            {/* Visualization (Hidden in Print) */}
                            <div className="p-6 border-b border-slate-100 relative chart-container">
                                <div className="h-40 w-40 mx-auto">
                                    <canvas ref={chartRef}></canvas>
                                </div>
                                <div className="absolute inset-0 flex items-center justify-center pointer-events-none mt-2">
                                    <div className="text-center">
                                        <div className="text-[10px] text-slate-400 uppercase font-bold">Base</div>
                                        <div className="text-sm font-bold text-slate-700">{sym}{data.mfgCost.toFixed(2)}</div>
                                    </div>
                                </div>
                            </div>

                            {/* Bill Breakdown */}
                            <div className="p-6 space-y-3">
                                <div className="flex justify-between text-sm items-center">
                                    <span className="text-slate-500 flex items-center gap-2"><span className="w-2 h-2 rounded-full bg-indigo-600 no-print"></span> Material Cost</span>
                                    <span className="font-medium text-slate-700">{sym}{data.mat.toFixed(2)}</span>
                                </div>
                                <div className="flex justify-between text-sm items-center">
                                    <span className="text-slate-500 flex items-center gap-2"><span className="w-2 h-2 rounded-full bg-amber-500 no-print"></span> Electricity</span>
                                    <span className="font-medium text-slate-700">{sym}{data.pwr.toFixed(2)}</span>
                                </div>
                                <div className="flex justify-between text-sm items-center">
                                    <span className="text-slate-500 flex items-center gap-2"><span className="w-2 h-2 rounded-full bg-cyan-500 no-print"></span> Labor</span>
                                    <span className="font-medium text-slate-700">{sym}{data.lab.toFixed(2)}</span>
                                </div>
                                <div className="flex justify-between text-sm items-center">
                                    <span className="text-slate-500 flex items-center gap-2"><span className="w-2 h-2 rounded-full bg-violet-500 no-print"></span> Machine Wear</span>
                                    <span className="font-medium text-slate-700">{sym}{data.dep.toFixed(2)}</span>
                                </div>
                                {data.failCost > 0 && (
                                <div className="flex justify-between text-sm items-center">
                                    <span className="text-slate-500 flex items-center gap-2"><span className="w-2 h-2 rounded-full bg-red-500 no-print"></span> Failure Buffer</span>
                                    <span className="font-medium text-slate-700">{sym}{data.failCost.toFixed(2)}</span>
                                </div>
                                )}
                                <div className="flex justify-between text-sm items-center pt-2 border-t border-slate-50">
                                    <span className="text-slate-500 flex items-center gap-2">Shipping & Handling</span>
                                    <span className="font-medium text-slate-700">{sym}{data.extras.toFixed(2)}</span>
                                </div>
                                
                                <div className="my-4 border-t border-slate-100"></div>
                                
                                <div className="space-y-2 no-print">
                                    <div className="flex justify-between items-center">
                                        <span className="text-xs font-bold text-slate-400 uppercase">Markup ({inputs.customMargin}%)</span>
                                        <input 
                                            type="range" min="0" max="300" value={inputs.customMargin} 
                                            onChange={e => setInputs({...inputs, customMargin: parseInt(e.target.value)})}
                                            className="w-24 h-1 bg-slate-200 rounded-lg appearance-none cursor-pointer"
                                        />
                                    </div>
                                    <div className="grid grid-cols-2 gap-2 mt-4">
                                        <PriceOption 
                                            title="Fair" margin={50} symbol={sym}
                                            price={(data.mfgCost * 1.5 + data.extras).toFixed(2)} 
                                            onClick={() => setInputs({...inputs, customMargin: 50})}
                                            selected={inputs.customMargin === 50}
                                        />
                                        <PriceOption 
                                            title="Retail" margin={100} symbol={sym}
                                            price={(data.mfgCost * 2.0 + data.extras).toFixed(2)} 
                                            onClick={() => setInputs({...inputs, customMargin: 100})}
                                            selected={inputs.customMargin === 100}
                                        />
                                    </div>
                                </div>
                                
                                <div className="hidden print-only mt-8 pt-4 border-t border-slate-300 text-center text-xs text-slate-400">
                                    Generated by PrintQuote • Valid for 30 days
                                </div>
                            </div>

                            {/* Footer Actions */}
                            <div className="bg-slate-50 p-4 border-t border-slate-100 text-center no-print">
                                <button 
                                    onClick={() => {
                                        const json = JSON.stringify(inputs);
                                        const url = window.location.href.split('#')[0] + '#' + btoa(json);
                                        navigator.clipboard.writeText(url);
                                        alert("Settings URL copied to clipboard!");
                                    }}
                                    className="text-indigo-600 text-xs font-bold uppercase tracking-wide hover:underline"
                                >
                                    <i className="fas fa-link mr-1"></i> Copy Share Link
                                </button>
                            </div>

                        </div>
                    </div>

                </div>
            </div>
        );
    };

    const root = ReactDOM.createRoot(document.getElementById('root'));
    root.render(<App />);
</script>
</body>
</html>

