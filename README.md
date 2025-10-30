<!DOCTYPE html>
<html lang="en" dir="ltr">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>📋 Usage Fv</title>
  <style>
    body {
      font-family: "Cairo", Arial, sans-serif;
      background: #f0f2f5;
      padding: 20px;
    }

    h2 {
      text-align: center;
      margin-bottom: 20px;
      font-size: 24px;
      color: #1e88e5;
    }

    input[type="file"] {
      display: block;
      margin: 0 auto 10px auto;
      padding: 10px 14px;
      border-radius: 6px;
      border: 1px solid #ccc;
      cursor: pointer;
      font-size: 16px;
    }

    #resetBtn {
      display: block;
      margin: 10px auto;
      padding: 8px 16px;
      font-size: 16px;
      border-radius: 50px;
      border: none;
      background: linear-gradient(90deg, #42a5f5, #1e88e5);
      color: #fff;
      font-weight: bold;
      cursor: pointer;
      box-shadow: 0 5px 15px rgba(0, 0, 0, 0.2);
      transition: all 0.3s ease;
    }

    #resetBtn:hover {
      transform: scale(1.1);
    }

    .day {
      background: #fff;
      border-radius: 12px;
      margin: 12px 0;
      padding: 16px;
      box-shadow: 0 4px 10px rgba(0, 0, 0, 0.08);
      cursor: pointer;
      transition: all 0.3s ease;
      position: relative;
    }

    .day:hover {
      transform: translateY(-3px) scale(1.02);
      box-shadow: 0 10px 20px rgba(0, 0, 0, 0.2);
      background: #e3f2fd;
    }

    .day-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      font-weight: bold;
      font-size: 20px;
      color: #1e88e5;
    }

    .day-header::after {
      content: "▼";
      font-size: 18px;
      transition: transform 0.3s ease;
    }

    .day.open .day-header::after {
      transform: rotate(-180deg);
    }

    .totals {
      margin-top: 10px;
      font-size: 16px;
      color: #555;
      background: #f1f5f9;
      padding: 8px 12px;
      border-radius: 8px;
    }

    table {
      width: 100%;
      border-collapse: collapse;
      display: none;
      margin-top: 12px;
    }

    th,
    td {
      padding: 8px 10px;
      text-align: right;
      border-bottom: 1px solid #eee;
      font-size: 16px;
      transition: background 0.2s;
    }

    th {
      background-color: #90caf9;
      color: #fff;
      font-weight: bold;
    }

    tbody tr:nth-child(even) {
      background-color: #f7f9fc;
    }

    #mazola {
      position: fixed;
      top: 10px;
      left: 10px;
      background: rgba(30, 136, 229, 0.8);
      color: #fff;
      font-weight: bold;
      font-size: 20px;
      padding: 8px 12px;
      border-radius: 8px;
      box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
      z-index: 9999;
      font-family: "Cairo", Arial, sans-serif;
      pointer-events: none;
      animation: pulse 2s infinite;
    }

    @keyframes pulse {
      0%,
      100% {
        transform: scale(1);
      }

      50% {
        transform: scale(1.1);
      }
    }

    .deducted {
      color: #c62828;
      font-weight: bold;
    }

    .usage-badge {
      background: #e3f2fd;
      color: #1565c0;
      padding: 4px 8px;
      border-radius: 6px;
      font-weight: bold;
    }

    tr.balance-row {
      background-color: #ffebee;
    }

    @media (max-width: 768px) {
      th,
      td {
        font-size: 14px;
        padding: 6px;
      }

      .totals {
        font-size: 14px;
      }

      h2 {
        font-size: 20px;
      }
    }
  </style>
</head>

<body>
  <div id="mazola">Mazola 😎🎶</div>

  <h2>📋 Usage Fv</h2>
  <input type="file" id="fileInput" accept=".xlsx,.xls,.csv" />
  <button id="resetBtn" title="Reset All">🔄 Reset</button>
  <div id="container"></div>

  <script src="https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js"></script>
  <script>
    const resetBtn = document.getElementById("resetBtn");
    const container = document.getElementById("container");
    const fileInput = document.getElementById("fileInput");

    resetBtn.addEventListener("click", () => {
      container.innerHTML = "";
      fileInput.value = "";
    });

    fileInput.addEventListener("change", (e) => {
      const file = e.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = function (evt) {
        let data = evt.target.result;
        let jsonData = [];

        if (file.name.endsWith(".csv")) {
          const text = new TextDecoder("utf-8").decode(data);
          const rows = text.split("\n").filter(r => r.trim() !== "");
          const headers = rows[0].split(",").map(h => h.trim());
          jsonData = rows.slice(1).map(r => {
            const cols = r.split(",");
            const obj = {};
            headers.forEach((h, i) => { obj[h] = cols[i] || ""; });
            return obj;
          });
        } else {
          const workbook = XLSX.read(data, { type: "binary" });
          const firstSheet = workbook.Sheets[workbook.SheetNames[0]];
          jsonData = XLSX.utils.sheet_to_json(firstSheet, { defval: "" });
        }

        // تجاهل Monthly tariff fees
        jsonData = jsonData.filter(r => !Object.values(r).join(" ").toLowerCase().includes("monthly tariff"));

        // حساب Rating Volume / 60 و BalanceDeducted
        jsonData.forEach(r => {
          let ratingVolume = parseFloat(r["Rating Volume"] || 0) || 0;
          r["Usage60"] = (ratingVolume / 60).toFixed(2);

          const balanceBefore = parseFloat(r["Balance Before"]) || 0;
          const balanceAfter = parseFloat(r["Balance After"]) || 0;

          if ((r["Balance Deduct From"] || "").toLowerCase().includes("balance")) {
            const used = balanceBefore - balanceAfter;
            r["BalanceDeducted"] = used > 0 ? used.toFixed(2) : "0.00";
            r.isBalanceRow = true;
          } else {
            r["BalanceDeducted"] = "0.00";
            r.isBalanceRow = false;
          }

          r["BalanceBefore"] = balanceBefore.toFixed(2);
          r["BalanceAfter"] = balanceAfter.toFixed(2);
        });

        // تجميع حسب التاريخ
        const grouped = {};
        jsonData.forEach(r => {
          let date = r["Start Time"]?.split(" ")[0] || "بدون تاريخ";
          if (!grouped[date]) grouped[date] = [];
          grouped[date].push(r);
        });

        renderGrouped(grouped);
      };

      if (file.name.endsWith(".csv")) reader.readAsArrayBuffer(file);
      else reader.readAsBinaryString(file);
    });

    function renderGrouped(grouped) {
      container.innerHTML = "";

      Object.entries(grouped).forEach(([date, rows]) => {
        const dayDiv = document.createElement("div");
        dayDiv.className = "day";

        const totalUsage = rows.reduce((sum, r) => sum + parseFloat(r["Usage60"]), 0).toFixed(2);
        const totalBalanceDeducted = rows.reduce((sum, r) => sum + parseFloat(r["BalanceDeducted"]), 0).toFixed(2);

        const header = document.createElement("div");
        header.className = "day-header";
        header.textContent = date;

        const totalDiv = document.createElement("div");
        totalDiv.className = "totals";
        totalDiv.innerHTML = `Minutes totals Usage: <span class="usage-badge">${totalUsage}</span> | Balance Usage: <span class="deducted">${totalBalanceDeducted}</span>`;

        const table = document.createElement("table");
        const thead = document.createElement("thead");
        thead.innerHTML = `<tr>
      <th>Service Type</th>
      <th>Start Time</th>
      <th>End Time</th>
      <th>Balance Before</th>
      <th>Balance After</th>
      <th>Balance Usage</th>
      <th>Minute Usage</th>
    </tr>`;
        table.appendChild(thead);

        const tbody = document.createElement("tbody");
        rows.forEach(r => {
          const tr = document.createElement("tr");
          if (r.isBalanceRow) tr.classList.add("balance-row");

          tr.innerHTML = `
        <td>${r["Service Type"] || "-"}</td>
        <td>${r["Start Time"] || "-"}</td>
        <td>${r["End Time"] || "-"}</td>
        <td>${r["BalanceBefore"]}</td>
        <td>${r["BalanceAfter"]}</td>
        <td>${parseFloat(r["BalanceDeducted"]) > 0 ? `<span class="deducted">${r["BalanceDeducted"]}</span>` : "0.00"}</td>
        <td><span class="usage-badge">${r["Usage60"]}</span></td>
      `;
          tbody.appendChild(tr);
        });
        table.appendChild(tbody);

        // Toggle open/close عند الضغط على أي مكان في اليوم
        dayDiv.addEventListener("click", (e) => {
          if (e.target.closest("table")) return;
          const isOpen = table.style.display === "table";
          table.style.display = isOpen ? "none" : "table";
          dayDiv.classList.toggle("open", !isOpen);
        });

        dayDiv.appendChild(header);
        dayDiv.appendChild(totalDiv);
        dayDiv.appendChild(table);
        container.appendChild(dayDiv);
      });
    }
  </script>
</body>

</html>
