<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Interactive 3D Periodic Table</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script async src="https://unpkg.com/es-module-shims@1.6.3/dist/es-module-shims.js"></script>
    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.157.0/build/three.module.js",
            "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.157.0/examples/jsm/"
        }
    }
    </script>
    <style>
        :root {
            --alkali-metal: #3b82f6;
            --alkaline-earth-metal: #ef4444;
            --lanthanide: #8b5cf6;
            --actinide: #d946ef;
            --transition-metal: #f59e0b;
            --post-transition-metal: #64748b;
            --metalloid: #10b981;
            --diatomic-nonmetal: #0ea5e9;
            --polyatomic-nonmetal: #0284c7;
            --noble-gas: #6366f1;
            --unknown: #a3a3a3;
        }
        body {
            font-family: 'Inter', sans-serif;
            background-color: #0f172a;
            color: white;
            overflow-y: auto; 
            background-image: radial-gradient(circle at 50% 50%, #1e293b 0%, #0f172a 100%);
            min-height: 100vh;
        }
        
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: rgba(255,255,255,0.05); }
        ::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.2); border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: rgba(255,255,255,0.3); }

        .periodic-table {
            display: grid;
            grid-template-columns: repeat(18, minmax(0, 1fr));
            gap: 6px; 
            width: 95%;
            margin: 0 auto;
            padding-bottom: 60px; 
            max-width: 1800px;
        }
        
        .element-cell {
            aspect-ratio: 1 / 1; 
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            align-items: center;
            border-radius: 6px;
            cursor: pointer;
            transition: all 0.2s cubic-bezier(0.25, 0.46, 0.45, 0.94);
            border: 1px solid rgba(255, 255, 255, 0.1);
            padding: 4px;
            position: relative;
            color: white;
            text-shadow: 0 1px 2px rgba(0,0,0,0.8);
            background-clip: padding-box;
            opacity: 1;
        }
        
        .element-cell:hover {
            transform: scale(1.4);
            box-shadow: 0 15px 35px rgba(0, 0, 0, 0.8);
            z-index: 100;
            border-color: rgba(255,255,255,0.8);
        }
        
        .element-cell.dimmed {
            opacity: 0.1;
            transform: scale(0.95);
            filter: grayscale(0.8);
        }

        .element-cell.selected {
            border: 2px solid #ffffff;
            transform: scale(1.1);
            z-index: 50;
            box-shadow: 0 0 20px rgba(255,255,255,0.3);
        }
        
        /* Typography scale based on cell size */
        .element-number { font-size: clamp(0.5rem, 1vw, 0.8rem); align-self: flex-start; margin-left: 2px; opacity: 0.7; }
        .element-symbol { font-size: clamp(0.8rem, 2.5vw, 1.8rem); font-weight: 800; letter-spacing: -0.5px; }
        /* Updated to allow text wrapping and prevent ellipsis */
        .element-name { 
            font-size: clamp(0.3rem, 0.7vw, 0.6rem); 
            white-space: normal; 
            overflow: visible; 
            width: 100%; 
            text-align: center; 
            font-weight: 500; 
            opacity: 0.9; 
            line-height: 0.95; 
            word-break: break-word;
        }
        .element-mass { font-size: clamp(0.3rem, 0.7vw, 0.55rem); opacity: 0.6; margin-bottom: 1px; }

        /* Gradients */
        .alkali-metal { background: linear-gradient(135deg, var(--alkali-metal), #1d4ed8); }
        .alkaline-earth-metal { background: linear-gradient(135deg, var(--alkaline-earth-metal), #b91c1c); }
        .lanthanide { background: linear-gradient(135deg, var(--lanthanide), #6d28d9); }
        .actinide { background: linear-gradient(135deg, var(--actinide), #be185d); }
        .transition-metal { background: linear-gradient(135deg, var(--transition-metal), #b45309); }
        .post-transition-metal { background: linear-gradient(135deg, var(--post-transition-metal), #334155); }
        .metalloid { background: linear-gradient(135deg, var(--metalloid), #047857); }
        .diatomic-nonmetal { background: linear-gradient(135deg, var(--diatomic-nonmetal), #0369a1); }
        .polyatomic-nonmetal { background: linear-gradient(135deg, var(--polyatomic-nonmetal), #1e3a8a); }
        .noble-gas { background: linear-gradient(135deg, var(--noble-gas), #4338ca); }
        .unknown { background: linear-gradient(135deg, var(--unknown), #525252); }
        
        .placeholder-cell {
            display: flex;
            justify-content: center;
            align-items: center;
            font-size: 0.8rem;
            color: rgba(255,255,255,0.2);
            border: 1px dashed rgba(255,255,255,0.1);
            border-radius: 4px;
            aspect-ratio: 1/1;
        }

        #atom-view-container {
            position: fixed;
            inset: 0;
            background-color: rgba(0, 0, 0, 0.6);
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 200;
            opacity: 0;
            pointer-events: none;
            transition: opacity 0.4s cubic-bezier(0.16, 1, 0.3, 1);
            backdrop-filter: blur(12px);
        }
        #atom-view-container.visible { opacity: 1; pointer-events: auto; }
        
        #atom-panel {
            position: absolute;
            top: 2rem;
            left: 2rem;
            background: rgba(15, 23, 42, 0.85);
            padding: 1.5rem;
            border-radius: 1.5rem;
            border: 1px solid rgba(255,255,255,0.1);
            width: 360px;
            box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5);
            backdrop-filter: blur(10px);
        }

        .legend-container {
            display: flex;
            flex-wrap: wrap;
            justify-content: center;
            gap: 8px;
            padding: 10px;
        }
        .legend-item {
            display: flex;
            align-items: center;
            font-size: 0.7rem;
            color: #cbd5e1;
            cursor: pointer;
            padding: 4px 8px;
            border-radius: 999px;
            border: 1px solid transparent;
            transition: background 0.2s;
        }
        .legend-item:hover { background: rgba(255,255,255,0.1); border-color: rgba(255,255,255,0.2); }
        .legend-item.active { background: rgba(255,255,255,0.15); border-color: white; color: white; font-weight: 600; }
        .legend-color { width: 10px; height: 10px; border-radius: 50%; margin-right: 6px; box-shadow: 0 0 5px rgba(0,0,0,0.3); }

        #search-input {
            background: rgba(255,255,255,0.1);
            border: 1px solid rgba(255,255,255,0.2);
            color: white;
            border-radius: 0.5rem;
            padding: 0.25rem 0.75rem;
            font-size: 0.9rem;
            outline: none;
            width: 200px;
            transition: width 0.3s, background 0.3s;
        }
        #search-input:focus { width: 250px; background: rgba(255,255,255,0.15); border-color: rgba(255,255,255,0.5); }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700;800&display=swap" rel="stylesheet">
</head>
<body class="p-2 min-h-screen flex flex-col">

    <header class="flex-none mb-2 px-4 pt-2">
        <div class="flex justify-between items-center mb-2">
            <h1 class="text-2xl font-bold tracking-tight text-transparent bg-clip-text bg-gradient-to-r from-white to-gray-400">Periodic Table</h1>
            <input type="text" id="search-input" placeholder="Search (e.g. Gold, Au, 79)">
        </div>
        
        <div class="legend-container" id="legend">
            <!-- Generated by JS -->
        </div>
    </header>

    <main class="periodic-table flex-grow" id="periodic-table-container">
        <!-- Elements injected here -->
    </main>

    <div id="atom-view-container">
        <div id="atom-panel" class="text-white transform transition-all duration-300 translate-x-0">
            <div class="flex justify-between items-end border-b border-gray-700/50 pb-4 mb-4">
                <div>
                    <h2 id="info-element-name" class="text-3xl font-bold tracking-tight">Element</h2>
                    <div class="flex items-center space-x-2 mt-1">
                         <span class="text-xs uppercase tracking-wider text-gray-400 bg-gray-800 px-2 py-0.5 rounded" id="info-category">Category</span>
                         <span class="text-sm text-gray-400">Mass: <span id="info-mass" class="text-gray-200"></span></span>
                    </div>
                </div>
                <p id="info-element-symbol" class="text-6xl font-extrabold text-white opacity-20 select-none"></p>
            </div>
            
            <div class="grid grid-cols-3 gap-3 text-center mb-6">
                <div class="bg-gray-800/50 rounded-xl p-2 border border-white/5">
                    <div class="text-[10px] uppercase tracking-widest text-gray-400 mb-1">Protons</div>
                    <div id="info-protons" class="text-xl font-mono text-blue-400 font-bold">0</div>
                </div>
                <div class="bg-gray-800/50 rounded-xl p-2 border border-white/5">
                    <div class="text-[10px] uppercase tracking-widest text-gray-400 mb-1">Neutrons</div>
                    <div id="info-neutrons" class="text-xl font-mono text-red-400 font-bold">0</div>
                </div>
                <div class="bg-gray-800/50 rounded-xl p-2 border border-white/5">
                    <div class="text-[10px] uppercase tracking-widest text-gray-400 mb-1">Electrons</div>
                    <div id="info-electrons" class="text-xl font-mono text-yellow-400 font-bold">0</div>
                </div>
            </div>

            <div class="space-y-3">
                <div class="flex space-x-2">
                    <button id="gemini-fact-btn" class="flex-1 bg-gradient-to-r from-blue-600 to-blue-700 hover:from-blue-500 hover:to-blue-600 text-white font-semibold py-2.5 px-3 rounded-lg text-sm transition shadow-lg shadow-blue-900/30 flex items-center justify-center gap-2">
                        <span>🎲</span> Fun Fact
                    </button>
                    <button id="gemini-explain-btn" class="flex-1 bg-gradient-to-r from-purple-600 to-purple-700 hover:from-purple-500 hover:to-purple-600 text-white font-semibold py-2.5 px-3 rounded-lg text-sm transition shadow-lg shadow-purple-900/30 flex items-center justify-center gap-2">
                        <span>🎓</span> Explain
                    </button>
                </div>
                
                <div id="gemini-loading" class="hidden">
                    <div class="flex items-center justify-center space-x-2 text-gray-400 text-sm py-2">
                        <div class="w-2 h-2 bg-blue-500 rounded-full animate-bounce" style="animation-delay: 0s"></div>
                        <div class="w-2 h-2 bg-purple-500 rounded-full animate-bounce" style="animation-delay: 0.1s"></div>
                        <div class="w-2 h-2 bg-pink-500 rounded-full animate-bounce" style="animation-delay: 0.2s"></div>
                        <span>AI is thinking...</span>
                    </div>
                </div>
                
                <div id="gemini-result" class="bg-gray-900/60 border border-white/10 rounded-lg p-4 text-sm text-gray-200 leading-relaxed hidden max-h-40 overflow-y-auto">
                </div>
            </div>
            
             <div class="mt-4 text-[10px] text-center text-gray-500">
                Left Click: Rotate • Right Click: Pan • Scroll: Zoom
            </div>
        </div>
        
        <canvas id="atom-canvas"></canvas>
        
        <button id="close-button" class="absolute top-6 right-6 p-3 bg-white/10 hover:bg-white/20 rounded-full backdrop-blur transition">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-white" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
            </svg>
        </button>
    </div>


    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

        // --- DATA ---
        const elementsData = [
            { atomicNumber: 1, symbol: "H", name: "Hydrogen", atomicMass: "1.008", category: "diatomic-nonmetal", period: 1, group: 1 },
            { atomicNumber: 2, symbol: "He", name: "Helium", atomicMass: "4.0026", category: "noble-gas", period: 1, group: 18 },
            { atomicNumber: 3, symbol: "Li", name: "Lithium", atomicMass: "6.94", category: "alkali-metal", period: 2, group: 1 },
            { atomicNumber: 4, symbol: "Be", name: "Beryllium", atomicMass: "9.0122", category: "alkaline-earth-metal", period: 2, group: 2 },
            { atomicNumber: 5, symbol: "B", name: "Boron", atomicMass: "10.81", category: "metalloid", period: 2, group: 13 },
            { atomicNumber: 6, symbol: "C", name: "Carbon", atomicMass: "12.011", category: "polyatomic-nonmetal", period: 2, group: 14 },
            { atomicNumber: 7, symbol: "N", name: "Nitrogen", atomicMass: "14.007", category: "diatomic-nonmetal", period: 2, group: 15 },
            { atomicNumber: 8, symbol: "O", name: "Oxygen", atomicMass: "15.999", category: "diatomic-nonmetal", period: 2, group: 16 },
            { atomicNumber: 9, symbol: "F", name: "Fluorine", atomicMass: "18.998", category: "diatomic-nonmetal", period: 2, group: 17 },
            { atomicNumber: 10, symbol: "Ne", name: "Neon", atomicMass: "20.180", category: "noble-gas", period: 2, group: 18 },
            { atomicNumber: 11, symbol: "Na", name: "Sodium", atomicMass: "22.990", category: "alkali-metal", period: 3, group: 1 },
            { atomicNumber: 12, symbol: "Mg", name: "Magnesium", atomicMass: "24.305", category: "alkaline-earth-metal", period: 3, group: 2 },
            { atomicNumber: 13, symbol: "Al", name: "Aluminum", atomicMass: "26.982", category: "post-transition-metal", period: 3, group: 13 },
            { atomicNumber: 14, symbol: "Si", name: "Silicon", atomicMass: "28.085", category: "metalloid", period: 3, group: 14 },
            { atomicNumber: 15, symbol: "P", name: "Phosphorus", atomicMass: "30.974", category: "polyatomic-nonmetal", period: 3, group: 15 },
            { atomicNumber: 16, symbol: "S", name: "Sulfur", atomicMass: "32.06", category: "polyatomic-nonmetal", period: 3, group: 16 },
            { atomicNumber: 17, symbol: "Cl", name: "Chlorine", atomicMass: "35.45", category: "diatomic-nonmetal", period: 3, group: 17 },
            { atomicNumber: 18, symbol: "Ar", name: "Argon", atomicMass: "39.948", category: "noble-gas", period: 3, group: 18 },
            { atomicNumber: 19, symbol: "K", name: "Potassium", atomicMass: "39.098", category: "alkali-metal", period: 4, group: 1 },
            { atomicNumber: 20, symbol: "Ca", name: "Calcium", atomicMass: "40.078", category: "alkaline-earth-metal", period: 4, group: 2 },
            { atomicNumber: 21, symbol: "Sc", name: "Scandium", atomicMass: "44.956", category: "transition-metal", period: 4, group: 3 },
            { atomicNumber: 22, symbol: "Ti", name: "Titanium", atomicMass: "47.867", category: "transition-metal", period: 4, group: 4 },
            { atomicNumber: 23, symbol: "V", name: "Vanadium", atomicMass: "50.942", category: "transition-metal", period: 4, group: 5 },
            { atomicNumber: 24, symbol: "Cr", name: "Chromium", atomicMass: "51.996", category: "transition-metal", period: 4, group: 6 },
            { atomicNumber: 25, symbol: "Mn", name: "Manganese", atomicMass: "54.938", category: "transition-metal", period: 4, group: 7 },
            { atomicNumber: 26, symbol: "Fe", name: "Iron", atomicMass: "55.845", category: "transition-metal", period: 4, group: 8 },
            { atomicNumber: 27, symbol: "Co", name: "Cobalt", atomicMass: "58.933", category: "transition-metal", period: 4, group: 9 },
            { atomicNumber: 28, symbol: "Ni", name: "Nickel", atomicMass: "58.693", category: "transition-metal", period: 4, group: 10 },
            { atomicNumber: 29, symbol: "Cu", name: "Copper", atomicMass: "63.546", category: "transition-metal", period: 4, group: 11 },
            { atomicNumber: 30, symbol: "Zn", name: "Zinc", atomicMass: "65.38", category: "transition-metal", period: 4, group: 12 },
            { atomicNumber: 31, symbol: "Ga", name: "Gallium", atomicMass: "69.723", category: "post-transition-metal", period: 4, group: 13 },
            { atomicNumber: 32, symbol: "Ge", name: "Germanium", atomicMass: "72.630", category: "metalloid", period: 4, group: 14 },
            { atomicNumber: 33, symbol: "As", name: "Arsenic", atomicMass: "74.922", category: "metalloid", period: 4, group: 15 },
            { atomicNumber: 34, symbol: "Se", name: "Selenium", atomicMass: "78.972", category: "polyatomic-nonmetal", period: 4, group: 16 },
            { atomicNumber: 35, symbol: "Br", name: "Bromine", atomicMass: "79.904", category: "diatomic-nonmetal", period: 4, group: 17 },
            { atomicNumber: 36, symbol: "Kr", name: "Krypton", atomicMass: "83.798", category: "noble-gas", period: 4, group: 18 },
            { atomicNumber: 37, symbol: "Rb", name: "Rubidium", atomicMass: "85.468", category: "alkali-metal", period: 5, group: 1 },
            { atomicNumber: 38, symbol: "Sr", name: "Strontium", atomicMass: "87.62", category: "alkaline-earth-metal", period: 5, group: 2 },
            { atomicNumber: 39, symbol: "Y", name: "Yttrium", atomicMass: "88.906", category: "transition-metal", period: 5, group: 3 },
            { atomicNumber: 40, symbol: "Zr", name: "Zirconium", atomicMass: "91.224", category: "transition-metal", period: 5, group: 4 },
            { atomicNumber: 41, symbol: "Nb", name: "Niobium", atomicMass: "92.906", category: "transition-metal", period: 5, group: 5 },
            { atomicNumber: 42, symbol: "Mo", name: "Molybdenum", atomicMass: "95.95", category: "transition-metal", period: 5, group: 6 },
            { atomicNumber: 43, symbol: "Tc", name: "Technetium", atomicMass: "98", category: "transition-metal", period: 5, group: 7 },
            { atomicNumber: 44, symbol: "Ru", name: "Ruthenium", atomicMass: "101.07", category: "transition-metal", period: 5, group: 8 },
            { atomicNumber: 45, symbol: "Rh", name: "Rhodium", atomicMass: "102.91", category: "transition-metal", period: 5, group: 9 },
            { atomicNumber: 46, symbol: "Pd", name: "Palladium", atomicMass: "106.42", category: "transition-metal", period: 5, group: 10 },
            { atomicNumber: 47, symbol: "Ag", name: "Silver", atomicMass: "107.87", category: "transition-metal", period: 5, group: 11 },
            { atomicNumber: 48, symbol: "Cd", name: "Cadmium", atomicMass: "112.41", category: "transition-metal", period: 5, group: 12 },
            { atomicNumber: 49, symbol: "In", name: "Indium", atomicMass: "114.82", category: "post-transition-metal", period: 5, group: 13 },
            { atomicNumber: 50, symbol: "Sn", name: "Tin", atomicMass: "118.71", category: "post-transition-metal", period: 5, group: 14 },
            { atomicNumber: 51, symbol: "Sb", name: "Antimony", atomicMass: "121.76", category: "metalloid", period: 5, group: 15 },
            { atomicNumber: 52, symbol: "Te", name: "Tellurium", atomicMass: "127.60", category: "metalloid", period: 5, group: 16 },
            { atomicNumber: 53, symbol: "I", name: "Iodine", atomicMass: "126.90", category: "diatomic-nonmetal", period: 5, group: 17 },
            { atomicNumber: 54, symbol: "Xe", name: "Xenon", atomicMass: "131.29", category: "noble-gas", period: 5, group: 18 },
            { atomicNumber: 55, symbol: "Cs", name: "Cesium", atomicMass: "132.91", category: "alkali-metal", period: 6, group: 1 },
            { atomicNumber: 56, symbol: "Ba", name: "Barium", atomicMass: "137.33", category: "alkaline-earth-metal", period: 6, group: 2 },
            { atomicNumber: 57, symbol: "La", name: "Lanthanum", atomicMass: "138.91", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 58, symbol: "Ce", name: "Cerium", atomicMass: "140.12", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 59, symbol: "Pr", name: "Praseodymium", atomicMass: "140.91", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 60, symbol: "Nd", name: "Neodymium", atomicMass: "144.24", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 61, symbol: "Pm", name: "Promethium", atomicMass: "145", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 62, symbol: "Sm", name: "Samarium", atomicMass: "150.36", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 63, symbol: "Eu", name: "Europium", atomicMass: "151.96", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 64, symbol: "Gd", name: "Gadolinium", atomicMass: "157.25", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 65, symbol: "Tb", name: "Terbium", atomicMass: "158.93", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 66, symbol: "Dy", name: "Dysprosium", atomicMass: "162.50", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 67, symbol: "Ho", name: "Holmium", atomicMass: "164.93", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 68, symbol: "Er", name: "Erbium", atomicMass: "167.26", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 69, symbol: "Tm", name: "Thulium", atomicMass: "168.93", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 70, symbol: "Yb", name: "Ytterbium", atomicMass: "173.05", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 71, symbol: "Lu", name: "Lutetium", atomicMass: "174.97", category: "lanthanide", period: 6, group: 3 },
            { atomicNumber: 72, symbol: "Hf", name: "Hafnium", atomicMass: "178.49", category: "transition-metal", period: 6, group: 4 },
            { atomicNumber: 73, symbol: "Ta", name: "Tantalum", atomicMass: "180.95", category: "transition-metal", period: 6, group: 5 },
            { atomicNumber: 74, symbol: "W", name: "Tungsten", atomicMass: "183.84", category: "transition-metal", period: 6, group: 6 },
            { atomicNumber: 75, symbol: "Re", name: "Rhenium", atomicMass: "186.21", category: "transition-metal", period: 6, group: 7 },
            { atomicNumber: 76, symbol: "Os", name: "Osmium", atomicMass: "190.23", category: "transition-metal", period: 6, group: 8 },
            { atomicNumber: 77, symbol: "Ir", name: "Iridium", atomicMass: "192.22", category: "transition-metal", period: 6, group: 9 },
            { atomicNumber: 78, symbol: "Pt", name: "Platinum", atomicMass: "195.08", category: "transition-metal", period: 6, group: 10 },
            { atomicNumber: 79, symbol: "Au", name: "Gold", atomicMass: "196.97", category: "transition-metal", period: 6, group: 11 },
            { atomicNumber: 80, symbol: "Hg", name: "Mercury", atomicMass: "200.59", category: "transition-metal", period: 6, group: 12 },
            { atomicNumber: 81, symbol: "Tl", name: "Thallium", atomicMass: "204.38", category: "post-transition-metal", period: 6, group: 13 },
            { atomicNumber: 82, symbol: "Pb", name: "Lead", atomicMass: "207.21", category: "post-transition-metal", period: 6, group: 14 },
            { atomicNumber: 83, symbol: "Bi", name: "Bismuth", atomicMass: "208.98", category: "post-transition-metal", period: 6, group: 15 },
            { atomicNumber: 84, symbol: "Po", name: "Polonium", atomicMass: "209", category: "post-transition-metal", period: 6, group: 16 },
            { atomicNumber: 85, symbol: "At", name: "Astatine", atomicMass: "210", category: "metalloid", period: 6, group: 17 },
            { atomicNumber: 86, symbol: "Rn", name: "Radon", atomicMass: "222", category: "noble-gas", period: 6, group: 18 },
            { atomicNumber: 87, symbol: "Fr", name: "Francium", atomicMass: "223", category: "alkali-metal", period: 7, group: 1 },
            { atomicNumber: 88, symbol: "Ra", name: "Radium", atomicMass: "226", category: "alkaline-earth-metal", period: 7, group: 2 },
            { atomicNumber: 89, symbol: "Ac", name: "Actinium", atomicMass: "227", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 90, symbol: "Th", name: "Thorium", atomicMass: "232.04", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 91, symbol: "Pa", name: "Protactinium", atomicMass: "231.04", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 92, symbol: "U", name: "Uranium", atomicMass: "238.03", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 93, symbol: "Np", name: "Neptunium", atomicMass: "237", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 94, symbol: "Pu", name: "Plutonium", atomicMass: "244", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 95, symbol: "Am", name: "Americium", atomicMass: "243", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 96, symbol: "Cm", name: "Curium", atomicMass: "247", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 97, symbol: "Bk", name: "Berkelium", atomicMass: "247", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 98, symbol: "Cf", name: "Californium", atomicMass: "251", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 99, symbol: "Es", name: "Einsteinium", atomicMass: "252", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 100, symbol: "Fm", name: "Fermium", atomicMass: "257", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 101, symbol: "Md", name: "Mendelevium", atomicMass: "258", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 102, symbol: "No", name: "Nobelium", atomicMass: "259", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 103, symbol: "Lr", name: "Lawrencium", atomicMass: "266", category: "actinide", period: 7, group: 3 },
            { atomicNumber: 104, symbol: "Rf", name: "Rutherfordium", atomicMass: "267", category: "transition-metal", period: 7, group: 4 },
            { atomicNumber: 105, symbol: "Db", name: "Dubnium", atomicMass: "268", category: "transition-metal", period: 7, group: 5 },
            { atomicNumber: 106, symbol: "Sg", name: "Seaborgium", atomicMass: "269", category: "transition-metal", period: 7, group: 6 },
            { atomicNumber: 107, symbol: "Bh", name: "Bohrium", atomicMass: "270", category: "transition-metal", period: 7, group: 7 },
            { atomicNumber: 108, symbol: "Hs", name: "Hassium", atomicMass: "277", category: "transition-metal", period: 7, group: 8 },
            { atomicNumber: 109, symbol: "Mt", name: "Meitnerium", atomicMass: "278", category: "unknown", period: 7, group: 9 },
            { atomicNumber: 110, symbol: "Ds", name: "Darmstadtium", atomicMass: "281", category: "unknown", period: 7, group: 10 },
            { atomicNumber: 111, symbol: "Rg", name: "Roentgenium", atomicMass: "282", category: "unknown", period: 7, group: 11 },
            { atomicNumber: 112, symbol: "Cn", name: "Copernicium", atomicMass: "285", category: "transition-metal", period: 7, group: 12 },
            { atomicNumber: 113, symbol: "Nh", name: "Nihonium", atomicMass: "286", category: "unknown", period: 7, group: 13 },
            { atomicNumber: 114, symbol: "Fl", name: "Flerovium", atomicMass: "289", category: "post-transition-metal", period: 7, group: 14 },
            { atomicNumber: 115, symbol: "Mc", name: "Moscovium", atomicMass: "290", category: "unknown", period: 7, group: 15 },
            { atomicNumber: 116, symbol: "Lv", name: "Livermorium", atomicMass: "293", category: "unknown", period: 7, group: 16 },
            { atomicNumber: 117, symbol: "Ts", name: "Tennessine", atomicMass: "294", category: "unknown", period: 7, group: 17 },
            { atomicNumber: 118, symbol: "Og", name: "Oganesson", atomicMass: "294", category: "unknown", period: 7, group: 18 }
        ];

        // --- CATEGORIES & COLORS ---
        const categories = {
            "alkali-metal": { name: "Alkali Metal", color: "#3b82f6" },
            "alkaline-earth-metal": { name: "Alkaline Earth", color: "#ef4444" },
            "transition-metal": { name: "Transition Metal", color: "#f59e0b" },
            "lanthanide": { name: "Lanthanide", color: "#8b5cf6" },
            "actinide": { name: "Actinide", color: "#d946ef" },
            "metalloid": { name: "Metalloid", color: "#10b981" },
            "diatomic-nonmetal": { name: "Nonmetal", color: "#0ea5e9" },
            "polyatomic-nonmetal": { name: "Polyatomic Nonmetal", color: "#0284c7" },
            "noble-gas": { name: "Noble Gas", color: "#6366f1" },
            "post-transition-metal": { name: "Post-Transition", color: "#64748b" },
            "unknown": { name: "Unknown", color: "#a3a3a3" }
        };

        // --- UI STATE ---
        let activeCategory = null;
        let searchTerm = "";

        // --- PERIODIC TABLE SETUP ---
        const tableContainer = document.getElementById('periodic-table-container');
        const legendContainer = document.getElementById('legend');
        const searchInput = document.getElementById('search-input');

        // Generate Legend
        Object.entries(categories).forEach(([key, val]) => {
            if (key === 'polyatomic-nonmetal') return; // Merge visually for simplicity or keep separate
            
            const item = document.createElement('div');
            item.className = `legend-item ${key}`;
            item.innerHTML = `<div class="legend-color" style="background:${val.color}"></div>${val.name}`;
            item.addEventListener('click', () => {
                if (activeCategory === key) {
                    activeCategory = null;
                    item.classList.remove('active');
                } else {
                    activeCategory = key;
                    document.querySelectorAll('.legend-item').forEach(i => i.classList.remove('active'));
                    item.classList.add('active');
                }
                filterTable();
            });
            legendContainer.appendChild(item);
        });

        // Search Logic
        searchInput.addEventListener('input', (e) => {
            searchTerm = e.target.value.toLowerCase();
            filterTable();
        });

        function filterTable() {
            const cells = document.querySelectorAll('.element-cell');
            cells.forEach(cell => {
                const data = JSON.parse(cell.dataset.element);
                const matchesCategory = activeCategory ? data.category === activeCategory : true;
                const matchesSearch = searchTerm === "" || 
                                      data.name.toLowerCase().includes(searchTerm) || 
                                      data.symbol.toLowerCase().includes(searchTerm) ||
                                      data.atomicNumber.toString() === searchTerm;
                
                if (matchesCategory && matchesSearch) {
                    cell.classList.remove('dimmed');
                } else {
                    cell.classList.add('dimmed');
                }
            });
        }
        
        // Add placeholders
        const p1 = document.createElement('div');
        p1.className = 'placeholder-cell';
        p1.textContent = '57-71';
        p1.style.gridRow = 6; p1.style.gridColumn = 3;
        tableContainer.appendChild(p1);
        
        const p2 = document.createElement('div');
        p2.className = 'placeholder-cell';
        p2.textContent = '89-103';
        p2.style.gridRow = 7; p2.style.gridColumn = 3;
        tableContainer.appendChild(p2);

        elementsData.forEach(el => {
            const cell = document.createElement('div');
            cell.className = `element-cell ${el.category}`;
            cell.dataset.element = JSON.stringify(el);
            
            let row = el.period;
            let col = el.group;
            if (el.atomicNumber >= 57 && el.atomicNumber <= 71) { row = 9; col = (el.atomicNumber - 57) + 3; } 
            else if (el.atomicNumber >= 89 && el.atomicNumber <= 103) { row = 10; col = (el.atomicNumber - 89) + 3; }
            
            cell.style.gridColumn = col;
            cell.style.gridRow = row;
            
            cell.addEventListener('click', () => {
                document.querySelectorAll('.element-cell').forEach(c => c.classList.remove('selected'));
                cell.classList.add('selected');
                showAtomViewer(el);
            });

            cell.innerHTML = `
                <div class="element-number">${el.atomicNumber}</div>
                <div class="element-symbol">${el.symbol}</div>
                <div class="element-name">${el.name}</div>
                <div class="element-mass">${el.atomicMass}</div>
            `;
            tableContainer.appendChild(cell);
        });

        // --- 3D ATOM VIEWER ---
        const viewerContainer = document.getElementById('atom-view-container');
        const canvas = document.getElementById('atom-canvas');
        const closeButton = document.getElementById('close-button');

        let scene, camera, renderer, controls;
        let atomGroup;
        const atomShells = [];

        function init3D() {
            scene = new THREE.Scene();
            // Starfield Background
            const starGeometry = new THREE.BufferGeometry();
            const starMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 0.1, transparent: true, opacity: 0.8 });
            const starVertices = [];
            for(let i=0; i<1500; i++) {
                const x = (Math.random() - 0.5) * 200;
                const y = (Math.random() - 0.5) * 200;
                const z = (Math.random() - 0.5) * 200;
                starVertices.push(x,y,z);
            }
            starGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starVertices, 3));
            scene.add(new THREE.Points(starGeometry, starMaterial));

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            renderer = new THREE.WebGLRenderer({ canvas: canvas, antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
            scene.add(ambientLight);
            
            const pointLight = new THREE.PointLight(0xffffff, 1, 100);
            pointLight.position.set(10, 10, 10);
            scene.add(pointLight);
            
            // Cam light
            const camLight = new THREE.PointLight(0xffffff, 0.5);
            camera.add(camLight);
            scene.add(camera);

            controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.05;
            controls.enablePan = true; // Enabled panning

            camera.position.z = 25;
            animate();
        }

        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            
            const time = Date.now() * 0.001;
            
            // Gentle float for the whole atom
            if(atomGroup) {
                atomGroup.rotation.y = Math.sin(time * 0.1) * 0.1;
                atomGroup.rotation.z = Math.cos(time * 0.08) * 0.05;
            }

            atomShells.forEach((shell, i) => {
                shell.electrons.forEach(electronData => {
                    const e = electronData;
                    // Inner shells faster
                    const speedMultiplier = 1 + (4 / (i + 1)); 
                    const angle = e.initialAngle + time * e.speed * speedMultiplier * 0.5;
                    // Corrected to move on XZ plane to match the rings
                    e.mesh.position.x = shell.radius * Math.cos(angle);
                    e.mesh.position.z = shell.radius * Math.sin(angle);
                    e.mesh.position.y = 0;
                });
            });

            renderer.render(scene, camera);
        }

        let currentElement = null;

        function showAtomViewer(element) {
            currentElement = element;
            updateInfoPanel(element);
            createAtomModel(element);
            
            // Reset AI UI
            document.getElementById('gemini-result').classList.add('hidden');
            document.getElementById('gemini-result').innerHTML = '';
            document.getElementById('gemini-loading').classList.add('hidden');

            viewerContainer.classList.add('visible');
            
            // Slight zoom effect on open
            if(atomGroup) {
                atomGroup.scale.set(0,0,0);
                new TWEEN.Tween(atomGroup.scale)
                    .to({ x: 1, y: 1, z: 1 }, 800)
                    .easing(TWEEN.Easing.Elastic.Out)
                    .start();
            }
        }

        function hideAtomViewer() {
            viewerContainer.classList.remove('visible');
            const sel = document.querySelector('.selected');
            if (sel) sel.classList.remove('selected');
        }
        
        function updateInfoPanel(element) {
            const protons = element.atomicNumber;
            const electrons = element.atomicNumber;
            // Robust mass parsing
            let mass = parseFloat(element.atomicMass);
            if(isNaN(mass)) mass = element.atomicNumber * 2; // Fallback estimation
            const neutrons = Math.round(mass) - protons;

            document.getElementById('info-element-name').textContent = element.name;
            document.getElementById('info-element-symbol').textContent = element.symbol;
            document.getElementById('info-category').textContent = element.category.replace(/-/g, ' ');
            document.getElementById('info-mass').textContent = element.atomicMass;
            document.getElementById('info-protons').textContent = protons;
            document.getElementById('info-neutrons').textContent = Math.max(0, neutrons);
            document.getElementById('info-electrons').textContent = electrons;
        }

        // Simple Tween library shim for smooth open animation
        const TWEEN = {
            tweens: [],
            getAll() { return this.tweens; },
            removeAll() { this.tweens = []; },
            add(tween) { this.tweens.push(tween); },
            remove(tween) { 
                const i = this.tweens.indexOf(tween); 
                if(i !== -1) this.tweens.splice(i, 1); 
            },
            update(time) {
                if (this.tweens.length === 0) return false;
                let i = 0;
                while (i < this.tweens.length) {
                    if (this.tweens[i].update(time)) i++;
                    else this.tweens.splice(i, 1);
                }
                return true;
            },
            Tween: function(object) {
                let _object = object;
                let _valuesStart = {};
                let _valuesEnd = {};
                let _duration = 1000;
                let _easingFunction = k => k;
                let _startTime = null;
                
                this.to = function(properties, duration) {
                    _valuesEnd = properties;
                    if (duration !== undefined) _duration = duration;
                    return this;
                };
                this.easing = function(easing) {
                    _easingFunction = easing;
                    return this;
                };
                this.start = function() {
                    _startTime = Date.now();
                    for (const property in _valuesEnd) {
                        _valuesStart[property] = _object[property];
                    }
                    TWEEN.add(this);
                    return this;
                };
                this.update = function(time) {
                    let now = Date.now();
                    let elapsed = (now - _startTime) / _duration;
                    elapsed = elapsed > 1 ? 1 : elapsed;
                    const value = _easingFunction(elapsed);
                    for (const property in _valuesEnd) {
                        const start = _valuesStart[property];
                        const end = _valuesEnd[property];
                        _object[property] = start + (end - start) * value;
                    }
                    if (elapsed === 1) return false;
                    return true;
                }
            },
            Easing: { Elastic: { Out: function(k) {
                if (k === 0) return 0;
                if (k === 1) return 1;
                return Math.pow(2, -10 * k) * Math.sin((k * 10 - 0.75) * (2 * Math.PI) / 3) + 1;
            }}}
        };
        
        // Add TWEEN update to loop
        const _animate = animate;
        animate = function() {
            TWEEN.update();
            _animate();
        }

        function createAtomModel(element) {
            if (atomGroup) scene.remove(atomGroup);
            atomShells.length = 0;
            atomGroup = new THREE.Group();

            const protons = element.atomicNumber;
            let mass = parseFloat(element.atomicMass);
            if(isNaN(mass)) mass = element.atomicNumber * 2.5; // Rough approximation for unknown
            const neutrons = Math.max(0, Math.round(mass) - protons);
            const electrons = element.atomicNumber;
            
            // High quality materials
            const protonMat = new THREE.MeshPhysicalMaterial({ 
                color: 0x3b82f6, metalness: 0.4, roughness: 0.2, clearcoat: 1.0, clearcoatRoughness: 0.1, emissive: 0x111133
            });
            const neutronMat = new THREE.MeshPhysicalMaterial({ 
                color: 0xef4444, metalness: 0.4, roughness: 0.2, clearcoat: 1.0, clearcoatRoughness: 0.1, emissive: 0x331111
            });
            
            const nucleons = [];
            const nucleonRadius = 0.6; 
            const totalNucleons = protons + neutrons;

            // --- LATTICE PACKING (The "Ball" Look) ---
            // Generate a Face-Centered Cubic (FCC) lattice to pack spheres tightly
            // This ensures the nucleus looks like a solid, spherical cluster.
            const latticePoints = [];
            const latticeRange = Math.ceil(Math.cbrt(totalNucleons)) + 2; // Search radius in lattice units
            
            // Distance between sphere centers in lattice
            // In unscaled integer FCC (x+y+z=even), nearest neighbor distance is sqrt(2).
            // We want nearest neighbor distance to be nucleonRadius * 2.
            // So Scale * sqrt(2) = 2 * r  =>  Scale = 2*r / sqrt(2) = r * sqrt(2)
            const scale = nucleonRadius * Math.sqrt(2);

            for (let x = -latticeRange; x <= latticeRange; x++) {
                for (let y = -latticeRange; y <= latticeRange; y++) {
                    for (let z = -latticeRange; z <= latticeRange; z++) {
                        // FCC condition: x, y, z must sum to an even number
                        if ((x + y + z) % 2 === 0) {
                            const pos = new THREE.Vector3(x, y, z).multiplyScalar(scale);
                            latticePoints.push({ 
                                pos: pos, 
                                dist: pos.lengthSq() // Use squared distance for sorting
                            });
                        }
                    }
                }
            }

            // Sort by distance from center to form a sphere
            latticePoints.sort((a, b) => a.dist - b.dist);

            // Create arrays for protons and neutrons to shuffle them
            const types = Array(protons).fill('p').concat(Array(neutrons).fill('n'));
            
            // Shuffle types for realistic random distribution
            for (let i = types.length - 1; i > 0; i--) {
                const j = Math.floor(Math.random() * (i + 1));
                [types[i], types[j]] = [types[j], types[i]];
            }

            // Limit for performance if massive
            const limit = Math.min(totalNucleons, latticePoints.length);

            for (let i = 0; i < limit; i++) {
                const type = types[i];
                const point = latticePoints[i];
                
                let mesh;
                if (type === 'p') {
                    // Proton with slight color variation
                    const mat = protonMat.clone();
                    const hueShift = (Math.random() - 0.5) * 0.05;
                    const baseColor = new THREE.Color(0x3b82f6);
                    baseColor.offsetHSL(hueShift, 0, 0);
                    mat.color = baseColor;
                    mesh = new THREE.Mesh(new THREE.SphereGeometry(nucleonRadius, 16, 16), mat);
                } else {
                    // Neutron with slight color variation
                    const mat = neutronMat.clone();
                    const hueShift = (Math.random() - 0.5) * 0.05;
                    const baseColor = new THREE.Color(0xef4444);
                    baseColor.offsetHSL(hueShift, 0, 0);
                    mat.color = baseColor;
                    mesh = new THREE.Mesh(new THREE.SphereGeometry(nucleonRadius, 16, 16), mat);
                }
                
                mesh.position.copy(point.pos);
                nucleons.push(mesh);
                atomGroup.add(mesh);
            }

            // Electrons (Aufbau Approximation)
            const shells = [];
            const orbitalCaps = [2, 8, 18, 32, 50, 72, 98]; // n=1 to 7
            
            // Simple visual distribution (2, 8, 18...)
            let eLeft = electrons;
            
            for(let i=0; i<orbitalCaps.length; i++) {
                if(eLeft <= 0) break;
                const inShell = Math.min(eLeft, orbitalCaps[i]);
                shells.push(inShell);
                eLeft -= inShell;
            }

            const electronGeo = new THREE.SphereGeometry(0.25, 12, 12);
            const electronMat = new THREE.MeshBasicMaterial({ color: 0xfacc15 });
            
            shells.forEach((count, idx) => {
                const radius = 6 + (idx * 3.5);
                const orbitGroup = new THREE.Group();
                
                // Orbit Ring
                const ring = new THREE.Mesh(
                    new THREE.TorusGeometry(radius, 0.03, 64, 100),
                    new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.15 })
                );
                ring.rotation.x = Math.PI / 2; // Lay flat
                orbitGroup.add(ring);
                
                // Electrons
                const shellEs = [];
                for(let i=0; i<count; i++) {
                    const e = new THREE.Mesh(electronGeo, electronMat);
                    orbitGroup.add(e);
                    shellEs.push({ 
                        mesh: e, 
                        initialAngle: (i / count) * Math.PI * 2, 
                        speed: 1 
                    });
                }
                
                atomShells.push({ radius: radius, electrons: shellEs });
                
                atomGroup.add(orbitGroup);
            });

            scene.add(atomGroup);
            
            // Fit camera
            const size = 6 + (shells.length * 3.5);
            camera.position.set(0, size * 0.8, size * 1.5);
            controls.target.set(0,0,0);
        }

        // --- GEMINI ---
        async function getGeminiResponse(prompt, retries = 2, delay = 1000) {
            const apiKey = ""; 
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;
            const payload = { contents: [{ parts: [{ text: prompt }] }] };

            try {
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                if (!response.ok) {
                    if (response.status === 429 && retries > 0) {
                        await new Promise(res => setTimeout(res, delay));
                        return getGeminiResponse(prompt, retries - 1, delay * 2);
                    }
                    throw new Error(`API Error`);
                }

                const result = await response.json();
                return result.candidates?.[0]?.content?.parts?.[0]?.text || "No output.";
            } catch (error) {
                console.warn(error);
                if (retries > 0) {
                    await new Promise(res => setTimeout(res, delay));
                    return getGeminiResponse(prompt, retries - 1, delay * 2);
                }
                return "Could not connect to AI.";
            }
        }
        
        const gLoading = document.getElementById('gemini-loading');
        const gResult = document.getElementById('gemini-result');

        document.getElementById('gemini-fact-btn').addEventListener('click', async () => {
            if (!currentElement) return;
            gResult.classList.add('hidden');
            gLoading.classList.remove('hidden');
            const txt = await getGeminiResponse(`Tell me a fun, obscure fact about ${currentElement.name}. Max 2 sentences.`);
            gLoading.classList.add('hidden');
            gResult.innerHTML = `<strong>Did you know?</strong> ${txt}`;
            gResult.classList.remove('hidden');
        });

        document.getElementById('gemini-explain-btn').addEventListener('click', async () => {
            if (!currentElement) return;
            gResult.classList.add('hidden');
            gLoading.classList.remove('hidden');
            const txt = await getGeminiResponse(`Explain ${currentElement.name}'s primary use to a child. Max 2 sentences.`);
            gLoading.classList.add('hidden');
            gResult.innerHTML = `<strong>Simple Explanation:</strong> ${txt}`;
            gResult.classList.remove('hidden');
        });

        closeButton.addEventListener('click', hideAtomViewer);
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
        window.addEventListener('keydown', (event) => {
            if (event.key === 'Escape') hideAtomViewer();
        });

        init3D();
    </script>
</body>
</html>
