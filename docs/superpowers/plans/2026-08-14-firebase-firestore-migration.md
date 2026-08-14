# Firestore + Auth Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace `localStorage`-based state in `index.html` with Firebase Auth (single shared login) + Firestore (realtime, multi-device), so the rental-accounting app works identically from any device.

**Architecture:** `index.html` stays a single static file with no build step. Firebase Web SDK is loaded via ESM CDN imports directly in the existing `<script>` (converted to `type="module"`). A login screen gates the app until `onAuthStateChanged` confirms a session. All reads/writes that used to touch `localStorage` are replaced with Firestore `onSnapshot` listeners (realtime, populate the existing `state` object) and `addDoc`/`deleteDoc`/`writeBatch`/`setDoc` writes.

**Tech Stack:** Vanilla JS, Firebase JS SDK v10 (Auth + Firestore) via `https://www.gstatic.com/firebasejs/10.12.2/...` ESM imports, GitHub Pages hosting.

## Global Constraints

- Firebase SDK version pinned to **10.12.2** everywhere (three import URLs must all use the same version string).
- Firestore collections/doc names are fixed by the approved spec (`docs/superpowers/specs/2026-08-14-firebase-sheets-migration-design.md`): `transactions` (collection), `sourceRecords` (collection), `settings/config` (single document).
- Firestore security rules are already published: `allow read, write: if request.auth != null;` — do not change them in this plan.
- One Firebase Auth user (email/password) already exists in the console — this plan does not create users.
- `firebaseConfig` values (below) are public/non-secret by design; they're committed directly in `index.html`.
- **No automated test framework exists in this project** (static HTML/JS, no build/test tooling, out of scope to add one per the approved spec). Every "test" step in this plan is a concrete manual verification in the browser + Firebase Console — follow the exact steps given, don't skip them.
- Local testing must use an HTTP server (`python3 -m http.server`), not `file://` — ES module imports and Firebase Auth behave unreliably over `file://`.
- This plan covers **Stage A only** (Firestore + Auth). Stage B (Cloud Function → Google Sheets sync) is a separate follow-up plan, created after Stage A is verified working in production.

```
firebaseConfig = {
  apiKey: "AIzaSyDUt6MAWdEUJtSKaYSQUQRmKdEl6o4wZIQ",
  authDomain: "adrenaline43.firebaseapp.com",
  projectId: "adrenaline43",
  storageBucket: "adrenaline43.firebasestorage.app",
  messagingSenderId: "694839019905",
  appId: "1:694839019905:web:2d99e7e080808ca3c98080",
  measurementId: "G-BPCB8G3G4X"
}
```

---

### Task 1: Login screen + Firebase Auth gate

**Files:**
- Modify: `index.html:946-948` (body opening — add login screen markup, give `.shell` an id, hide it by default)
- Modify: `index.html:950-956` (header — add logout button)
- Modify: `index.html:1218-1220` (script tag — add `type="module"`, Firebase imports, config, auth wiring)

**Interfaces:**
- Produces: module-scope `auth` (Firebase Auth instance), `db` (Firestore instance), and inside the app IIFE a forward-referenced call to `startFirestoreSync()` (defined in Task 2 — safe because `async function startFirestoreSync(){...}` is a hoisted function declaration in the same scope).

- [ ] **Step 1: Add login screen markup and hide the app shell by default**

In `index.html`, find:
```html
<body>

<div class="shell">
```
Replace with:
```html
<body>

<div class="modal-overlay open" id="loginScreen">
  <div class="modal">
    <div class="modal-head">
      <h2>Вход</h2>
    </div>
    <form id="loginForm">
      <div class="field">
        <label for="loginEmail">Email</label>
        <input type="email" id="loginEmail" required autocomplete="username">
      </div>
      <div class="field">
        <label for="loginPassword">Пароль</label>
        <input type="password" id="loginPassword" required autocomplete="current-password">
      </div>
      <p id="loginError" style="display:none;color:var(--coral);font-size:13px;margin:0 0 14px;"></p>
      <button type="submit" class="btn-submit">Войти</button>
    </form>
  </div>
</div>

<div class="shell" id="shell" style="display:none;">
```

- [ ] **Step 2: Add a logout button to the header**

Find:
```html
  <header class="top">
    <div>
      <p class="brand-eyebrow">Журнал операций</p>
      <h1>Учет проката мототехники</h1>
    </div>
    <button type="button" class="btn-new btn-new-desktop open-modal-trigger">+ Новая запись</button>
  </header>
```
Replace with:
```html
  <header class="top">
    <div>
      <p class="brand-eyebrow">Журнал операций</p>
      <h1>Учет проката мототехники</h1>
    </div>
    <div style="display:flex;gap:10px;align-items:center;">
      <button type="button" class="btn-new btn-new-desktop open-modal-trigger">+ Новая запись</button>
      <button type="button" id="logoutBtn" class="ghost-btn">Выйти</button>
    </div>
  </header>
```

- [ ] **Step 3: Convert the app script to a module and wire up Firebase Auth**

Find:
```html
<script>
(function(){
  const STORAGE_KEY = 'prokat_uchet_v1';
```
Replace with:
```html
<script type="module">
import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js';
import {
  getAuth, onAuthStateChanged, signInWithEmailAndPassword, signOut
} from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js';
import {
  getFirestore, collection, doc, getDoc, setDoc, addDoc, deleteDoc,
  writeBatch, onSnapshot, getDocs
} from 'https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js';

const firebaseConfig = {
  apiKey: "AIzaSyDUt6MAWdEUJtSKaYSQUQRmKdEl6o4wZIQ",
  authDomain: "adrenaline43.firebaseapp.com",
  projectId: "adrenaline43",
  storageBucket: "adrenaline43.firebasestorage.app",
  messagingSenderId: "694839019905",
  appId: "1:694839019905:web:2d99e7e080808ca3c98080",
  measurementId: "G-BPCB8G3G4X"
};
const firebaseApp = initializeApp(firebaseConfig);
const auth = getAuth(firebaseApp);
const db = getFirestore(firebaseApp);

(function(){
  const loginScreen = document.getElementById('loginScreen');
  const shellEl = document.getElementById('shell');
  const loginForm = document.getElementById('loginForm');
  const loginEmail = document.getElementById('loginEmail');
  const loginPassword = document.getElementById('loginPassword');
  const loginError = document.getElementById('loginError');
  const logoutBtn = document.getElementById('logoutBtn');

  loginForm.addEventListener('submit', (e)=>{
    e.preventDefault();
    loginError.style.display = 'none';
    signInWithEmailAndPassword(auth, loginEmail.value.trim(), loginPassword.value)
      .catch(()=>{
        loginError.textContent = 'Неверный email или пароль';
        loginError.style.display = 'block';
      });
  });
  logoutBtn.addEventListener('click', ()=> signOut(auth));

  onAuthStateChanged(auth, (user)=>{
    if(user){
      loginScreen.classList.remove('open');
      shellEl.style.display = '';
      loginForm.reset();
      startFirestoreSync(); // defined below in this same IIFE (Task 2) — hoisted, safe to call here
    } else {
      loginScreen.classList.add('open');
      shellEl.style.display = 'none';
      stopFirestoreSync(); // defined below in this same IIFE (Task 2) — hoisted, safe to call here
    }
  });

  const STORAGE_KEY = 'prokat_uchet_v1';
```

- [ ] **Step 4: Verify locally**

Run:
```bash
cd "/Users/aleksej/Desktop/Мотопрокат учет"
python3 -m http.server 8080
```
Open `http://localhost:8080/` in a browser.

Expected: you see only the "Вход" (login) card — the rest of the app is invisible. Open the browser console (F12) — there should be **no red errors** about failed imports or `startFirestoreSync is not defined` (a "not defined" error at this point is expected to be silent until you actually log in, since it's only called inside the auth callback — if it throws when you submit login, that's fine for now, Task 2 fixes it).

Log in with the email/password you created in Firebase Console. Expected: login card disappears, the app shell appears (it will look empty/broken since Firestore wiring isn't done yet — that's expected, Task 2 fixes it). Click "Выйти" — expected: login card reappears.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add Firebase Auth login gate"
```

---

### Task 2: Firestore data layer — settings, transactions, sourceRecords

This is one task because every mutation site in the file shares the same `state` object and the same removed `save()` function — splitting it further would leave the app in a broken intermediate state (calls to a function that no longer exists).

**Files:**
- Modify: `index.html:1234-1292` (remove `seedData`/`load`/`save`, add Firestore refs + `startFirestoreSync`)
- Modify: `index.html` `resolveValue` function (persist new manual category/source/payment values to Firestore)
- Modify: `index.html` `sourceForm` submit handler
- Modify: `index.html` `entryForm` submit handler
- Modify: `index.html` `clearBtn` click handler
- Modify: `index.html` `deleteRecord` function
- Modify: `index.html` `render()` — fix id-based sort and delete-button wiring (Firestore doc IDs are strings, not numbers)

**Interfaces:**
- Consumes: `auth`, `db`, `collection`, `doc`, `getDoc`, `setDoc`, `addDoc`, `deleteDoc`, `writeBatch`, `onSnapshot`, `getDocs` (imported in Task 1, module scope).
- Produces: `startFirestoreSync()` and `stopFirestoreSync()` (called from Task 1's `onAuthStateChanged`), `state` object with the same shape as before (`equipmentTypes`, `otherIncomeCategories`, `expenseCategories`, `paymentMethods`, `sources`, `transactions`, `sourceRecords`), `saveSettings()`.

- [ ] **Step 1: Replace `seedData`/`load`/`save` with Firestore refs and the sync bootstrap**

Find (the entire block from `function seedData(){` through `let state = load();`):
```javascript
  function seedData(){
    return {
      equipmentTypes: ['Эндуро','Питбайк','Квадроцикл','Снегоход'],
      otherIncomeCategories: ['Поломки'],
      expenseCategories: ['ГСМ','Реклама','Аренда','ЗП работника','Запчасти','Расходники'],
      paymentMethods: ['Нал','Безнал','Сертификат'],
      sources: ['ВКонтакте','Инст','Авито','Яндекс','Рекомендация'],
      transactions: [
        {id:1, date:'2026-08-05', type:'income', status:'completed', category:'Питбайк', qty:1, amount:10000, payment:'Нал', source:'Яндекс', client:'Григорий', prepayment:3000},
        {id:2, date:'2026-08-05', type:'expense', status:'completed', category:'ГСМ', qty:1, amount:2500, payment:'', source:''},
        {id:3, date:'2026-08-06', type:'income', status:'completed', category:'Эндуро', qty:2, amount:24000, payment:'Безнал', source:'ВК', client:'', prepayment:0},
        {id:4, date:'2026-08-06', type:'expense', status:'completed', category:'Реклама', qty:1, amount:15000, payment:'', source:''},
        {id:5, date:'2026-08-07', type:'income', status:'completed', category:'Квадроцикл', qty:1, amount:18000, payment:'Сертификат', source:'Инст', client:'', prepayment:0},
        {id:6, date:'2026-08-07', type:'expense', status:'completed', category:'Запчасти', qty:1, amount:3000, payment:'', source:''},
        {id:7, date:'2026-08-09', type:'income', status:'completed', category:'Предоплата', qty:0, amount:5000, payment:'', source:'', client:'Александр, +7 900 123-45-67', prepayment:5000}
      ],
      sourceRecords: [
        {id:1, date:'2026-08-05', source:'ВКонтакте', qty:3},
        {id:2, date:'2026-08-06', source:'Инст', qty:2},
        {id:3, date:'2026-08-07', source:'Яндекс', qty:1},
        {id:4, date:'2026-08-08', source:'Рекомендация', qty:1}
      ]
    };
  }

  function load(){
    try{
      const raw = localStorage.getItem(STORAGE_KEY);
      if(!raw) return seedData();
      const parsed = JSON.parse(raw);
      if(!parsed.transactions) return seedData();
      if(!parsed.sources) parsed.sources = ['ВКонтакте','Инст','Авито','Яндекс','Рекомендация'];
      if(!parsed.sourceRecords) parsed.sourceRecords = [];
      if(!parsed.otherIncomeCategories) parsed.otherIncomeCategories = ['Поломки'];
      if(parsed.equipmentTypes.includes('Поломки')){
        parsed.equipmentTypes = parsed.equipmentTypes.filter(c=> c !== 'Поломки');
        if(!parsed.otherIncomeCategories.includes('Поломки')) parsed.otherIncomeCategories.push('Поломки');
      }
      if(parsed.expenseCategories.includes('Зарплата')){
        parsed.expenseCategories = parsed.expenseCategories.map(c=> c === 'Зарплата' ? 'ЗП работника' : c);
      }
      if(!parsed.expenseCategories.includes('ЗП работника')) parsed.expenseCategories.push('ЗП работника');
      parsed.transactions.forEach(t=>{
        if(t.category === 'Зарплата') t.category = 'ЗП работника';
        if(t.status === 'prepaid'){
          t.status = 'completed';
          t.category = 'Предоплата';
        }
      });
      return parsed;
    }catch(e){
      return seedData();
    }
  }
  function save(){
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
  }

  let state = load();
```
Replace with:
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

  function saveSettings(){
    setDoc(settingsRef, {
      equipmentTypes: state.equipmentTypes,
      otherIncomeCategories: state.otherIncomeCategories,
      expenseCategories: state.expenseCategories,
      paymentMethods: state.paymentMethods,
      sources: state.sources
    });
  }

  async function startFirestoreSync(){
    if(syncStarted) return;
    syncStarted = true;
    const settingsSnap = await getDoc(settingsRef);
    if(!settingsSnap.exists()){
      await setDoc(settingsRef, {
        equipmentTypes: ['Эндуро','Питбайк','Квадроцикл','Снегоход'],
        otherIncomeCategories: ['Поломки'],
        expenseCategories: ['ГСМ','Реклама','Аренда','ЗП работника','Запчасти','Расходники'],
        paymentMethods: ['Нал','Безнал','Сертификат'],
        sources: ['ВКонтакте','Инст','Авито','Яндекс','Рекомендация']
      });
    }
    unsubscribers.push(onSnapshot(settingsRef, (snap)=>{
      const data = snap.data() || {};
      state.equipmentTypes = data.equipmentTypes || [];
      state.otherIncomeCategories = data.otherIncomeCategories || [];
      state.expenseCategories = data.expenseCategories || [];
      state.paymentMethods = data.paymentMethods || [];
      state.sources = data.sources || [];
      render();
    }));
    unsubscribers.push(onSnapshot(transactionsCol, (snap)=>{
      state.transactions = snap.docs.map(d=> ({ id: d.id, ...d.data() }));
      render();
    }));
    unsubscribers.push(onSnapshot(sourceRecordsCol, (snap)=>{
      state.sourceRecords = snap.docs.map(d=> ({ id: d.id, ...d.data() }));
      render();
    }));
  }

  function stopFirestoreSync(){
    unsubscribers.forEach(fn=> fn());
    unsubscribers = [];
    syncStarted = false;
  }
```

- [ ] **Step 2: Persist newly-typed manual category/source/payment values to Firestore**

Find:
```javascript
  function resolveValue(select, manualInput, list){
    if(select.value === MANUAL_OPTION){
      const val = (manualInput.value || '').trim();
      if(val && !list.includes(val)) list.push(val);
      return val;
    }
    return select.value;
  }
```
Replace with:
```javascript
  function resolveValue(select, manualInput, list){
    if(select.value === MANUAL_OPTION){
      const val = (manualInput.value || '').trim();
      if(val && !list.includes(val)){
        list.push(val);
        saveSettings();
      }
      return val;
    }
    return select.value;
  }
```

- [ ] **Step 3: Write source records to Firestore instead of `state.sourceRecords`**

Find:
```javascript
  sourceForm.addEventListener('submit', (e)=>{
    e.preventDefault();
    if(!sDate.value) return;
    const rows = Array.from(sourceItemsList.querySelectorAll('.technika-item-row'));
    const items = [];
    for(const row of rows){
      const select = row.querySelector('.si-select');
      const manualInput = row.querySelector('.si-manual');
      const qtyInput = row.querySelector('.si-qty');
      const val = resolveValue(select, manualInput, state.sources);
      const qty = parseInt(qtyInput.value,10) || 0;
      if(val && qty > 0) items.push({ source: val, qty: qty });
    }
    if(items.length === 0){ alert('Добавьте источник и количество'); return; }
    items.forEach((it, idx)=>{
      state.sourceRecords.push({ id: Date.now() + idx, date: sDate.value, source: it.source, qty: it.qty });
    });
    save();
    closeSourceModal();
    render();
  });
```
Replace with:
```javascript
  sourceForm.addEventListener('submit', (e)=>{
    e.preventDefault();
    if(!sDate.value) return;
    const rows = Array.from(sourceItemsList.querySelectorAll('.technika-item-row'));
    const items = [];
    for(const row of rows){
      const select = row.querySelector('.si-select');
      const manualInput = row.querySelector('.si-manual');
      const qtyInput = row.querySelector('.si-qty');
      const val = resolveValue(select, manualInput, state.sources);
      const qty = parseInt(qtyInput.value,10) || 0;
      if(val && qty > 0) items.push({ source: val, qty: qty });
    }
    if(items.length === 0){ alert('Добавьте источник и количество'); return; }
    const batch = writeBatch(db);
    items.forEach((it)=>{
      batch.set(doc(sourceRecordsCol), { date: sDate.value, source: it.source, qty: it.qty, createdAt: Date.now() });
    });
    batch.commit();
    closeSourceModal();
  });
```

- [ ] **Step 4: Write transactions (income + expense) to Firestore instead of `state.transactions`**

Find:
```javascript
  entryForm.addEventListener('submit', (e)=>{
    e.preventDefault();
    if(!fDate.value) return;

    if(currentType === 'expense'){
      const rows = Array.from(expenseItemsList.querySelectorAll('.expense-item-row'));
      const items = [];
      for(const row of rows){
        const select = row.querySelector('.ei-select');
        const manualInput = row.querySelector('.ei-manual');
        const amountInput = row.querySelector('.ei-amount');
        const activePartnerBtn = row.querySelector('.ei-partner-toggle button.active');
        const categoryVal = resolveValue(select, manualInput, state.expenseCategories);
        const amountVal = parseFloat(amountInput.value);
        const partnerVal = activePartnerBtn ? activePartnerBtn.dataset.partner : '';
        if(categoryVal && !isNaN(amountVal) && amountVal > 0){
          items.push({ category: categoryVal, amount: amountVal, partner: partnerVal });
        }
      }
      if(items.length === 0){ alert('Добавьте расход, категорию и сумму'); return; }
      items.forEach((it, idx)=>{
        state.transactions.push({
          id: Date.now() + idx, date: fDate.value, type:'expense', status:'completed',
          amount: it.amount, category: it.category, qty:1, payment:'', source:'',
          client:'', prepayment:0, partner: it.partner
        });
      });
      save();
      resetEntryForm();
      closeModal();
      render();
      return;
    }

    // income
    let record;
    if(incomeCategoryMode === 'prepay'){
      const prepaymentVal = parseFloat(fPrepayment.value);
      const hasPrepayment = !isNaN(prepaymentVal) && prepaymentVal > 0;
      const clientName = (fClient.value || '').trim();
      if(!clientName){ alert('Укажите имя или номер клиента'); return; }
      if(!hasPrepayment){ alert('Укажите сумму предоплаты'); return; }
      record = {
        id: Date.now(), date: fDate.value, type:'income', status:'completed',
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
        id: Date.now(), date: fDate.value, type:'income', status:'completed',
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
        id: Date.now(), date: fDate.value, type:'income', status:'completed',
        category: categoryVal, qty: 0,
        payment: paymentVal, source: '',
        client: '',
        prepayment: 0,
        amount: amountVal
      };
    }
    state.transactions.push(record);
    save();
    resetEntryForm();
    closeModal();
    render();
  });
```
Replace with:
```javascript
  entryForm.addEventListener('submit', (e)=>{
    e.preventDefault();
    if(!fDate.value) return;

    if(currentType === 'expense'){
      const rows = Array.from(expenseItemsList.querySelectorAll('.expense-item-row'));
      const items = [];
      for(const row of rows){
        const select = row.querySelector('.ei-select');
        const manualInput = row.querySelector('.ei-manual');
        const amountInput = row.querySelector('.ei-amount');
        const activePartnerBtn = row.querySelector('.ei-partner-toggle button.active');
        const categoryVal = resolveValue(select, manualInput, state.expenseCategories);
        const amountVal = parseFloat(amountInput.value);
        const partnerVal = activePartnerBtn ? activePartnerBtn.dataset.partner : '';
        if(categoryVal && !isNaN(amountVal) && amountVal > 0){
          items.push({ category: categoryVal, amount: amountVal, partner: partnerVal });
        }
      }
      if(items.length === 0){ alert('Добавьте расход, категорию и сумму'); return; }
      const batch = writeBatch(db);
      items.forEach((it)=>{
        batch.set(doc(transactionsCol), {
          date: fDate.value, type:'expense', status:'completed',
          amount: it.amount, category: it.category, qty:1, payment:'', source:'',
          client:'', prepayment:0, partner: it.partner, createdAt: Date.now()
        });
      });
      batch.commit();
      resetEntryForm();
      closeModal();
      return;
    }

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

- [ ] **Step 5: Batch-delete Firestore docs on "Очистить все данные" (settings stay untouched)**

Find:
```javascript
  document.getElementById('clearBtn').addEventListener('click', ()=>{
    if(confirm('Удалить все записи безвозвратно?')){
      state = {equipmentTypes: state.equipmentTypes, otherIncomeCategories: state.otherIncomeCategories, expenseCategories: state.expenseCategories, paymentMethods: state.paymentMethods, sources: state.sources, transactions: [], sourceRecords: []};
      save();
      render();
    }
  });
```
Replace with:
```javascript
  document.getElementById('clearBtn').addEventListener('click', async ()=>{
    if(!confirm('Удалить все записи безвозвратно?')) return;
    const [txSnap, srcSnap] = await Promise.all([getDocs(transactionsCol), getDocs(sourceRecordsCol)]);
    const allDocs = [...txSnap.docs, ...srcSnap.docs];
    for(let i = 0; i < allDocs.length; i += 450){
      const batch = writeBatch(db);
      allDocs.slice(i, i + 450).forEach(d=> batch.delete(d.ref));
      await batch.commit();
    }
  });
```

- [ ] **Step 6: Delete a single transaction via Firestore**

Find:
```javascript
  function deleteRecord(id){
    state.transactions = state.transactions.filter(t=> t.id !== id);
    save();
    render();
  }
```
Replace with:
```javascript
  function deleteRecord(id){
    deleteDoc(doc(transactionsCol, id));
  }
```

- [ ] **Step 7: Fix id-based sort and delete-button wiring (Firestore doc IDs are strings, not numbers)**

Find:
```javascript
    const sorted = filtered.slice().sort((a,b)=> b.date.localeCompare(a.date) || b.id-a.id);
```
Replace with:
```javascript
    const sorted = filtered.slice().sort((a,b)=> b.date.localeCompare(a.date) || (b.createdAt||0)-(a.createdAt||0));
```

Find:
```javascript
        btn.addEventListener('click', ()=> deleteRecord(Number(btn.dataset.id)));
```
Replace with:
```javascript
        btn.addEventListener('click', ()=> deleteRecord(btn.dataset.id));
```

- [ ] **Step 8: Verify locally — full CRUD + realtime sync**

Run:
```bash
cd "/Users/aleksej/Desktop/Мотопрокат учет"
python3 -m http.server 8080
```
Open two browser tabs at `http://localhost:8080/` and log in on both.

1. In Firebase Console → Firestore Database → Data: after logging in, a `settings` collection with a `config` document should appear, containing the default `equipmentTypes`/`expenseCategories`/`paymentMethods`/`sources`/`otherIncomeCategories` arrays.
2. In tab 1, add an income entry (Техника → any type, qty 1, any amount) and submit. Expected: it appears in the ledger table in **both tabs** within ~1 second, without refreshing. Expected: a new document appears in the `transactions` collection in Firestore Console.
3. In tab 1, add an expense entry. Expected: same realtime behavior, new doc in `transactions` with `type: 'expense'`.
4. Open "Источник клиента" modal, add a source record. Expected: new doc in `sourceRecords` collection, both tabs update.
5. In the entry form, choose "Заполнить вручную" for Техника and type a brand-new equipment name (e.g. "Тест-байк"), submit. Expected: `settings/config.equipmentTypes` in Firestore Console now includes "Тест-байк", and it appears in the dropdown in tab 2 without refreshing.
6. Click the × delete button on a row in tab 1. Expected: it disappears from both tabs and from the `transactions` collection in Firestore Console.
7. Click "Очистить все данные", confirm. Expected: `transactions` and `sourceRecords` collections become empty in Firestore Console; the `settings/config` document is untouched (still has your custom "Тест-байк").
8. In tab 1, click "Выйти", then log back in without reloading the page. Expected: the app shows your data again (not a blank/broken state) — this confirms `stopFirestoreSync`/`startFirestoreSync` correctly re-subscribe.

If any step doesn't match, stop and debug before proceeding — check the browser console for errors first.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Replace localStorage with Firestore realtime sync"
```

---

### Task 3: Deploy and verify on GitHub Pages

**Files:** none (deployment + verification only)

- [ ] **Step 1: Authorize the GitHub Pages domain in Firebase Auth**

In Firebase Console → Authentication → Settings → Authorized domains → Add domain → enter `ialexrd.github.io`. Without this step, login on the live site will fail with an `auth/unauthorized-domain` error (it will still work on `localhost`, which Firebase authorizes automatically).

- [ ] **Step 2: Push to GitHub**

```bash
cd "/Users/aleksej/Desktop/Мотопрокат учет"
git push
```

- [ ] **Step 3: Verify on the live site**

Wait ~60 seconds for GitHub Pages to redeploy, then check:
```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://ialexrd.github.io/adrenaline43/"
```
Expected: `HTTP 200`.

Open `https://ialexrd.github.io/adrenaline43/` in a browser (ideally on a phone, to confirm cross-device access). Log in, add a test entry, confirm it shows up. Then open the same URL on a second device (or the desktop browser) and confirm the entry is visible there too — this is the actual goal of the whole migration.

- [ ] **Step 4: Clean up test data**

If you added test entries ("Тест-байк", test transactions) while verifying, delete them now via the UI's × buttons or "Очистить все данные", so the production data starts clean.

---

## Next steps (not in this plan)

Stage B — Cloud Function trigger that mirrors new transactions/sourceRecords into a Google Sheet — is a separate plan, to be written once Stage A is confirmed working live. It needs its own setup (Blaze billing plan, a Node.js Cloud Functions project, a Google Cloud service account, Sheets API enablement) that's independent of everything in this plan.
