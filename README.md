<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Fuel Tracking System</title>

  <!-- Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- QR Code Generator -->
  <script src="https://cdn.jsdelivr.net/npm/qrcodejs/qrcode.min.js"></script>

  <style>
    body { background-color: #0f172a; }
    .glass {
      background: rgba(255, 255, 255, 0.05);
      border: 1px solid rgba(255, 255, 255, 0.1);
      backdrop-filter: blur(10px);
    }
  </style>
</head>

<body class="text-gray-100 font-sans flex flex-col items-center justify-center min-h-screen">

  <!-- LOGIN PAGE -->
  <div id="loginPage" class="w-full max-w-md glass p-8 rounded-2xl shadow-lg">
    <h1 class="text-3xl font-bold text-center mb-6 text-emerald-400">🚗 Car Login</h1>

    <form id="loginForm" class="space-y-4">
      <div>
        <label class="block mb-1 text-sm font-medium">Car ID</label>
        <input id="carId" type="text" required class="w-full p-2 rounded bg-gray-800 border border-gray-700 focus:ring-2 focus:ring-emerald-400">
      </div>

      <div>
        <label class="block mb-1 text-sm font-medium">Password</label>
        <input id="password" type="password" required class="w-full p-2 rounded bg-gray-800 border border-gray-700 focus:ring-2 focus:ring-emerald-400">
      </div>

      <button type="submit" class="w-full bg-emerald-500 hover:bg-emerald-600 text-white font-semibold py-2 rounded-lg transition">
        Login
      </button>
    </form>

    <p id="errorMsg" class="text-red-400 mt-3 text-center hidden">Invalid credentials!</p>
  </div>

  <!-- DASHBOARD PAGE -->
  <div id="dashboardPage" class="hidden w-full max-w-2xl glass p-8 rounded-2xl shadow-lg mt-8">
    <h2 id="carInfo" class="text-2xl font-semibold mb-4 text-center text-emerald-400"></h2>

    <!-- HISTORY SECTION -->
    <div class="mb-6">
      <h3 class="text-xl font-semibold mb-3 text-emerald-300">📜 Previous Entries</h3>
      <div id="historyList" class="space-y-2 max-h-48 overflow-y-auto"></div>
    </div>

    <form id="fuelForm" class="space-y-4">
      <div>
        <label class="block mb-1 text-sm font-medium">Driver Name</label>
        <input id="driverName" type="text" required class="w-full p-2 rounded bg-gray-800 border border-gray-700 focus:ring-2 focus:ring-emerald-400">
      </div>

      <div>
        <label class="block mb-1 text-sm font-medium">Odometer (KM)</label>
        <input id="odometer" type="number" required class="w-full p-2 rounded bg-gray-800 border border-gray-700 focus:ring-2 focus:ring-emerald-400">
        <p id="odoWarning" class="text-yellow-400 text-sm hidden mt-1">⚠️ Odometer locked for 5 hours since last entry.</p>
      </div>

      <div>
        <label class="block mb-1 text-sm font-medium">Fuel Quantity (Litres)</label>
        <input id="fuelLitres" type="number" required class="w-full p-2 rounded bg-gray-800 border border-gray-700 focus:ring-2 focus:ring-emerald-400">
      </div>

      <div>
        <label class="block mb-1 text-sm font-medium">Fuel Type</label>
        <select id="fuelType" class="w-full p-2 rounded bg-gray-800 border border-gray-700 focus:ring-2 focus:ring-emerald-400">
          <option>Petrol</option>
          <option>Diesel</option>
          <option>High Octane</option>
          <option>CNG</option>
        </select>
      </div>

      <div class="text-center mt-4">
        <video id="camera" width="200" height="150" autoplay class="mx-auto rounded"></video>
      </div>

      <button type="submit" class="w-full bg-emerald-500 hover:bg-emerald-600 text-white font-semibold py-2 rounded-lg transition">
        Submit Entry
      </button>
    </form>
  </div>

  <!-- QR CODE PAGE -->
  <div id="qrPage" class="hidden text-center mt-8">
    <h2 class="text-2xl font-semibold text-emerald-400 mb-3">✅ Entry Submitted</h2>
    <div id="qrcode" class="inline-block"></div>
    <p id="qrData" class="mt-3 text-gray-300"></p>
  </div>

  <script>
    // 🔐 Car Login Credentials
    const validCars = {
      "BJ123": "1234",
      "ASW234": "5678"
    };

    const loginPage = document.getElementById("loginPage");
    const dashboardPage = document.getElementById("dashboardPage");
    const qrPage = document.getElementById("qrPage");

    // 🔹 Login Handling
    document.getElementById("loginForm").addEventListener("submit", (e) => {
      e.preventDefault();
      const carId = document.getElementById("carId").value.trim();
      const password = document.getElementById("password").value.trim();
      const errorMsg = document.getElementById("errorMsg");

      if (validCars[carId] && validCars[carId] === password) {
        errorMsg.classList.add("hidden");
        loginPage.classList.add("hidden");
        dashboardPage.classList.remove("hidden");
        localStorage.setItem("activeCar", carId);
        document.getElementById("carInfo").innerText = "Dashboard - Car: " + carId;
        loadHistory();
        navigator.mediaDevices.getUserMedia({ video: true }).then(stream => {
          document.getElementById('camera').srcObject = stream;
        });
        checkOdoLock(carId);
      } else {
        errorMsg.classList.remove("hidden");
      }
    });

    // 🧭 Check Odometer Lock (5 hours)
    function checkOdoLock(carId) {
      const last = JSON.parse(localStorage.getItem("fuel_" + carId));
      const odoInput = document.getElementById("odometer");
      const warn = document.getElementById("odoWarning");
      if (last && last.timestamp) {
        const diffHrs = (Date.now() - last.timestamp) / (1000 * 60 * 60);
        if (diffHrs < 5) {
          odoInput.disabled = true;
          warn.classList.remove("hidden");
        } else {
          odoInput.disabled = false;
          warn.classList.add("hidden");
        }
      }
    }

    // 📋 Load History
    function loadHistory() {
      const carId = localStorage.getItem("activeCar");
      const historyDiv = document.getElementById("historyList");
      const history = JSON.parse(localStorage.getItem("history_" + carId)) || [];
      if (history.length === 0) {
        historyDiv.innerHTML = `<p class='text-gray-400'>No previous entries.</p>`;
      } else {
        historyDiv.innerHTML = history.map(h => `
          <div class="glass p-2 rounded border border-gray-700 text-sm">
            <span class="text-emerald-400">${h.date}</span> |
            ${h.driverName} | ${h.fuelLitres}L | ${h.fuelType} | ${h.odometer} KM
          </div>
        `).join("");
      }
    }

    // 🧾 Form Submission + QR + Alert System
    document.getElementById("fuelForm").addEventListener("submit", (e) => {
      e.preventDefault();
      const carId = localStorage.getItem("activeCar");

      const entry = {
        carId,
        driverName: document.getElementById("driverName").value,
        odometer: document.getElementById("odometer").value,
        fuelLitres: document.getElementById("fuelLitres").value,
        fuelType: document.getElementById("fuelType").value,
        date: new Date().toLocaleString(),
        timestamp: Date.now()
      };

      // 🔒 Odometer Lock Check (Backend alert simulation)
      const last = JSON.parse(localStorage.getItem("fuel_" + carId));
      if (last && (Date.now() - last.timestamp) / (1000 * 60 * 60) < 5) {
        console.warn("⚠️ ALERT: Odometer edited within 5 hours for Car " + carId);
        alert("⚠️ Backend Alert Sent: Odometer changed within 5 hours!");
      }

      // Save last entry and history separately
      localStorage.setItem("fuel_" + carId, JSON.stringify(entry));

      const history = JSON.parse(localStorage.getItem("history_" + carId)) || [];
      history.unshift(entry);
      localStorage.setItem("history_" + carId, JSON.stringify(history));

      dashboardPage.classList.add("hidden");
      qrPage.classList.remove("hidden");

      document.getElementById("qrcode").innerHTML = "";
      new QRCode(document.getElementById("qrcode"), JSON.stringify(entry, null, 2));
      document.getElementById("qrData").innerText = "Car: " + carId + " | " + entry.date;
    });
  </script>
</body>
</html>
