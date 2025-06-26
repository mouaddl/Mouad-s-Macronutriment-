# Mouad-s-Macronutriment-
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <title>Suivi des Macros - Avancé</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body {
      font-family: 'Segoe UI', sans-serif;
      background: #f7f9fc;
      color: #333;
      padding: 20px;
      max-width: 800px;
      margin: auto;
    }
    h1, h2 {
      color: #222;
    }
    .bar {
      height: 20px;
      background: #e0e0e0;
      border-radius: 5px;
      margin-bottom: 10px;
      overflow: hidden;
    }
    .bar-fill {
      height: 100%;
      color: white;
      text-align: right;
      padding-right: 5px;
      font-size: 12px;
      font-weight: bold;
    }
    .calories { background: #ff6666; }
    .proteins { background: #66cc66; }
    .fats { background: #ffcc66; }
    .carbs { background: #66b3ff; }
    .fibers { background: #cc99ff; }
    input, select, button {
      margin: 5px 0;
      padding: 8px;
      width: 100%;
      font-size: 14px;
    }
    button {
      background-color: #007bff;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
    }
    button:hover {
      background-color: #0056b3;
    }
    .meal-item, .weight-entry {
      border: 1px solid #ccc;
      background: white;
      padding: 10px;
      border-radius: 5px;
      margin-bottom: 10px;
    }
    .section {
      margin-bottom: 30px;
    }
  </style>
</head>
<body>
  <h1>Suivi des Macros (par jour)</h1>

  <div class="section">
    <label for="date">Date : </label>
    <input type="date" id="date">

    <div id="macros-bars"></div>
    <div id="alert"></div>
  </div>

  <div class="section">
    <h2>Modifier les objectifs</h2>
    <form id="goal-form">
      <input type="number" id="goal-calories" placeholder="Calories">
      <input type="number" id="goal-proteins" placeholder="Protéines (g)">
      <input type="number" id="goal-fibers" placeholder="Fibres (g)">
      <input type="number" id="goal-fats" placeholder="Lipides (g)">
      <input type="number" id="goal-carbs" placeholder="Glucides (g)">
      <button type="submit">Mettre à jour</button>
    </form>
  </div>

  <div class="section">
    <h2>Ajouter un repas</h2>
    <form id="meal-form">
      <input type="text" id="meal-name" placeholder="Nom du repas" required>
      <input type="number" id="calories" placeholder="Calories">
      <input type="number" id="proteins" placeholder="Protéines (g)">
      <input type="number" id="fibers" placeholder="Fibres (g)">
      <input type="number" id="fats" placeholder="Lipides (g)">
      <input type="number" id="carbs" placeholder="Glucides (g)">
      <button type="submit">Ajouter</button>
    </form>
    <div id="meal-list"></div>
  </div>

  <div class="section">
    <h2>Historique des macros</h2>
    <div id="macro-history"></div>
  </div>

  <div class="section">
    <h2>Suivi du poids</h2>
    <form id="weight-form">
      <input type="date" id="weight-date">
      <input type="number" id="weight-input" placeholder="Poids (kg)">
      <button type="submit">Ajouter</button>
    </form>
    <div id="weight-list"></div>
    <canvas id="weightChart"></canvas>
  </div>

  <script>
    let macroData = JSON.parse(localStorage.getItem('macroData')) || {};
    let weightHistory = JSON.parse(localStorage.getItem('weightHistory')) || [];
    const defaultGoals = { calories: 2000, proteins: 136, fibers: 36, fats: 130, carbs: 130 };

    function getToday() {
      return document.getElementById('date').value;
    }

    function initDate() {
      const today = new Date();
      document.getElementById('date').value = today.toISOString().split('T')[0];
    }

    function updateGoalsForm(goals) {
      document.getElementById('goal-calories').value = goals.calories;
      document.getElementById('goal-proteins').value = goals.proteins;
      document.getElementById('goal-fibers').value = goals.fibers;
      document.getElementById('goal-fats').value = goals.fats;
      document.getElementById('goal-carbs').value = goals.carbs;
    }

    function getOrInitDay(date) {
      if (!macroData[date]) {
        macroData[date] = {
          goals: { ...defaultGoals },
          meals: []
        };
      }
      return macroData[date];
    }

    function updateBars(date) {
      const day = getOrInitDay(date);
      const totals = { calories: 0, proteins: 0, fibers: 0, fats: 0, carbs: 0 };
      day.meals.forEach(m => {
        for (let k in totals) totals[k] += m[k];
      });
      const remaining = {};
      for (let k in day.goals) {
        remaining[k] = Math.max(0, day.goals[k] - totals[k]);
      }
      const bars = Object.keys(day.goals).map(key => {
        const percent = Math.min(100, (remaining[key] / day.goals[key]) * 100);
        return `
          <div>
            <label>${key.charAt(0).toUpperCase() + key.slice(1)}: ${remaining[key].toFixed(1)}</label>
            <div class="bar">
              <div class="bar-fill ${key}" style="width:${percent}%">${remaining[key].toFixed(0)}</div>
            </div>
          </div>`;
      }).join('');
      document.getElementById('macros-bars').innerHTML = bars;
    }

    function renderMeals(date) {
      const meals = getOrInitDay(date).meals;
      const container = document.getElementById('meal-list');
      container.innerHTML = meals.map((meal, i) => `
        <div class="meal-item">
          <strong>${meal.name}</strong><br>
          ${meal.calories} kcal, ${meal.proteins}g prot, ${meal.fibers}g fibres, ${meal.fats}g lip, ${meal.carbs}g gluc<br>
          <button onclick="deleteMeal('${date}', ${i})">🗑️ Supprimer</button>
        </div>
      `).join('');
    }

    function deleteMeal(date, index) {
      macroData[date].meals.splice(index, 1);
      saveData();
      update(date);
    }

    function renderMacroHistory() {
      const historyDiv = document.getElementById('macro-history');
      historyDiv.innerHTML = Object.entries(macroData).map(([date, day]) => {
        const totals = { calories: 0, proteins: 0, fibers: 0, fats: 0, carbs: 0 };
        day.meals.forEach(m => {
          for (let k in totals) totals[k] += m[k];
        });
        return `<div class="meal-item"><strong>${date}</strong>: ${totals.calories} kcal, ${totals.proteins}g prot, ${totals.carbs}g gluc</div>`;
      }).join('');
    }

    function renderWeights() {
      const list = document.getElementById('weight-list');
      list.innerHTML = weightHistory.map((w, i) => `
        <div class="weight-entry">
          ${w.date} — ${w.weight} kg <button onclick="deleteWeight(${i})">❌</button>
        </div>
      `).join('');
    }

    function deleteWeight(index) {
      weightHistory.splice(index, 1);
      saveWeights();
      renderWeights();
      renderChart();
    }

    function renderChart() {
      const ctx = document.getElementById('weightChart').getContext('2d');
      const sorted = [...weightHistory].sort((a, b) => a.date.localeCompare(b.date));
      new Chart(ctx, {
        type: 'line',
        data: {
          labels: sorted.map(e => e.date),
          datasets: [{
            label: 'Poids (kg)',
            data: sorted.map(e => e.weight),
            borderColor: '#007bff',
            backgroundColor: 'rgba(0,123,255,0.1)',
            fill: true
          }]
        },
        options: { responsive: true, plugins: { legend: { display: true } } }
      });
    }

    function update(date) {
      updateBars(date);
      renderMeals(date);
      renderMacroHistory();
    }

    function saveData() {
      localStorage.setItem('macroData', JSON.stringify(macroData));
    }

    function saveWeights() {
      localStorage.setItem('weightHistory', JSON.stringify(weightHistory));
    }

    document.getElementById('meal-form').addEventListener('submit', e => {
      e.preventDefault();
      const date = getToday();
      const meal = {
        name: document.getElementById('meal-name').value,
        calories: parseFloat(document.getElementById('calories').value) || 0,
        proteins: parseFloat(document.getElementById('proteins').value) || 0,
        fibers: parseFloat(document.getElementById('fibers').value) || 0,
        fats: parseFloat(document.getElementById('fats').value) || 0,
        carbs: parseFloat(document.getElementById('carbs').value) || 0
      };
      macroData[date].meals.push(meal);
      saveData();
      update(date);
      e.target.reset();
    });

    document.getElementById('goal-form').addEventListener('submit', e => {
      e.preventDefault();
      const date = getToday();
      const goals = {
        calories: parseFloat(document.getElementById('goal-calories').value) || defaultGoals.calories,
        proteins: parseFloat(document.getElementById('goal-proteins').value) || defaultGoals.proteins,
        fibers: parseFloat(document.getElementById('goal-fibers').value) || defaultGoals.fibers,
        fats: parseFloat(document.getElementById('goal-fats').value) || defaultGoals.fats,
        carbs: parseFloat(document.getElementById('goal-carbs').value) || defaultGoals.carbs
      };
      macroData[date].goals = goals;
      saveData();
      update(date);
    });

    document.getElementById('weight-form').addEventListener('submit', e => {
      e.preventDefault();
      const date = document.getElementById('weight-date').value;
      const weight = parseFloat(document.getElementById('weight-input').value);
      if (date && weight) {
        weightHistory.push({ date, weight });
        saveWeights();
        renderWeights();
        renderChart();
        e.target.reset();
      }
    });

    document.getElementById('date').addEventListener('change', e => {
      const date = e.target.value;
      updateGoalsForm(getOrInitDay(date).goals);
      update(date);
    });

    initDate();
    const today = getToday();
    updateGoalsForm(getOrInitDay(today).goals);
    update(today);
    renderWeights();
    renderChart();
  </script>
</body>
</html>
