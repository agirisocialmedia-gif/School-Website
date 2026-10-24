<!DOCTYPE html>
<html lang="hi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>+2 High School, Sudhani - Result Portal</title>

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://unpkg.com/lucide@latest"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.2/papaparse.min.js"></script>

<script>
tailwind.config = {
  theme: {
    extend: {
      colors: {
        brand: {
          50:'#f0f9ff', 100:'#e0f2fe', 500:'#0284c7',
          600:'#0369a1', 700:'#075985', 800:'#0c4a6e',
          900:'#0a3651'
        }
      }
    }
  }
};
</script>

<style>
@media print {
  @page { size: A4; margin: 12mm; }
  body * { visibility: hidden !important; }
  #printable-report-card,
  #printable-report-card * { visibility: visible !important; }
  #printable-report-card {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    max-width: none;
    box-shadow: none;
    border: none;
  }
  .no-print { display: none !important; }
}
</style>
</head>

<body class="min-h-screen bg-slate-50 text-slate-800">

<header class="bg-brand-900 text-white p-4 no-print">
  <div class="max-w-4xl mx-auto flex items-center gap-3">
    <div class="bg-brand-500 p-2 rounded-lg font-bold">HSS</div>
    <div>
      <h1 class="text-lg font-bold">+2 HIGH SCHOOL, SUDHANI</h1>
      <p class="text-xs text-blue-100">Official Student Result Portal</p>
    </div>
    <span id="sync-status" class="ml-auto text-xs">Connecting...</span>
  </div>
</header>

<main class="max-w-3xl mx-auto p-4 sm:p-6 space-y-6">

  <div class="bg-white p-6 rounded-xl border text-center">
    <h2 class="text-2xl font-bold">+2 High School, Sudhani</h2>
    <p class="text-sm text-slate-600 mt-2">
      Apni class, roll number, student name aur father's name bharein.
    </p>
  </div>

  <div class="bg-white p-6 rounded-xl border shadow-sm">
    <h3 class="text-lg font-bold mb-5">Check Examination Results</h3>

    <form id="result-search-form" class="space-y-4">

      <div>
        <label for="student-class" class="block text-sm mb-1">Class / Category</label>
        <select id="student-class" required class="w-full p-3 border rounded-lg">
          <option value="Class 10">Class 10th</option>
          <option value="Class 9">Class 9th</option>
          <option value="Class 11">Class 11th</option>
          <option value="Class 12">Class 12th</option>
          <option value="Employees Performa">Employees Performa</option>
        </select>
      </div>

      <div>
        <label for="student-roll" class="block text-sm mb-1">Roll Number</label>
        <input id="student-roll" required class="w-full p-3 border rounded-lg"
               placeholder="Enter Roll Number">
      </div>

      <div>
        <label for="student-name" class="block text-sm mb-1">Student's Name</label>
        <input id="student-name" required class="w-full p-3 border rounded-lg"
               placeholder="Enter Student Name">
      </div>

      <div>
        <label for="student-father" class="block text-sm mb-1">Father's Name</label>
        <input id="student-father" required class="w-full p-3 border rounded-lg"
               placeholder="Enter Father's Name">
      </div>

      <button id="search-btn" disabled
        class="w-full bg-slate-400 text-white p-3 rounded-lg font-bold">
        Loading Database...
      </button>

      <p id="form-message" class="text-sm text-red-600" role="status"></p>
    </form>
  </div>

  <div id="report-card-container" class="hidden"></div>

</main>

<script>
// Aapki nayi Google Sheet ka CSV export link
const GOOGLE_SHEET_CSV_URL =
"https://docs.google.com/spreadsheets/d/1b2jpFNI2bTAZrjBEdvhSVjKd0fT-HLeGy6FordFX2yY/export?format=csv";

let studentDatabase = [];

const $ = id => document.getElementById(id);

function normalizeText(value) {
  return String(value ?? "")
    .trim()
    .toLowerCase()
    .replace(/\s+/g, " ");
}

function normalizeClass(value) {
  const s = normalizeText(value);
  const match = s.match(/(?:class\s*)?(9|10|11|12)(?:th|st|nd|rd)?/i);
  if (match) return "class " + match[1];
  if (s.includes("employee") || s.includes("staff")) {
    return "employees performa";
  }
  return s;
}

function getFieldValue(row, possibleKeys) {
  const keys = Object.keys(row);
  for (const key of keys) {
    const cleanKey = key.trim().toUpperCase().replace(/\s+/g, " ");
    for (const targetKey of possibleKeys) {
      if (cleanKey === targetKey.toUpperCase()) {
        return row[key];
      }
    }
  }
  return "";
}

function escapeHTML(value) {
  return String(value ?? "").replace(/[&<>"']/g, ch => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;"
  })[ch]);
}

function showMessage(message, error = true) {
  $("form-message").textContent = message;
  $("form-message").className =
    "text-sm " + (error ? "text-red-600" : "text-emerald-700");
}

function fetchGoogleSheetData() {
  if (!window.Papa) {
    $("sync-status").textContent = "CSV Library Error";
    showMessage("Internet check karein aur page reload karein.");
    return;
  }

  $("sync-status").textContent = "Connecting...";

  Papa.parse(GOOGLE_SHEET_CSV_URL, {
    download: true,
    header: true,
    skipEmptyLines: true,
    complete: function(results) {
      studentDatabase = results.data.filter(row =>
        Object.values(row).some(v => String(v ?? "").trim() !== "")
      );

      if (!studentDatabase.length) {
        $("sync-status").textContent = "No Data";
        showMessage(
          "Sheet se data nahi mila. Google Sheet sharing aur column headers check karein."
        );
        return;
      }

      $("sync-status").textContent = "Live Sheet Connected";

      const btn = $("search-btn");
      btn.disabled = false;
      btn.textContent = "View Marksheet";
      btn.className =
        "w-full bg-brand-600 hover:bg-brand-700 text-white p-3 rounded-lg font-bold";
      
      console.log("Loaded Database Records:", studentDatabase.length);
      console.log("Sample Row Keys:", Object.keys(studentDatabase[0] || {}));
    },

    error: function(error) {
      console.error(error);
      $("sync-status").textContent = "Connection Error";
      showMessage(
        "Google Sheet connect nahi hui. Internet aur Sheet sharing settings check karein."
      );
    }
  });
}

$("result-search-form").addEventListener("submit", function(event) {
  event.preventDefault();

  $("report-card-container").classList.add("hidden");
  $("report-card-container").innerHTML = "";

  if (!studentDatabase.length) {
    showMessage("Database load nahi hua. Thodi der baad try karein.");
    return;
  }

  const cls = normalizeClass($("student-class").value);
  const roll = normalizeText($("student-roll").value);
  const name = normalizeText($("student-name").value);
  const father = normalizeText($("student-father").value);

  const record = studentDatabase.find(row => {
    const rClass = normalizeClass(
      getFieldValue(row, ["CLASS", "CATEGORY", "CLASS / CATEGORY"])
    );

    const rRoll = normalizeText(
      getFieldValue(row, ["ROLL NO.", "ROLL NO", "ROLLNO", "ROLL NUMBER", "ROLL"])
    );

    const rName = normalizeText(
      getFieldValue(row, ["STUDENT'S NAME", "STUDENT NAME", "NAME"])
    );

    const rFather = normalizeText(
      getFieldValue(row, ["FATHER'S NAME", "FATHERS NAME", "FATHER NAME"])
    );

    return rClass === cls &&
           rRoll === roll &&
           rName === name &&
           rFather === father;
  });

  if (!record) {
    showMessage(
      "Record nahi mila. Class, Roll Number, Student Name aur Father's Name check karein."
    );
    return;
  }

  renderReportCard(record, cls);
});

function renderReportCard(student, selectedClass) {
  const subjects = [
    {name:"HINDI", keys:["HINDI", "HIN"]},
    {name:"URDU / SANSKRIT", keys:["URDU/SANSKRIT","URDU / SANSKRIT","SANSKRIT","URDU"]},
    {name:"ENGLISH", keys:["ENGLISH", "ENG"]},
    {name:"MATHEMATICS", keys:["MATHS","MATHEMATICS","MATH"]},
    {name:"SCIENCE", keys:["SCIENCE", "SCI"]},
    {name:"SOCIAL SCIENCE", keys:["S. SCIENCE","S SCIENCE","SOCIAL SCIENCE", "SST"]}
  ];

  let totalObtained = 0;
  let totalMax = 0;
  let subjectCount = 0;

  const rowsHTML = subjects.map(subject => {
    const raw = getFieldValue(student, subject.keys);

    if (raw === "" || raw === null || raw === undefined) return "";

    const marks = Number(String(raw).trim());

    if (!Number.isFinite(marks) || marks < 0) return "";

    const maxKeys = subject.keys.flatMap(key => [
      key + " MAX",
      key + " FULL MARKS",
      key + " FULL MARK"
    ]);

    const maxRaw = getFieldValue(student, maxKeys);
    const parsedMax = maxRaw === "" ? 100 : Number(maxRaw);
    const maxMarks = Number.isFinite(parsedMax) && parsedMax > 0
      ? parsedMax : 100;

    totalObtained += marks;
    totalMax += maxMarks;
    subjectCount++;

    return `
      <tr class="border-b">
        <td class="p-3">${escapeHTML(subject.name)}</td>
        <td class="p-3 text-center">${maxMarks}</td>
        <td class="p-3 text-center font-bold">${marks}</td>
      </tr>`;
  }).join("");

  if (!subjectCount) {
    showMessage("Student record mila, lekin marks ke columns nahi mile. Header names check karein.");
    return;
  }

  const percentage = totalMax ? (totalObtained / totalMax) * 100 : 0;

  let division = "Fail";
  if (percentage >= 60) division = "1st Division";
  else if (percentage >= 45) division = "2nd Division";
  else if (percentage >= 33) division = "3rd Division";

  const studentClass = getFieldValue(student, ["CLASS", "CATEGORY"]) || selectedClass;
  const studentName = getFieldValue(student, ["STUDENT'S NAME", "STUDENT NAME", "NAME"]) || "-";
  const fatherName = getFieldValue(student, ["FATHER'S NAME", "FATHERS NAME", "FATHER NAME"]) || "-";
  const rollNumber = getFieldValue(student, ["ROLL NO.", "ROLL NO", "ROLLNO", "ROLL NUMBER"]) || "-";

  const
