# Income Form + Fleet Log Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the income form's Техника/Прочее/Предоплата classification with Наличные/Безнал/Прочее, move equipment tracking out of the income form into its own Firestore-backed log (mirroring the existing "Источник" panel), and regroup both the "Расходы/Доходы" and fleet charts to match.

**Architecture:** Same single-file `index.html`, same Firestore realtime pattern established in the previous migration (`onSnapshot` populating `state`, writes via `addDoc`/`writeBatch`). The fleet log is a structural copy of the existing `sourceRecords`/"Источник" implementation — same modal pattern, same chart-grouping pattern (core list vs "Прочее" with expandable sublist).

**Tech Stack:** Vanilla JS, Firebase Firestore (already wired up), Chart.js (already loaded).

## Global Constraints

- Spec: `docs/superpowers/specs/2026-08-15-income-form-fleet-log-redesign.md` — all decisions below are copied verbatim from it.
- "Предоплата" mode is removed entirely (not merged, not kept as a 4th option).
- "Прочее" income section has only category text + amount — no payment method, no client field.
- Equipment tracking is removed from the income form entirely and lives only in the new `fleetRecords` log.
- Old income records (test entry, historical import) are **not** migrated — they'll simply show up under "Прочее" in the income chart, and contribute nothing to the fleet chart. This is intentional, confirmed by the user.
- Expense form/logic and the "Источник" panel are untouched.
- Local verification uses `python3 -m http.server 8080` in the project directory (already the established workflow) — keep each verification to 1-2 concrete actions, not a long checklist (per explicit user feedback earlier in this project).
- Commit after each task.

---

### Task 1: New income form (Наличные/Безнал/Прочее) + "Расходы/Доходы" chart regrouping

**Files:**
- Modify: `index.html` (income form markup, income form JS logic, chart 1 income-grouping logic)

**Interfaces:**
- Produces: `CORE_INCOME_CATEGORIES` constant (`['Наличные','Безнал']`), `incomeCategoryMode` values `'cash'|'noncash'|'other'` (replacing `'technika'|'other'|'prepay'`).

- [ ] **Step 1: Simplify the income form markup**

Find:
```html
      <div id="incomeFields">
        <div class="type-toggle income-cat-toggle">
          <button type="button" data-icat="technika" class="active">Техника</button>
          <button type="button" data-icat="other">Прочее</button>
          <button type="button" data-icat="prepay">Предоплата</button>
        </div>

        <div id="technikaItemsWrap">
          <div id="technikaItemsList"></div>
          <button type="button" id="addTechnikaItemBtn" class="add-technika-btn">+ Добавить технику</button>
        </div>

        <div class="field" id="fOtherCategoryWrap" style="display:none;">
          <label for="fOtherCategory">Категория</label>
          <input type="text" id="fOtherCategory" placeholder="Например, поломки">
        </div>

        <div id="fPaymentGroup">
          <div class="field">
            <label for="fPayment">Способ оплаты</label>
            <select id="fPayment"></select>
          </div>
          <div class="field" id="fPaymentManualWrap" style="display:none;">
            <label for="fPaymentManual">Свой способ оплаты</label>
            <input type="text" id="fPaymentManual" placeholder="Введите способ оплаты">
          </div>
        </div>

        <div class="field" id="fClientWrap" style="display:none;">
          <label for="fClient">Имя или номер клиента</label>
          <input type="text" id="fClient" placeholder="Необязательно">
        </div>
        <div class="field" id="fPrepaymentWrap" style="display:none;">
          <label for="fPrepayment">Сумма предоплаты, ₽</label>
          <input type="number" id="fPrepayment" min="0" step="1">
        </div>
      </div>
```
Replace with:
```html
      <div id="incomeFields">
        <div class="type-toggle income-cat-toggle">
          <button type="button" data-icat="cash" class="active">Наличные</button>
          <button type="button" data-icat="noncash">Безнал</button>
          <button type="button" data-icat="other">Прочее</button>
        </div>

        <div class="field" id="fOtherCategoryWrap" style="display:none;">
          <label for="fOtherCategory">Категория</label>
          <input type="text" id="fOtherCategory" placeholder="Например, поломки">
        </div>
      </div>
```

- [ ] **Step 2: Remove DOM references to the deleted fields**

Find:
```javascript
  const typeToggleBtns = document.querySelectorAll('.entry-type-toggle button');
  const incomeCatBtns = document.querySelectorAll('.income-cat-toggle button');
  const fOtherCategoryWrap = document.getElementById('fOtherCategoryWrap');
  const fOtherCategory = document.getElementById('fOtherCategory');
  const fClientWrap = document.getElementById('fClientWrap');
  const fPrepaymentWrap = document.getElementById('fPrepaymentWrap');
  const incomeFields = document.getElementById('incomeFields');
  const expenseFields = document.getElementById('expenseFields');
  const entryForm = document.getElementById('entryForm');
  const fDate = document.getElementById('fDate');
  const technikaItemsWrap = document.getElementById('technikaItemsWrap');
  const technikaItemsList = document.getElementById('technikaItemsList');
  const addTechnikaItemBtn = document.getElementById('addTechnikaItemBtn');
  const fPaymentGroup = document.getElementById('fPaymentGroup');
  const fPayment = document.getElementById('fPayment');
  const fPaymentManualWrap = document.getElementById('fPaymentManualWrap');
  const fPaymentManual = document.getElementById('fPaymentManual');
  const expenseItemsList = document.getElementById('expenseItemsList');
  const addExpenseItemBtn = document.getElementById('addExpenseItemBtn');
  const fClient = document.getElementById('fClient');
  const fPrepayment = document.getElementById('fPrepayment');
  const fAmountWrap = document.getElementById('fAmountWrap');
```
Replace with:
```javascript
  const typeToggleBtns = document.querySelectorAll('.entry-type-toggle button');
  const incomeCatBtns = document.querySelectorAll('.income-cat-toggle button');
  const fOtherCategoryWrap = document.getElementById('fOtherCategoryWrap');
  const fOtherCategory = document.getElementById('fOtherCategory');
  const incomeFields = document.getElementById('incomeFields');
  const expenseFields = document.getElementById('expenseFields');
  const entryForm = document.getElementById('entryForm');
  const fDate = document.getElementById('fDate');
  const expenseItemsList = document.getElementById('expenseItemsList');
  const addExpenseItemBtn = document.getElementById('addExpenseItemBtn');
  const fAmountWrap = document.getElementById('fAmountWrap');
```

- [ ] **Step 3: Remove the technika-item and payment-method helper functions**

Find (from `function resetManualFields(){` through the blank line right before `function createExpenseItemRow(showLabel){`):
```javascript
  function resetManualFields(){
    fPaymentManualWrap.style.display = 'none'; fPaymentManual.value = '';
  }
  function wireManualToggle(select, wrapEl, inputEl){
    select.addEventListener('change', ()=>{
      if(select.value === MANUAL_OPTION){
        wrapEl.style.display = 'block';
        inputEl.focus();
      } else {
        wrapEl.style.display = 'none';
        inputEl.value = '';
      }
    });
  }
  wireManualToggle(fPayment, fPaymentManualWrap, fPaymentManual);

  function createTechnikaItemRow(showLabels){
    const row = document.createElement('div');
    row.className = 'technika-item-row';
    row.innerHTML =
      '<div class="field">' +
        (showLabels ? '<label>Техника</label>' : '') +
        '<select class="ti-select"></select>' +
        '<input type="text" class="ti-manual" placeholder="Введите название техники" style="display:none;margin-top:8px;">' +
      '</div>' +
      '<div class="field ti-qty-field">' +
        (showLabels ? '<label>Кол-во</label>' : '') +
        '<input type="number" class="ti-qty" min="1" step="1" value="1">' +
      '</div>' +
      '<button type="button" class="ti-remove" aria-label="Удалить">&times;</button>';
    const select = row.querySelector('.ti-select');
    populateSelect(select, state.equipmentTypes, MANUAL_OPTION, 'Заполнить вручную');
    const manualInput = row.querySelector('.ti-manual');
    select.addEventListener('change', ()=>{
      if(select.value === MANUAL_OPTION){
        manualInput.style.display = 'block';
        manualInput.focus();
      } else {
        manualInput.style.display = 'none';
        manualInput.value = '';
      }
    });
    row.querySelector('.ti-remove').addEventListener('click', ()=>{
      if(technikaItemsList.children.length > 1){
        row.remove();
      }
    });
    return row;
  }
  function resetTechnikaItems(){
    technikaItemsList.innerHTML = '';
    technikaItemsList.appendChild(createTechnikaItemRow(true));
  }
  function refreshTechnikaItemSelects(){
    document.querySelectorAll('#technikaItemsList .ti-select').forEach(sel=>{
      const current = sel.value;
      populateSelect(sel, state.equipmentTypes, MANUAL_OPTION, 'Заполнить вручную');
      if(Array.from(sel.options).some(o=>o.value===current)) sel.value = current;
    });
  }
  addTechnikaItemBtn.addEventListener('click', ()=>{
    technikaItemsList.appendChild(createTechnikaItemRow(false));
  });

  function createExpenseItemRow(showLabel){
```
Replace with:
```javascript
  function createExpenseItemRow(showLabel){
```

- [ ] **Step 4: Rewrite `updateIncomeCategoryMode()` for the 3 new modes**

Find:
```javascript
  let incomeCategoryMode = 'technika';
  function updateIncomeCategoryMode(){
    const isTechnika = incomeCategoryMode === 'technika';
    const isOther = incomeCategoryMode === 'other';
    const isPrepay = incomeCategoryMode === 'prepay';

    technikaItemsWrap.style.display = isTechnika ? 'block' : 'none';
    fOtherCategoryWrap.style.display = isOther ? 'block' : 'none';
    fPaymentGroup.style.display = isOther ? 'block' : 'none';
    fAmountWrap.style.display = isPrepay ? 'none' : 'block';
    fClientWrap.style.display = isPrepay ? 'block' : 'none';
    fPrepaymentWrap.style.display = isPrepay ? 'block' : 'none';
  }
```
Replace with:
```javascript
  let incomeCategoryMode = 'cash';
  function updateIncomeCategoryMode(){
    const isOther = incomeCategoryMode === 'other';
    fOtherCategoryWrap.style.display = isOther ? 'block' : 'none';
  }
```

- [ ] **Step 5: Simplify `openModal()`**

Find:
```javascript
  function openModal(){
    resetManualFields();
    fClient.value = '';
    fPrepayment.value = '';
    fOtherCategory.value = '';
    resetTechnikaItems();
    resetExpenseItems();
    incomeCategoryMode = 'technika';
    incomeCatBtns.forEach(b=> b.classList.remove('active'));
    if(incomeCatBtns[0]) incomeCatBtns[0].classList.add('active');
    updateIncomeCategoryMode();
    modalOverlay.classList.add('open');
  }
```
Replace with:
```javascript
  function openModal(){
    fOtherCategory.value = '';
    resetExpenseItems();
    incomeCategoryMode = 'cash';
    incomeCatBtns.forEach(b=> b.classList.remove('active'));
    if(incomeCatBtns[0]) incomeCatBtns[0].classList.add('active');
    updateIncomeCategoryMode();
    modalOverlay.classList.add('open');
  }
```

- [ ] **Step 6: Simplify the amount field visibility in the type-toggle handler**

Find:
```javascript
      fAmountWrap.style.display = currentType === 'expense' ? 'none' : (incomeCategoryMode === 'prepay' ? 'none' : 'block');
```
Replace with:
```javascript
      fAmountWrap.style.display = currentType === 'expense' ? 'none' : 'block';
```

- [ ] **Step 7: Simplify `refreshSelects()`**

Find:
```javascript
  function refreshSelects(){
    populateSelect(fPayment, state.paymentMethods, MANUAL_OPTION, 'Заполнить вручную');
    refreshTechnikaItemSelects();
    refreshExpenseItemSelects();
    refreshSourceItemSelects();
    resetManualFields();
  }
```
Replace with:
```javascript
  function refreshSelects(){
    refreshExpenseItemSelects();
    refreshSourceItemSelects();
  }
```
(Task 2 will add a `refreshFleetItemSelects()` call here once that function exists.)

- [ ] **Step 8: Simplify `resetEntryForm()`**

Find:
```javascript
  function resetEntryForm(){
    fAmount.value = '';
    fClient.value = '';
    fPrepayment.value = '';
    fOtherCategory.value = '';
    resetTechnikaItems();
    resetExpenseItems();
  }
```
Replace with:
```javascript
  function resetEntryForm(){
    fAmount.value = '';
    fOtherCategory.value = '';
    resetExpenseItems();
  }
```

- [ ] **Step 9: Rewrite the income branch of `entryForm`'s submit handler**

Find:
```javascript
    // income
    let record;
    if(incomeCategoryMode === 'prepay'){
      const prepaymentVal = parseFloat(fPrepayment.value);
      const hasPrepayment = !isNaN(prepaymentVal) && prepaymentVal > 0;
      const clientName = (fClient.value || '').trim();
      if(!clientName){ alert('Укажите имя или номер клиента'); return; }
      if(!hasPrepayment){ alert('Укажите сумму предоплаты'); return; }
      record = {
        createdAt: Date.now(), date: fDate.value, type:'income', status:'completed',
        client: clientName, prepayment: prepaymentVal, amount: prepaymentVal,
        category:'Предоплата', qty:0, payment:'', source:''
      };
    } else if(incomeCategoryMode === 'technika'){
      const amountVal = parseFloat(fAmount.value);
      if(isNaN(amountVal) || amountVal <= 0){ alert('Укажите сумму'); return; }
      const items = [];
      Array.from(technikaItemsList.querySelectorAll('.technika-item-row')).forEach(row=>{
        const select = row.querySelector('.ti-select');
        const manualInput = row.querySelector('.ti-manual');
        const qtyInput = row.querySelector('.ti-qty');
        const val = resolveValue(select, manualInput, state.equipmentTypes);
        const qty = parseInt(qtyInput.value,10) || 0;
        if(val && qty > 0) items.push({category: val, qty: qty});
      });
      if(items.length === 0){ alert('Добавьте технику и количество'); return; }
      const totalQty = items.reduce((a,b)=> a+b.qty, 0);
      const categoryVal = items.map(it=> it.category + ' ×' + it.qty).join(', ');
      record = {
        createdAt: Date.now(), date: fDate.value, type:'income', status:'completed',
        category: categoryVal, qty: totalQty,
        payment:'', source:'', client:'',
        prepayment: 0,
        amount: amountVal,
        items: items
      };
    } else {
      const amountVal = parseFloat(fAmount.value);
      if(isNaN(amountVal) || amountVal <= 0){ alert('Укажите сумму'); return; }
      const paymentVal = resolveValue(fPayment, fPaymentManual, state.paymentMethods);
      if(!paymentVal){ alert('Укажите способ оплаты'); return; }
      const categoryVal = (fOtherCategory.value || '').trim();
      if(!categoryVal){ alert('Укажите категорию'); return; }
      record = {
        createdAt: Date.now(), date: fDate.value, type:'income', status:'completed',
        category: categoryVal, qty: 0,
        payment: paymentVal, source: '',
        client: '',
        prepayment: 0,
        amount: amountVal
      };
    }
    addDoc(transactionsCol, record);
    resetEntryForm();
    closeModal();
  });
```
Replace with:
```javascript
    // income
    const amountVal = parseFloat(fAmount.value);
    if(isNaN(amountVal) || amountVal <= 0){ alert('Укажите сумму'); return; }
    let record;
    if(incomeCategoryMode === 'cash'){
      record = {
        createdAt: Date.now(), date: fDate.value, type:'income', status:'completed',
        category:'Наличные', qty:0, payment:'', source:'', client:'', prepayment:0,
        amount: amountVal
      };
    } else if(incomeCategoryMode === 'noncash'){
      record = {
        createdAt: Date.now(), date: fDate.value, type:'income', status:'completed',
        category:'Безнал', qty:0, payment:'', source:'', client:'', prepayment:0,
        amount: amountVal
      };
    } else {
      const categoryVal = (fOtherCategory.value || '').trim();
      if(!categoryVal){ alert('Укажите категорию'); return; }
      record = {
        createdAt: Date.now(), date: fDate.value, type:'income', status:'completed',
        category: categoryVal, qty:0, payment:'', source:'', client:'', prepayment:0,
        amount: amountVal
      };
    }
    addDoc(transactionsCol, record);
    resetEntryForm();
    closeModal();
  });
```

- [ ] **Step 10: Add `CORE_INCOME_CATEGORIES` and regroup the "Расходы/Доходы" income chart**

Find:
```javascript
  const CORE_EXPENSE_CATEGORIES = ['ГСМ','Реклама','Аренда','ЗП работника','Запчасти'];
  const CORE_SOURCES = ['ВКонтакте','Инст','Авито','Яндекс','Рекомендация'];
```
Replace with:
```javascript
  const CORE_EXPENSE_CATEGORIES = ['ГСМ','Реклама','Аренда','ЗП работника','Запчасти'];
  const CORE_SOURCES = ['ВКонтакте','Инст','Авито','Яндекс','Рекомендация'];
  const CORE_INCOME_CATEGORIES = ['Наличные','Безнал'];
```

Find:
```javascript
    const incomeByCat = {};
    const otherIncomeDetails = {};
    incomeTx.forEach(t=>{
      if(!t.category) return;
      const isTechnika = (t.items && t.items.length) || CORE_FLEET_TYPES.includes(t.category);
      const label = isTechnika ? 'Техника' : 'Прочее';
      incomeByCat[label] = (incomeByCat[label]||0) + t.amount;
      if(!isTechnika) otherIncomeDetails[t.category] = (otherIncomeDetails[t.category]||0) + t.amount;
    });
```
Replace with:
```javascript
    const incomeByCat = {};
    const otherIncomeDetails = {};
    incomeTx.forEach(t=>{
      if(!t.category) return;
      const isCoreIncome = CORE_INCOME_CATEGORIES.includes(t.category);
      const label = isCoreIncome ? t.category : 'Прочее';
      incomeByCat[label] = (incomeByCat[label]||0) + t.amount;
      if(!isCoreIncome) otherIncomeDetails[t.category] = (otherIncomeDetails[t.category]||0) + t.amount;
    });
```

- [ ] **Step 11: Verify locally**

```bash
cd "/Users/aleksej/Desktop/Мотопрокат учет"
(lsof -i :8080 -sTCP:LISTEN -t | xargs -r kill) 2>/dev/null
python3 -m http.server 8080
```
Open `http://localhost:8080/`, log in, click "+ Новая запись". You should see three buttons: **Наличные / Безнал / Прочее**. Leave "Наличные" selected, enter an amount (e.g. 1000), submit.

Expected: the entry appears in the table with category "Наличные", and on the "Расходы/Доходы" chart there's now a "Наличные" slice.

- [ ] **Step 12: Commit**

```bash
git add index.html
git commit -m "Replace income form categories with cash/noncash/other"
```

---

### Task 2: Fleet log (own Firestore collection + modal) and fleet chart rewrite

**Files:**
- Modify: `index.html` (new panel button + modal markup, new JS mirroring the "Источник" implementation, `render()`'s fleet chart)

**Interfaces:**
- Consumes: `resolveValue`, `populateSelect`, `MANUAL_OPTION`, `db`, `writeBatch`, `doc`, `collection`, `onSnapshot`, `CORE_FLEET_TYPES`, `colorFor`, `escapeHtml` (all already defined).
- Produces: `fleetRecordsCol`, `state.fleetRecords`, `getFilteredFleetRecords()`, `refreshFleetItemSelects()`, `openFleetModal()`/`closeFleetModal()`.

- [ ] **Step 1: Add the "+" button to the "Учет проката техники" panel**

Find:
```html
      <div class="panel">
        <h2>Учет проката техники</h2>
        <div class="chart-wrap"><canvas id="chartFleet"></canvas></div>
        <ul class="chart-list" id="fleetList"></ul>
      </div>
```
Replace with:
```html
      <div class="panel">
        <div class="panel-head-row">
          <h2>Учет проката техники</h2>
          <button type="button" class="panel-add-btn" id="openFleetModalBtn" aria-label="Добавить технику">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>
          </button>
        </div>
        <div class="chart-wrap"><canvas id="chartFleet"></canvas></div>
        <ul class="chart-list" id="fleetList"></ul>
      </div>
```

- [ ] **Step 2: Add the fleet modal markup**

Find:
```html
      <div id="sourceItemsList"></div>
      <button type="button" id="addSourceItemBtn" class="add-technika-btn">+ Добавить источник</button>
      <button type="submit" class="btn-submit">Добавить запись</button>
    </form>
  </div>
</div>

<div class="modal-overlay" id="modalOverlay">
```
Replace with:
```html
      <div id="sourceItemsList"></div>
      <button type="button" id="addSourceItemBtn" class="add-technika-btn">+ Добавить источник</button>
      <button type="submit" class="btn-submit">Добавить запись</button>
    </form>
  </div>
</div>

<div class="modal-overlay" id="fleetModalOverlay">
  <div class="modal">
    <div class="modal-head">
      <h2>Учёт техники</h2>
      <button type="button" class="modal-close" id="closeFleetModalBtn">&times;</button>
    </div>
    <form id="fleetForm">
      <div class="field">
        <label for="flDate">Дата</label>
        <input type="date" id="flDate" required>
      </div>
      <div id="fleetItemsList"></div>
      <button type="button" id="addFleetItemBtn" class="add-technika-btn">+ Добавить технику</button>
      <button type="submit" class="btn-submit">Добавить запись</button>
    </form>
  </div>
</div>

<div class="modal-overlay" id="modalOverlay">
```

- [ ] **Step 3: Add DOM references for the fleet modal**

Find:
```javascript
  const sourceModalOverlay = document.getElementById('sourceModalOverlay');
  const openSourceModalBtn = document.getElementById('openSourceModalBtn');
  const closeSourceModalBtn = document.getElementById('closeSourceModalBtn');
  const sourceForm = document.getElementById('sourceForm');
  const sDate = document.getElementById('sDate');
  const sourceItemsList = document.getElementById('sourceItemsList');
  const addSourceItemBtn = document.getElementById('addSourceItemBtn');
```
Replace with:
```javascript
  const sourceModalOverlay = document.getElementById('sourceModalOverlay');
  const openSourceModalBtn = document.getElementById('openSourceModalBtn');
  const closeSourceModalBtn = document.getElementById('closeSourceModalBtn');
  const sourceForm = document.getElementById('sourceForm');
  const sDate = document.getElementById('sDate');
  const sourceItemsList = document.getElementById('sourceItemsList');
  const addSourceItemBtn = document.getElementById('addSourceItemBtn');

  const fleetModalOverlay = document.getElementById('fleetModalOverlay');
  const openFleetModalBtn = document.getElementById('openFleetModalBtn');
  const closeFleetModalBtn = document.getElementById('closeFleetModalBtn');
  const fleetForm = document.getElementById('fleetForm');
  const flDate = document.getElementById('flDate');
  const fleetItemsList = document.getElementById('fleetItemsList');
  const addFleetItemBtn = document.getElementById('addFleetItemBtn');
```

- [ ] **Step 4: Add the Firestore collection ref and `state.fleetRecords`**

Find:
```javascript
  const settingsRef = doc(db, 'settings', 'config');
  const transactionsCol = collection(db, 'transactions');
  const sourceRecordsCol = collection(db, 'sourceRecords');
  let syncStarted = false;
  let unsubscribers = [];

  let state = {
    equipmentTypes: [], otherIncomeCategories: [], expenseCategories: [],
    paymentMethods: [], sources: [], transactions: [], sourceRecords: []
  };
```
Replace with:
```javascript
  const settingsRef = doc(db, 'settings', 'config');
  const transactionsCol = collection(db, 'transactions');
  const sourceRecordsCol = collection(db, 'sourceRecords');
  const fleetRecordsCol = collection(db, 'fleetRecords');
  let syncStarted = false;
  let unsubscribers = [];

  let state = {
    equipmentTypes: [], otherIncomeCategories: [], expenseCategories: [],
    paymentMethods: [], sources: [], transactions: [], sourceRecords: [], fleetRecords: []
  };
```

- [ ] **Step 5: Subscribe to `fleetRecords` in `startFirestoreSync()`**

Find:
```javascript
    unsubscribers.push(onSnapshot(sourceRecordsCol, (snap)=>{
      state.sourceRecords = snap.docs.map(d=> ({ id: d.id, ...d.data() }));
      render();
    }));
  }
```
Replace with:
```javascript
    unsubscribers.push(onSnapshot(sourceRecordsCol, (snap)=>{
      state.sourceRecords = snap.docs.map(d=> ({ id: d.id, ...d.data() }));
      render();
    }));
    unsubscribers.push(onSnapshot(fleetRecordsCol, (snap)=>{
      state.fleetRecords = snap.docs.map(d=> ({ id: d.id, ...d.data() }));
      render();
    }));
  }
```

- [ ] **Step 6: Add the fleet modal's row builder, open/close, and submit handler (mirrors the source modal)**

Find:
```javascript
    const batch = writeBatch(db);
    items.forEach((it)=>{
      batch.set(doc(sourceRecordsCol), { date: sDate.value, source: it.source, qty: it.qty, createdAt: Date.now() });
    });
    batch.commit();
    closeSourceModal();
  });

  let incomeCategoryMode = 'cash';
```
Replace with:
```javascript
    const batch = writeBatch(db);
    items.forEach((it)=>{
      batch.set(doc(sourceRecordsCol), { date: sDate.value, source: it.source, qty: it.qty, createdAt: Date.now() });
    });
    batch.commit();
    closeSourceModal();
  });

  function createFleetItemRow(showLabels){
    const row = document.createElement('div');
    row.className = 'technika-item-row';
    row.innerHTML =
      '<div class="field">' +
        (showLabels ? '<label>Техника</label>' : '') +
        '<select class="fi-select"></select>' +
        '<input type="text" class="fi-manual" placeholder="Введите название техники" style="display:none;margin-top:8px;">' +
      '</div>' +
      '<div class="field ti-qty-field">' +
        (showLabels ? '<label>Кол-во</label>' : '') +
        '<input type="number" class="fi-qty" min="1" step="1" value="1">' +
      '</div>' +
      '<button type="button" class="ti-remove" aria-label="Удалить">&times;</button>';
    const select = row.querySelector('.fi-select');
    populateSelect(select, state.equipmentTypes, MANUAL_OPTION, 'Заполнить вручную');
    const manualInput = row.querySelector('.fi-manual');
    select.addEventListener('change', ()=>{
      if(select.value === MANUAL_OPTION){
        manualInput.style.display = 'block';
        manualInput.focus();
      } else {
        manualInput.style.display = 'none';
        manualInput.value = '';
      }
    });
    row.querySelector('.ti-remove').addEventListener('click', ()=>{
      if(fleetItemsList.children.length > 1){
        row.remove();
      }
    });
    return row;
  }
  function resetFleetItems(){
    fleetItemsList.innerHTML = '';
    fleetItemsList.appendChild(createFleetItemRow(true));
  }
  function refreshFleetItemSelects(){
    document.querySelectorAll('#fleetItemsList .fi-select').forEach(sel=>{
      const current = sel.value;
      populateSelect(sel, state.equipmentTypes, MANUAL_OPTION, 'Заполнить вручную');
      if(Array.from(sel.options).some(o=>o.value===current)) sel.value = current;
    });
  }
  addFleetItemBtn.addEventListener('click', ()=>{
    fleetItemsList.appendChild(createFleetItemRow(false));
  });

  function openFleetModal(){
    flDate.value = todayISO();
    resetFleetItems();
    fleetModalOverlay.classList.add('open');
  }
  function closeFleetModal(){ fleetModalOverlay.classList.remove('open'); }
  openFleetModalBtn.addEventListener('click', openFleetModal);
  closeFleetModalBtn.addEventListener('click', closeFleetModal);
  fleetModalOverlay.addEventListener('click', (e)=>{ if(e.target === fleetModalOverlay) closeFleetModal(); });

  fleetForm.addEventListener('submit', (e)=>{
    e.preventDefault();
    if(!flDate.value) return;
    const rows = Array.from(fleetItemsList.querySelectorAll('.technika-item-row'));
    const items = [];
    for(const row of rows){
      const select = row.querySelector('.fi-select');
      const manualInput = row.querySelector('.fi-manual');
      const qtyInput = row.querySelector('.fi-qty');
      const val = resolveValue(select, manualInput, state.equipmentTypes);
      const qty = parseInt(qtyInput.value,10) || 0;
      if(val && qty > 0) items.push({ equipmentType: val, qty: qty });
    }
    if(items.length === 0){ alert('Добавьте технику и количество'); return; }
    const batch = writeBatch(db);
    items.forEach((it)=>{
      batch.set(doc(fleetRecordsCol), { date: flDate.value, equipmentType: it.equipmentType, qty: it.qty, createdAt: Date.now() });
    });
    batch.commit();
    closeFleetModal();
  });

  let incomeCategoryMode = 'cash';
```

- [ ] **Step 7: Close the fleet modal on Escape**

Find:
```javascript
  document.addEventListener('keydown', (e)=>{ if(e.key === 'Escape'){ closeModal(); closeCompareModal(); closeSourceModal(); } });
```
Replace with:
```javascript
  document.addEventListener('keydown', (e)=>{ if(e.key === 'Escape'){ closeModal(); closeCompareModal(); closeSourceModal(); closeFleetModal(); } });
```

- [ ] **Step 8: Wire `refreshFleetItemSelects()` into `refreshSelects()`**

Find:
```javascript
  function refreshSelects(){
    refreshExpenseItemSelects();
    refreshSourceItemSelects();
  }
```
Replace with:
```javascript
  function refreshSelects(){
    refreshExpenseItemSelects();
    refreshSourceItemSelects();
    refreshFleetItemSelects();
  }
```

- [ ] **Step 9: Add `getFilteredFleetRecords()`**

Find:
```javascript
  function getFilteredSourceRecords(){
    if(periodMode === 'month'){
      const mk = monthFilter.value;
      if(!mk) return state.sourceRecords;
      return state.sourceRecords.filter(r=> monthKey(r.date) === mk);
    }
    const from = dateFrom.value;
    const to = dateTo.value;
    return state.sourceRecords.filter(r=>{
      if(from && r.date < from) return false;
      if(to && r.date > to) return false;
      return true;
    });
  }

  function render(){
```
Replace with:
```javascript
  function getFilteredSourceRecords(){
    if(periodMode === 'month'){
      const mk = monthFilter.value;
      if(!mk) return state.sourceRecords;
      return state.sourceRecords.filter(r=> monthKey(r.date) === mk);
    }
    const from = dateFrom.value;
    const to = dateTo.value;
    return state.sourceRecords.filter(r=>{
      if(from && r.date < from) return false;
      if(to && r.date > to) return false;
      return true;
    });
  }

  function getFilteredFleetRecords(){
    if(periodMode === 'month'){
      const mk = monthFilter.value;
      if(!mk) return state.fleetRecords;
      return state.fleetRecords.filter(r=> monthKey(r.date) === mk);
    }
    const from = dateFrom.value;
    const to = dateTo.value;
    return state.fleetRecords.filter(r=>{
      if(from && r.date < from) return false;
      if(to && r.date > to) return false;
      return true;
    });
  }

  function render(){
```

- [ ] **Step 10: Rewrite the fleet chart's data computation to read from `fleetRecords`**

Find:
```javascript
    // --- chart 2: fleet usage (qty per equipment type) ---
    const fleetByType = {};
    incomeTx.forEach(t=>{
      if(t.items && t.items.length){
        t.items.forEach(item=>{
          if(CORE_FLEET_TYPES.includes(item.category)) fleetByType[item.category] = (fleetByType[item.category]||0) + (item.qty||0);
        });
      } else if(CORE_FLEET_TYPES.includes(t.category)){
        fleetByType[t.category] = (fleetByType[t.category]||0) + (t.qty||0);
      }
    });
    const fleetLabels = Object.keys(fleetByType);
    const fleetValues = Object.values(fleetByType);
    const fleetTotal = fleetValues.reduce((a,b)=>a+b,0);
    const fleetPercents = fleetValues.map(v=> fleetTotal ? Math.round(v/fleetTotal*100) : 0);
```
Replace with:
```javascript
    // --- chart 2: fleet usage (qty per equipment type) ---
    const filteredFleetRecords = getFilteredFleetRecords();
    const fleetByType = {};
    const otherFleetDetails = {};
    filteredFleetRecords.forEach(r=>{
      if(!r.equipmentType) return;
      const isCore = CORE_FLEET_TYPES.includes(r.equipmentType);
      const label = isCore ? r.equipmentType : 'Прочее';
      fleetByType[label] = (fleetByType[label]||0) + (r.qty||0);
      if(!isCore) otherFleetDetails[r.equipmentType] = (otherFleetDetails[r.equipmentType]||0) + (r.qty||0);
    });
    const fleetLabels = Object.keys(fleetByType);
    const fleetValues = Object.values(fleetByType);
    const fleetTotal = fleetValues.reduce((a,b)=>a+b,0);
    const fleetPercents = fleetValues.map(v=> fleetTotal ? Math.round(v/fleetTotal*100) : 0);
```

- [ ] **Step 11: Add the "Прочее" expandable sublist to the fleet list**

Find:
```javascript
    document.getElementById('fleetList').innerHTML = fleetLabels.length
      ? fleetLabels.map((label,i)=>
          '<li><div class="chart-list-row"><span class="name"><span class="dot" style="background:'+colorFor(i)+';"></span><span class="label-text">'+escapeHtml(label)+'</span><span class="percent">'+fleetPercents[i]+'%</span></span><span class="amount">'+fleetValues[i]+' шт</span></div></li>'
        ).join('')
      : '<li class="empty">Нет данных за период</li>';
```
Replace with:
```javascript
    document.getElementById('fleetList').innerHTML = fleetLabels.length
      ? fleetLabels.map((label,i)=>{
          const isOther = label === 'Прочее';
          const detailKeys = isOther ? Object.keys(otherFleetDetails) : [];
          const sublist = isOther && detailKeys.length
            ? '<ul class="chart-sublist">' + detailKeys.map(k=>
                '<li><span class="label-text">'+escapeHtml(k)+'</span><span class="amount">'+otherFleetDetails[k]+' шт</span></li>'
              ).join('') + '</ul>'
            : '';
          const rowClass = isOther && detailKeys.length ? ' expandable' : '';
          return '<li class="'+rowClass.trim()+'">'
            + '<div class="chart-list-row">'
            + '<span class="name"><span class="dot" style="background:'+colorFor(i)+';"></span><span class="label-text">'+escapeHtml(label)+'</span><span class="percent">'+fleetPercents[i]+'%</span></span>'
            + '<span class="amount">'+fleetValues[i]+' шт'+(rowClass ? '<span class="expand-arrow">&#9662;</span>' : '')+'</span>'
            + '</div>'
            + sublist
            + '</li>';
        }).join('')
      : '<li class="empty">Нет данных за период</li>';

    document.querySelectorAll('#fleetList li.expandable > .chart-list-row').forEach(row=>{
      row.addEventListener('click', ()=>{
        row.parentElement.classList.toggle('open');
      });
    });
```

- [ ] **Step 12: Verify locally**

Refresh `http://localhost:8080/` (server from Task 1 is still running — if not, restart it as in Task 1 Step 11). On the "Учет проката техники" panel, click the new "+" button, pick a date, choose a equipment type and quantity, submit.

Expected: the record appears on the "Учет проката техники" chart right away (a slice with that equipment type and its quantity).

- [ ] **Step 13: Commit**

```bash
git add index.html
git commit -m "Move fleet tracking to its own Firestore log and chart"
```

---

### Task 3: Deploy and verify on GitHub Pages

**Files:** none (deployment only)

- [ ] **Step 1: Push**

```bash
cd "/Users/aleksej/Desktop/Мотопрокат учет"
git push
```

- [ ] **Step 2: Verify live**

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://ialexrd.github.io/adrenaline43/"
```
Expected: `HTTP 200`. Then ask the user to open the live site, add one "Наличные" income record and one fleet record, and confirm both show up (same two checks as Task 1 Step 11 and Task 2 Step 12, just on the live URL instead of localhost).

---

## Notes

- The historical/test income entries created earlier in this project will now render under "Прочее" on the "Расходы/Доходы" chart (they have no `Наличные`/`Безнал` category) — this is expected, not a bug.
- The fleet chart starts empty — old equipment counts are not backfilled (confirmed with the user).
