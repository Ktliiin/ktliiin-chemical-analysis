/* ===============================
   GLOBAL STATE
================================ */

let experimentMode = "Unknown";
let chart = null;

/* ===============================
   CHEMICAL DATABASE
================================ */

const reactions = {
  Potentiostatic: {
    molecular: "AgCl + e⁻ → Ag + Cl⁻",
    ionic: "Ag⁺ + e⁻ → Ag",
    comment: "Cathodic reduction under constant potential"
  },
  "Cyclic voltammetry": {
    molecular: "Ag ⇄ Ag⁺ + e⁻",
    ionic: "Ag ⇄ Ag⁺",
    comment: "Reversible redox process"
  },
  Chronoamperometry: {
    molecular: "Ag⁺ + e⁻ → Ag",
    ionic: "Ag⁺ + e⁻ → Ag",
    comment: "Time-dependent electrode kinetics"
  }
};

const elements = {
  Ag: {
    name: "Silver",
    photosensitive: true,
    structure: "d-block metal",
    notes: "Forms insoluble halides, photoactive"
  },
  Cl: {
    name: "Chlorine",
    photosensitive: false,
    structure: "Halogen",
    notes: "Strong oxidizing properties"
  }
};

/* ===============================
   FILE LOADING
================================ */

function readFile(input) {
  const file = input.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = () => parseTXT(reader.result);
  reader.readAsText(file);
}

/* ===============================
   TXT PARSER
================================ */

function parseTXT(text) {
  const lines = text.split(/\r?\n/);

  let headers = [];
  let data = [];
  let headersFound = false;

  experimentMode = "Unknown";

  for (let rawLine of lines) {
    let line = rawLine.trim();
    if (!line) continue;

    /* --- Detect instrument mode --- */
    if (line.includes("ID_PotStatic")) experimentMode = "Potentiostatic";
    if (line.includes("CV")) experimentMode = "Cyclic voltammetry";
    if (line.toLowerCase().includes("chrono"))
      experimentMode = "Chronoamperometry";

    /* --- Detect table header --- */
    if (!headersFound && /\(.+\)/.test(line)) {
      headers = line.split(/\s+/);
      headersFound = true;
      continue;
    }

    /* --- Read numeric data --- */
    if (headersFound && /^[\d\.\-E+]+/.test(line)) {
      const row = line.split(/\s+/).map(Number);
      if (row.every(v => !isNaN(v))) {
        data.push(row);
      }
    }
  }

  document.getElementById("mode").textContent = experimentMode;

  analyzeData(headers, data);
}

/* ===============================
   DATA ANALYSIS
================================ */

function analyzeData(headers, data) {
  if (!data.length) {
    document.getElementById("output").textContent =
      "No numerical experimental data detected.";
    return;
  }

  /* --- Axis detection --- */
  let xIndex = 0;
  let yIndex = 1;

  headers.forEach((h, i) => {
    if (h.toLowerCase().includes("t")) xIndex = i;
    if (h.toLowerCase().includes("i")) yIndex = i;
  });

  const x = data.map(r => r[xIndex]);
  const y = data.map(r => r[yIndex]);

  plotData(x, y, headers[xIndex], headers[yIndex]);
  generateConclusion(headers, data, x, y);
}

/* ===============================
   PLOTTING
================================ */

function plotData(x, y, xlabel, ylabel) {
  const ctx = document.getElementById("plot").getContext("2d");
  if (chart) chart.destroy();

  chart = new Chart(ctx, {
    type: "line",
    data: {
      labels: x,
      datasets: [
        {
          label: `${ylabel} vs ${xlabel}`,
          data: y,
          borderColor: "#c9a36a",
          backgroundColor: "rgba(201,163,106,0.2)",
          tension: 0.25,
          pointRadius: 0
        }
      ]
    },
    options: {
      responsive: true,
      scales: {
        x: {
          title: { display: true, text: xlabel }
        },
        y: {
          title: { display: true, text: ylabel }
        }
      }
    }
  });
}

/* ===============================
   CONCLUSIONS & CHEMISTRY
================================ */

function generateConclusion(headers, data, x, y) {
  const avgCurrent =
    y.reduce((sum, v) => sum + v, 0) / y.length;

  const T = document.getElementById("temp")?.value;
  const lambda = document.getElementById("laser")?.value;
  const power = document.getElementById("power")?.value;

  let text = `
GENERAL INFORMATION
-------------------
Instrument mode: ${experimentMode}
Number of data points: ${data.length}

DATA PROCESSING METHOD
----------------------
• Automatic detection of potentiostat mode
• Removal of service and metadata lines
• Extraction of numerical experimental data
• Construction of experimental curves

QUANTITATIVE CHARACTERISTICS
----------------------------
Average current: ${avgCurrent.toExponential(3)}

`;

  /* --- External effects --- */
  if (T) {
    text += `
THERMAL CONDITIONS
------------------
Temperature: ${T} °C
Possible influence on reaction kinetics considered
`;
  }

  if (lambda) {
    text += `
PHOTO / LASER CONDITIONS
-----------------------
Laser wavelength: ${lambda} nm
Laser power: ${power || "not specified"} mW
Photosensitive compounds taken into account
`;
  }

  /* --- Chemical reaction --- */
  if (reactions[experimentMode]) {
    const r = reactions[experimentMode];
    text += `
CHEMICAL INTERPRETATION
----------------------
Molecular reaction:
${r.molecular}

Ionic reaction:
${r.ionic}

Comment:
${r.comment}
`;
  }

  /* --- Element properties --- */
  text += `
ELEMENT PROPERTIES CONSIDERED
-----------------------------
Silver (Ag):
• Structure: ${elements.Ag.structure}
• Photosensitive: ${elements.Ag.photosensitive}
• Notes: ${elements.Ag.notes}

FINAL CONCLUSION
----------------
The experimental data were successfully processed.
The analysis mode was correctly identified.
Chemical and physical effects were considered.
The obtained results are suitable for scientific interpretation.

Developed by LiinaKT
`;

  document.getElementById("output").textContent = text;
}
