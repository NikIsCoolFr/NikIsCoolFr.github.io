<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Print Cost Calculator</title>
    
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
    
    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Font Awesome -->
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">

    <style>
        body {
            background-color: #111827; /* Gray 900 */
            font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
            color: #f3f4f6;
        }
        
        /* Input Styling */
        .custom-input {
            background: #1f2937; /* Gray 800 */
            border: 1px solid #374151; /* Gray 700 */
            color: white;
            transition: all 0.2s;
        }
        .custom-input:focus {
            border-color: #3b82f6;
            ring: 2px;
            outline: none;
        }

        /* Remove spinners from number inputs */
        input[type=number]::-webkit-inner-spin-button, 
        input[type=number]::-webkit-outer-spin-button { 
            -webkit-appearance: none; 
            margin: 0; 
        }

        .card-shadow {
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.5), 0 2px 4px -1px rgba(0, 0, 0, 0.3);
        }
    </style>
</head>
<body>

    <div id="root"></div>

    <script type="text/babel">
        const { useState } = React;

        // --- REUSABLE COMPONENTS ---
        
        const InputGroup = ({ label, icon, children, subtext }) => (
            <div className="mb-5">
                <label className="block text-gray-400 text-xs font-bold uppercase tracking-wider mb-2 flex items-center gap-2">
                    <i className={`fas ${icon} w-4 text-center text-blue-500`}></i> {label}
                </label>
                {children}
                {subtext && <p className="text-xs text-gray-500 mt-1 ml-1">{subtext}</p>}
            </div>
        );

        const NumberInput = ({ value, onChange, suffix, prefix, step = 1, min = 0 }) => (
            <div className="flex relative rounded-md shadow-sm">
                {prefix && <span className="inline-flex items-center px-3 rounded-l-md border border-r-0 border-gray-600 bg-gray-700 text-gray-300 text-sm">{prefix}</span>}
                <input 
                    type="number" 
                    value={value} 
                    onChange={(e) => onChange(e.target.value === '' ? 0 : parseFloat(e.target.value))}
                    className={`flex-1 block w-full min-w-0 custom-input py-2 px-3 sm:text-sm ${prefix ? '' : 'rounded-l-md'} ${suffix ? '' : 'rounded-r-md'}`}
                    step={step}
                    min={min}
                    onFocus={(e) => e.target.select()}
                />
                {suffix && <span className="inline-flex items-center px-3 rounded-r-md border border-l-0 border-gray-600 bg-gray-700 text-gray-300 text-sm">{suffix}</span>}
            </div>
        );

        // --- MAIN APP ---

        const App = () => {
            // State
            const [filamentCost, setFilamentCost] = useState(20.00);
            const [spoolWeight, setSpoolWeight] = useState(1000);
            const [printWeight, setPrintWeight] = useState(50);
            const [printTimeHours, setPrintTimeHours] = useState(4);
            const [printTimeMinutes, setPrintTimeMinutes] = useState(30);
            const [watts, setWatts] = useState(150);
            const [kwhCost, setKwhCost] = useState(0.15);
            
            const [showAdvanced, setShowAdvanced] = useState(false);
            const [laborTime, setLaborTime] = useState(0);
            const [laborRate, setLaborRate] = useState(15.00);
            const [depreciationPerHr, setDepreciationPerHr] = useState(0.10);
            const [failureRate, setFailureRate] = useState(0);
            const [markup, setMarkup] = useState(0);
            const [shipping, setShipping] = useState(0);
            const [handling, setHandling] = useState(0);

            const currency = "$";

            // Calculations
            const calculate = () => {
                const materialCost = (filamentCost / spoolWeight) * printWeight;
                
                const totalHours = parseFloat(printTimeHours) + (parseFloat(printTimeMinutes) / 60);
                const energyCost = (watts / 1000) * totalHours * kwhCost;
                
                const laborCost = (laborTime / 60) * laborRate;
                const depreciationCost = totalHours * depreciationPerHr;
                
                const baseCost = materialCost + energyCost + laborCost + depreciationCost;
                const failureCost = baseCost * (failureRate / 100);
                const costWithFailure = baseCost + failureCost;
                
                const profit = costWithFailure * (markup / 100);
                const extras = parseFloat(shipping) + parseFloat(handling);
                
                const total = costWithFailure + profit + extras;

                return {
                    material: materialCost,
                    energy: energyCost,
                    ops: laborCost + depreciationCost,
                    failure: failureCost,
                    profit: profit,
                    extras: extras,
                    total: total,
                    raw: baseCost
                };
            };

            const costs = calculate();

            return (
                <div className="min-h-screen p-4 md:p-8 max-w-7xl mx-auto">
                    
                    {/* Header */}
                    <div className="flex flex-col md:flex-row justify-between items-center mb-8 border-b border-gray-700 pb-6">
                        <div className="flex items-center gap-3 mb-4 md:mb-0">
                            <div className="bg-blue-600 p-2 rounded-lg">
                                <i className="fas fa-cube text-white text-xl"></i>
                            </div>
                            <div>
                                <h1 className="text-2xl font-bold text-white">3D Print Calculator</h1>
                                <p className="text-gray-400 text-sm">Standard Utility</p>
                            </div>
                        </div>
                        <button 
                            onClick={() => setShowAdvanced(!showAdvanced)}
                            className={`px-4 py-2 rounded-md text-sm font-medium transition-colors border ${showAdvanced ? 'bg-blue-600 border-blue-600 text-white' : 'bg-gray-800 border-gray-600 text-gray-300 hover:bg-gray-700'}`}
                        >
                            <i className={`fas ${showAdvanced ? 'fa-check-square' : 'fa-square'} mr-2`}></i>
                            Advanced Mode
                        </button>
                    </div>

                    <div className="grid grid-cols-1 lg:grid-cols-12 gap-8">
                        
                        {/* LEFT COLUMN: Inputs */}
                        <div className="lg:col-span-7 space-y-6">
                            
                            {/* Material Section */}
                            <div className="bg-gray-800 p-6 rounded-xl card-shadow border border-gray-700">
                                <h2 className="text-lg font-semibold text-white mb-4 pb-2 border-b border-gray-700">1. Material & Time</h2>
                                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                                    <div className="space-y-4">
                                        <InputGroup label="Spool Cost" icon="fa-tag">
                                            <NumberInput value={filamentCost} onChange={setFilamentCost} prefix={currency} step="0.01" />
                                        </InputGroup>
                                        <InputGroup label="Spool Weight" icon="fa-weight-hanging">
                                            <NumberInput value={spoolWeight} onChange={setSpoolWeight} suffix="g" step="100" />
                                        </InputGroup>
                                    </div>
                                    <div className="space-y-4">
                                        <InputGroup label="Print Weight" icon="fa-layer-group">
                                            <NumberInput value={printWeight} onChange={setPrintWeight} suffix="g" />
                                        </InputGroup>
                                        <div className="grid grid-cols-2 gap-3">
                                            <InputGroup label="Hours" icon="fa-clock">
                                                <NumberInput value={printTimeHours} onChange={setPrintTimeHours} suffix="h" />
                                            </InputGroup>
                                            <InputGroup label="Minutes" icon="fa-stopwatch">
                                                <NumberInput value={printTimeMinutes} onChange={setPrintTimeMinutes} suffix="m" />
                                            </InputGroup>
                                        </div>
                                    </div>
                                </div>
                            </div>

                            {/* Energy Section */}
                            <div className="bg-gray-800 p-6 rounded-xl card-shadow border border-gray-700">
                                <h2 className="text-lg font-semibold text-white mb-4 pb-2 border-b border-gray-700">2. Electricity</h2>
                                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                                    <InputGroup label="Printer Wattage" icon="fa-bolt" subtext="Avg: 150W">
                                        <NumberInput value={watts} onChange={setWatts} suffix="W" step="10" />
                                    </InputGroup>
                                    <InputGroup label="Electricity Rate" icon="fa-plug" subtext="Cost per kWh">
                                        <NumberInput value={kwhCost} onChange={setKwhCost} prefix={currency} suffix="/kWh" step="0.01" />
                                    </InputGroup>
                                </div>
                            </div>

                            {/* Advanced Section */}
                            {showAdvanced && (
                                <div className="bg-gray-800 p-6 rounded-xl card-shadow border border-gray-700 animate-fade-in">
                                    <h2 className="text-lg font-semibold text-white mb-4 pb-2 border-b border-gray-700">3. Business & Extras</h2>
                                    
                                    <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mb-6">
                                        <InputGroup label="Labor Time" icon="fa-user-clock">
                                            <NumberInput value={laborTime} onChange={setLaborTime} suffix="min" />
                                        </InputGroup>
                                        <InputGroup label="Labor Rate" icon="fa-user-tag">
                                            <NumberInput value={laborRate} onChange={setLaborRate} prefix={currency} suffix="/hr" />
                                        </InputGroup>
                                        <InputGroup label="Machine Wear" icon="fa-wrench">
                                            <NumberInput value={depreciationPerHr} onChange={setDepreciationPerHr} prefix={currency} suffix="/hr" step="0.05" />
                                        </InputGroup>
                                        <InputGroup label="Failure Rate" icon="fa-exclamation-triangle">
                                            <NumberInput value={failureRate} onChange={setFailureRate} suffix="%" />
                                        </InputGroup>
                                    </div>

                                    <div className="border-t border-gray-700 pt-4 grid grid-cols-1 md:grid-cols-3 gap-4">
                                        <InputGroup label="Shipping" icon="fa-truck">
                                            <NumberInput value={shipping} onChange={setShipping} prefix={currency} step="0.50" />
                                        </InputGroup>
                                        <InputGroup label="Handling/Fees" icon="fa-box-open">
                                            <NumberInput value={handling} onChange={setHandling} prefix={currency} step="0.50" />
                                        </InputGroup>
                                        <div className="bg-gray-700/50 p-3 rounded-lg">
                                            <label className="block text-gray-400 text-xs font-bold uppercase mb-2">Profit Markup: {markup}%</label>
                                            <input type="range" min="0" max="200" value={markup} onChange={e => setMarkup(parseInt(e.target.value))} className="w-full h-2 bg-gray-600 rounded-lg appearance-none cursor-pointer" />
                                        </div>
                                    </div>
                                </div>
                            )}

                        </div>

                        {/* RIGHT COLUMN: Results */}
                        <div className="lg:col-span-5 space-y-6">
                            
                            {/* Main Price Card */}
                            <div className="bg-gray-800 p-8 rounded-xl card-shadow border border-gray-700 text-center relative overflow-hidden">
                                <div className="absolute top-0 left-0 w-full h-1 bg-gradient-to-r from-blue-500 to-purple-600"></div>
                                <h3 className="text-gray-400 text-sm font-bold uppercase tracking-widest mb-2">Estimated Total</h3>
                                <div className="text-5xl font-bold text-white mb-2">
                                    {currency}{costs.total.toFixed(2)}
                                </div>
                                <div className="flex justify-center gap-4 text-xs text-gray-400">
                                    <span>Raw Cost: {currency}{costs.raw.toFixed(2)}</span>
                                    {costs.profit > 0 && <span className="text-green-400">Profit: +{currency}{costs.profit.toFixed(2)}</span>}
                                </div>
                            </div>

                            {/* Breakdown */}
                            <div className="bg-gray-800 p-6 rounded-xl card-shadow border border-gray-700">
                                <h3 className="text-white font-bold mb-4">Cost Breakdown</h3>
                                <div className="space-y-4">
                                    {[
                                        { label: "Material", val: costs.material, color: "bg-blue-500", icon: "fa-cube" },
                                        { label: "Electricity", val: costs.energy, color: "bg-yellow-500", icon: "fa-bolt" },
                                        { label: "Ops & Labor", val: costs.ops, color: "bg-purple-500", icon: "fa-tools", hide: !showAdvanced },
                                        { label: "Extras", val: costs.extras, color: "bg-green-500", icon: "fa-truck", hide: !showAdvanced },
                                        { label: "Profit", val: costs.profit, color: "bg-emerald-500", icon: "fa-coins", hide: !showAdvanced }
                                    ].map((item, i) => {
                                        if (item.hide && item.val === 0) return null;
                                        const pct = Math.min((item.val / costs.total) * 100, 100) || 0;
                                        return (
                                            <div key={i}>
                                                <div className="flex justify-between text-sm mb-1 text-gray-300">
                                                    <span className="flex items-center gap-2"><i className={`fas ${item.icon} w-4 text-center opacity-50`}></i> {item.label}</span>
                                                    <span>{currency}{item.val.toFixed(2)}</span>
                                                </div>
                                                <div className="w-full bg-gray-700 rounded-full h-2">
                                                    <div className={`${item.color} h-2 rounded-full`} style={{ width: `${pct}%` }}></div>
                                                </div>
                                            </div>
                                        )
                                    })}
                                </div>
                            </div>

                            {/* Actions */}
                            <div className="grid grid-cols-2 gap-4">
                                <button onClick={() => window.print()} className="bg-gray-700 hover:bg-gray-600 text-white py-3 rounded-lg font-medium transition-colors flex items-center justify-center gap-2">
                                    <i className="fas fa-print"></i> Print
                                </button>
                                <button onClick={() => {
                                    const text = `Quote: ${currency}${costs.total.toFixed(2)}\nMaterial: ${currency}${costs.material.toFixed(2)}\nEnergy: ${currency}${costs.energy.toFixed(2)}`;
                                    navigator.clipboard.writeText(text);
                                    alert('Copied to clipboard');
                                }} className="bg-blue-600 hover:bg-blue-700 text-white py-3 rounded-lg font-medium transition-colors flex items-center justify-center gap-2">
                                    <i className="fas fa-copy"></i> Copy
                                </button>
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
