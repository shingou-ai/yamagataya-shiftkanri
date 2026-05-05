# JP/EN 言語切替機能 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** スタッフ画面をJA/EN切替できるようにし、英語モードでの申請備考はGoogle翻訳APIで日本語に変換して保存する。

**Architecture:** 単一HTMLファイル内に I18N 辞書と `t()` 関数を追加し、静的要素には `data-i18n` 属性、動的レンダリングには `t()` 呼び出しを使う。言語設定は `localStorage('ymLang')` で保持。管理者画面は対象外。

**Tech Stack:** Vanilla JS, Firebase Realtime Database, Google Cloud Translation API v2

---

## ファイル構成

変更対象ファイルは `index.html` のみ。以下のセクションを追加・変更する：

| 対象箇所 | 変更内容 |
|---|---|
| `<style>` ブロック | `.lang-toggle-btn` CSS 追加 |
| ヘッダー HTML (line ~512) | JA/EN ボタン追加 |
| ログイン画面 HTML (line ~484) | `data-i18n` 属性追加 |
| スタッフナビ HTML (line ~548, 1077) | `data-i18n` 属性追加 |
| `const WEEKDAYS` (line 1532) | 英語曜日用定数追加 |
| `const UNAVAIL_REASONS` (line 1853) | キー配列に変更 |
| JS スクリプト冒頭 (line ~1530) | I18N 辞書・コア関数追加 |
| `renderMyShift()` (line ~6522) | `t()` / `shiftTypeLabel()` 対応 |
| `renderUnavailRequest()` (line ~2847) | `t()` 対応・理由セレクトキー化 |
| `submitUnavailRequest()` (line ~3047) | キー保存・自動翻訳統合 |
| `renderUnavailManage()` (line ~3095) | `t_ja()` で理由表示 |

---

## Task 1: i18n コアシステム（辞書・関数・定数）

**Files:**
- Modify: `index.html` — `const WEEKDAYS` の直後（line 1532 付近）に挿入

- [ ] **Step 1: I18N辞書・コア関数をWEEKDAYSの直後に追加する**

`const WEEKDAYS = ['日','月','火','水','木','金','土'];` の直後に以下を追加：

```javascript
// ===== i18n =====
let currentLang = (() => { try { return localStorage.getItem('ymLang') || 'ja'; } catch(e) { return 'ja'; } })();

const I18N = {
  ja: {
    select_role:'ログイン区分を選択してください', role_admin:'管理者', role_staff:'スタッフ',
    login_id:'ID', login_password:'パスワード', login_btn:'ログイン',
    login_error:'IDまたはパスワードが正しくありません',
    logout_btn:'ログアウト', badge_staff:'スタッフ',
    nav_my_shift:'マイシフト', nav_all_shift:'全体シフト表',
    nav_shift_req:'休み希望申請', nav_unavail_req:'勤務希望申請',
    my_shift_title:'マイシフト', all_shift_title:'全体シフト表',
    shift_req_title:'休み希望申請',
    weekday_sun:'日', weekday_mon:'月', weekday_tue:'火', weekday_wed:'水',
    weekday_thu:'木', weekday_fri:'金', weekday_sat:'土',
    shift_kokyu:'公休', shift_yukyu:'有休', shift_kibou:'希望休', shift_kekkin:'欠勤',
    half_am_kokyu:'半休AM（公休）', half_pm_kokyu:'半休PM（公休）',
    half_am_yukyu:'半休AM（有休）', half_pm_yukyu:'半休PM（有休）',
    unavail_title:'勤務希望申請',
    unavail_remaining:'残り申請回数', unavail_used:'申請済み（承認待含む）', unavail_limit:'月の上限',
    unavail_deadline:'締切', unavail_deadline_passed:'（締切済み）',
    unavail_date_label:'希望日', unavail_pattern_label:'希望の勤務パターン',
    unavail_reason_label:'希望理由', unavail_note_label:'備考（任意・日本語で記入）',
    unavail_submit_btn:'申請する', unavail_form_closed:'申請期限が過ぎています',
    unavail_status_pending:'承認待ち', unavail_status_approved:'承認',
    unavail_status_rejected:'却下', unavail_status_cancelled:'取消',
    reason_private:'📅 私的な予定がある', reason_medical:'🏥 通院・医療',
    reason_ceremony:'💒 冠婚葬祭', reason_school:'🏫 子の学校行事',
    reason_family:'👨‍👩‍👦 家族の介護・育児', reason_other:'📋 その他',
    toast_select_date:'希望日を選択してください',
    toast_select_pattern:'希望の勤務パターンを選択してください',
    toast_select_reason:'希望理由を選択してください',
    toast_other_required:'「その他」を選択した場合は理由を記入してください',
    toast_past_date:'過去・当日の日付は申請できません',
    toast_already_applied:'この日はすでに申請済みです',
    toast_submitted:'勤務希望申請を提出しました ✅',
    toast_translating:'備考を翻訳中...',
  },
  en: {
    select_role:'Please select your role', role_admin:'Admin', role_staff:'Staff',
    login_id:'ID', login_password:'Password', login_btn:'Login',
    login_error:'Incorrect ID or password',
    logout_btn:'Logout', badge_staff:'Staff',
    nav_my_shift:'My Shift', nav_all_shift:'All Shifts',
    nav_shift_req:'Request Day Off', nav_unavail_req:'Work Preference',
    my_shift_title:'My Shift', all_shift_title:'All Shifts',
    shift_req_title:'Day Off Request',
    weekday_sun:'Sun', weekday_mon:'Mon', weekday_tue:'Tue', weekday_wed:'Wed',
    weekday_thu:'Thu', weekday_fri:'Fri', weekday_sat:'Sat',
    shift_kokyu:'Day Off', shift_yukyu:'Paid Leave', shift_kibou:'Requested Off', shift_kekkin:'Absent',
    half_am_kokyu:'Half Day AM (Day Off)', half_pm_kokyu:'Half Day PM (Day Off)',
    half_am_yukyu:'Half Day AM (Paid Leave)', half_pm_yukyu:'Half Day PM (Paid Leave)',
    unavail_title:'Work Preference Request',
    unavail_remaining:'Requests Remaining', unavail_used:'Submitted (incl. pending)', unavail_limit:'Monthly Limit',
    unavail_deadline:'Deadline', unavail_deadline_passed:'(Deadline Passed)',
    unavail_date_label:'Preferred Date', unavail_pattern_label:'Preferred Work Pattern',
    unavail_reason_label:'Reason', unavail_note_label:'Notes (optional, any language)',
    unavail_submit_btn:'Submit', unavail_form_closed:'The application period has ended',
    unavail_status_pending:'Pending', unavail_status_approved:'Approved',
    unavail_status_rejected:'Rejected', unavail_status_cancelled:'Cancelled',
    reason_private:'📅 Personal plans', reason_medical:'🏥 Medical appointment',
    reason_ceremony:'💒 Wedding / Funeral', reason_school:"🏫 Child's school event",
    reason_family:'👨‍👩‍👦 Family care / Childcare', reason_other:'📋 Other',
    toast_select_date:'Please select a preferred date',
    toast_select_pattern:'Please select a work pattern',
    toast_select_reason:'Please select a reason',
    toast_other_required:'Please describe the reason when selecting "Other"',
    toast_past_date:"Cannot apply for today's or a past date",
    toast_already_applied:'You have already applied for this date',
    toast_submitted:'Work preference submitted ✅',
    toast_translating:'Translating notes...',
  }
};

function t(key) { return I18N[currentLang]?.[key] ?? I18N.ja[key] ?? key; }
function t_ja(key) { return I18N.ja[key] ?? key; }

const SHIFT_TYPE_I18N = {
  '公休': 'shift_kokyu', '有休': 'shift_yukyu', '希望休': 'shift_kibou', '欠勤': 'shift_kekkin',
};
const HALF_DAY_I18N = {
  'AM_kokyu': 'half_am_kokyu', 'PM_kokyu': 'half_pm_kokyu',
  'AM_yukyu': 'half_am_yukyu', 'PM_yukyu': 'half_pm_yukyu',
};
function shiftTypeLabel(name) {
  const key = SHIFT_TYPE_I18N[name];
  return key ? t(key) : name;
}
function halfDayLabel(flag) {
  const key = HALF_DAY_I18N[flag];
  return key ? t(key) : flag;
}

const WEEKDAYS_I18N = ['weekday_sun','weekday_mon','weekday_tue','weekday_wed','weekday_thu','weekday_fri','weekday_sat'];
function weekdayLabel(index) { return t(WEEKDAYS_I18N[index]); }

function applyI18n() {
  document.querySelectorAll('[data-i18n]').forEach(el => {
    el.textContent = t(el.dataset.i18n);
  });
}
function updateLangBtn() {
  const ja = document.getElementById('lang-label-ja');
  const en = document.getElementById('lang-label-en');
  if (!ja || !en) return;
  ja.style.fontWeight = currentLang === 'ja' ? '700' : '400';
  ja.style.opacity    = currentLang === 'ja' ? '1'   : '0.5';
  en.style.fontWeight = currentLang === 'en' ? '700' : '400';
  en.style.opacity    = currentLang === 'en' ? '1'   : '0.5';
}
function rerenderStaff() {
  applyI18n();
  const pid = document.querySelector('.page.active')?.id;
  if (pid === 'page-my-shift')        renderMyShift();
  if (pid === 'page-all-shift')       renderAllShiftTable();
  if (pid === 'page-shift-req-form')  renderReqHistory();
  if (pid === 'page-unavail-request') renderUnavailRequest();
}
function setLang(lang) {
  currentLang = lang;
  try { localStorage.setItem('ymLang', lang); } catch(e) {}
  updateLangBtn();
  rerenderStaff();
}
// Google Translate API
const GOOGLE_TRANSLATE_API_KEY = ''; // Google CloudコンソールでTranslation APIに限定したキーを設定
async function translateToJa(text) {
  if (!GOOGLE_TRANSLATE_API_KEY || !text.trim()) return null;
  try {
    const res = await fetch(
      `https://translation.googleapis.com/language/translate/v2?key=${GOOGLE_TRANSLATE_API_KEY}`,
      { method:'POST', headers:{'Content-Type':'application/json'},
        body: JSON.stringify({ q: text, target: 'ja', source: 'en' }) }
    );
    const data = await res.json();
    return data.translations?.[0]?.translatedText ?? null;
  } catch(e) { return null; }
}
```

- [ ] **Step 2: `.lang-toggle-btn` のCSSを `<style>` ブロックに追加する**

`.logout-btn:hover { ... }` の直後（line ~57）に以下を追加：

```css
  .lang-toggle-btn { background:rgba(255,255,255,.15); border:none; color:white; padding:6px 10px; border-radius:20px; cursor:pointer; font-size:12px; letter-spacing:1px; }
  .lang-toggle-btn:hover { background:rgba(255,255,255,.25); }
```

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: i18n辞書・コア関数・shiftTypeLabel・translateToJa を追加"
```

---

## Task 2: ヘッダー言語トグルボタン追加

**Files:**
- Modify: `index.html` — ヘッダーHTML (line ~512)

- [ ] **Step 1: ヘッダーにJA/ENボタンを追加する**

```html
<button class="logout-btn" onclick="doLogout()">ログアウト</button>
```
↓ この行を以下に置き換える：

```html
      <button class="lang-toggle-btn" onclick="setLang(currentLang==='ja'?'en':'ja')"><span id="lang-label-ja" style="font-weight:700">JA</span> / <span id="lang-label-en" style="opacity:.5">EN</span></button>
      <button class="logout-btn" data-i18n="logout_btn" onclick="doLogout()">ログアウト</button>
```

- [ ] **Step 2: `_enterApp()` の末尾で `updateLangBtn()` を呼ぶ**

`_enterApp` 関数内の `showPage(...)` 呼び出しの直前に以下を追加：

```javascript
  updateLangBtn();
```

現在 `_enterApp` には `updateRoleSwitchBtn()` の呼び出しがある（line ~2557）。その直後に追記する：

```javascript
  updateRoleSwitchBtn();
  updateLangBtn(); // ← 追加
```

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: ヘッダーにJA/EN言語切替ボタンを追加"
```

---

## Task 3: 静的HTML要素に `data-i18n` 属性を付与

**Files:**
- Modify: `index.html` — ログイン画面・ナビゲーション

- [ ] **Step 1: ログイン画面のラベルに `data-i18n` を追加する**

以下の各行を変更する：

```html
<!-- 変更前 -->
<div class="login-role-label">ログイン区分を選択してください</div>
<!-- 変更後 -->
<div class="login-role-label" data-i18n="select_role">ログイン区分を選択してください</div>
```

```html
<!-- 変更前 -->
<div class="role-btn active" onclick="selectRole('admin')" id="role-admin"><span class="role-icon">🔑</span>管理者</div>
<div class="role-btn" onclick="selectRole('staff')" id="role-staff"><span class="role-icon">👤</span>スタッフ</div>
<!-- 変更後 -->
<div class="role-btn active" onclick="selectRole('admin')" id="role-admin"><span class="role-icon">🔑</span><span data-i18n="role_admin">管理者</span></div>
<div class="role-btn" onclick="selectRole('staff')" id="role-staff"><span class="role-icon">👤</span><span data-i18n="role_staff">スタッフ</span></div>
```

```html
<!-- 変更前 -->
<div class="form-group"><label>ID</label><input ...></div>
<div class="form-group"><label>パスワード</label><input ...></div>
<!-- 変更後 -->
<div class="form-group"><label data-i18n="login_id">ID</label><input ...></div>
<div class="form-group"><label data-i18n="login_password">パスワード</label><input ...></div>
```

```html
<!-- 変更前 -->
<div id="login-error" ...>IDまたはパスワードが正しくありません</div>
<!-- 変更後 -->
<div id="login-error" data-i18n="login_error" ...>IDまたはパスワードが正しくありません</div>
```

- [ ] **Step 2: スタッフサイドバーナビに `data-i18n` を追加する**

`id="sidebar-staff"` ブロック内（line ~548）：

```html
<!-- 変更前 -->
<div class="nav-item" onclick="showPage('my-shift')" id="nav-my-shift"><span class="nav-icon">📅</span>マイシフト</div>
<div class="nav-item" onclick="showPage('all-shift')" id="nav-all-shift"><span class="nav-icon">👥</span>全体シフト表</div>
<div class="nav-item" onclick="showPage('shift-req-form')" id="nav-shift-req-form"><span class="nav-icon">✏️</span>休み希望申請</div>
<div class="nav-item" onclick="showPage('unavail-request')" id="nav-unavail-request"><span class="nav-icon">🙋</span>勤務希望申請</div>
<!-- 変更後 -->
<div class="nav-item" onclick="showPage('my-shift')" id="nav-my-shift"><span class="nav-icon">📅</span><span data-i18n="nav_my_shift">マイシフト</span></div>
<div class="nav-item" onclick="showPage('all-shift')" id="nav-all-shift"><span class="nav-icon">👥</span><span data-i18n="nav_all_shift">全体シフト表</span></div>
<div class="nav-item" onclick="showPage('shift-req-form')" id="nav-shift-req-form"><span class="nav-icon">✏️</span><span data-i18n="nav_shift_req">休み希望申請</span></div>
<div class="nav-item" onclick="showPage('unavail-request')" id="nav-unavail-request"><span class="nav-icon">🙋</span><span data-i18n="nav_unavail_req">勤務希望申請</span></div>
```

- [ ] **Step 3: スタッフ用ボトムナビに `data-i18n` を追加する**

`id="bottom-nav-staff"` ブロック内（line ~1077）：

```html
<!-- 変更前 -->
<button class="bottom-nav-item active" onclick="showPage('my-shift');setBottomActive(this)"><span class="bottom-nav-icon">📅</span>マイシフト</button>
<button class="bottom-nav-item" onclick="showPage('all-shift');setBottomActive(this)"><span class="bottom-nav-icon">👥</span>全体表</button>
<button class="bottom-nav-item" onclick="showPage('shift-req-form');setBottomActive(this)"><span class="bottom-nav-icon">✏️</span>休み希望</button>
<button class="bottom-nav-item" onclick="showPage('unavail-request');setBottomActive(this)"><span class="bottom-nav-icon">🙋</span>希望申請</button>
<!-- 変更後 -->
<button class="bottom-nav-item active" onclick="showPage('my-shift');setBottomActive(this)"><span class="bottom-nav-icon">📅</span><span data-i18n="nav_my_shift">マイシフト</span></button>
<button class="bottom-nav-item" onclick="showPage('all-shift');setBottomActive(this)"><span class="bottom-nav-icon">👥</span><span data-i18n="nav_all_shift">全体表</span></button>
<button class="bottom-nav-item" onclick="showPage('shift-req-form');setBottomActive(this)"><span class="bottom-nav-icon">✏️</span><span data-i18n="nav_shift_req">休み希望</span></button>
<button class="bottom-nav-item" onclick="showPage('unavail-request');setBottomActive(this)"><span class="bottom-nav-icon">🙋</span><span data-i18n="nav_unavail_req">希望申請</span></button>
```

- [ ] **Step 4: ヘッダーのスタッフバッジに `data-i18n` を追加する**

`_enterApp()` 内のバッジ設定を変更する（line ~2551）：

```javascript
// 変更前
document.getElementById('role-badge').textContent = currentRole==='admin' ? '管理者' : 'スタッフ';
// 変更後
document.getElementById('role-badge').textContent = currentRole==='admin' ? '管理者' : t('badge_staff');
```

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat: ログイン・ナビ・ヘッダーにdata-i18n属性を付与"
```

---

## Task 4: マイシフト画面の翻訳対応

**Files:**
- Modify: `index.html` — `renderMyShift()` 関数（line ~6522）

- [ ] **Step 1: マイシフトのページタイトルに `data-i18n` を付与する**

`id="page-my-shift"` のページタイトル要素を探し、以下のように変更する：

`<div class="page-title">📅 マイシフト</div>` を：
```html
<div class="page-title">📅 <span data-i18n="my_shift_title">マイシフト</span></div>
```

- [ ] **Step 2: renderMyShift() の subtitle を翻訳対応にする**

```javascript
// 変更前（line ~6529）
document.getElementById('my-shift-subtitle').textContent = `${staffName}さんのシフト (${year}年${month}月)`;
// 変更後
document.getElementById('my-shift-subtitle').textContent = currentLang === 'en'
  ? `${staffName}'s Shift (${year}/${month})`
  : `${staffName}さんのシフト (${year}年${month}月)`;
```

- [ ] **Step 3: カレンダーの曜日ヘッダーを翻訳対応にする**

`renderMyShift()` 内でカレンダーの曜日ヘッダーを生成している箇所を探す。`WEEKDAYS` 配列を直接参照している箇所を `weekdayLabel()` に変更する。

検索パターン: `WEEKDAYS[` が使われている `renderMyShift()` 内の全箇所

例：
```javascript
// 変更前
`${WEEKDAYS[w]}`
// 変更後
`${weekdayLabel(w)}`
```

- [ ] **Step 4: シフト種別の表示を翻訳対応にする**

`renderMyShift()` 内でシフトセルのテキストを表示している箇所（`patName(...)` や `p.name` を使っている箇所）を `shiftTypeLabel()` で包む。

検索パターン: `p.name` または `patName(` が `renderMyShift()` 内で使われている箇所

```javascript
// 変更前（概念的な例）
cellText = p.name;
// 変更後
cellText = shiftTypeLabel(p.name);
```

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat: マイシフト画面の曜日・シフト種別を翻訳対応"
```

---

## Task 5: 勤務希望申請画面の翻訳対応（フォーム・UNAVAIL_REASONS キー化）

**Files:**
- Modify: `index.html` — `UNAVAIL_REASONS` 定数・`renderUnavailRequest()` (line ~1853, 2847)

- [ ] **Step 1: UNAVAIL_REASONS をキー配列に変更する**

```javascript
// 変更前（line 1853）
const UNAVAIL_REASONS = [
  '📅 私的な予定がある',
  '🏥 通院・医療',
  '💒 冠婚葬祭',
  '🏫 子の学校行事',
  '👨‍👩‍👦 家族の介護・育児',
  '📋 その他',
];
// 変更後
const UNAVAIL_REASONS = [
  'reason_private',
  'reason_medical',
  'reason_ceremony',
  'reason_school',
  'reason_family',
  'reason_other',
];
```

- [ ] **Step 2: renderUnavailRequest() の理由セレクト生成をキー対応にする**

`sel.options.length <= 1` のブロック内（line ~2915）：

```javascript
// 変更前
UNAVAIL_REASONS.forEach(r => {
  const opt = document.createElement('option');
  opt.value = r; opt.textContent = r;
  sel.appendChild(opt);
});
// 変更後
UNAVAIL_REASONS.forEach(key => {
  const opt = document.createElement('option');
  opt.value = key; opt.textContent = t(key);
  sel.appendChild(opt);
});
```

言語切替時に選択肢テキストを更新するため、毎回オプションを再生成する。ただしイベントリスナーの重複を避けるため `onchange` プロパティを使う：

```javascript
// 変更前
if (sel && sel.options.length <= 1) {
  UNAVAIL_REASONS.forEach(r => {
    const opt = document.createElement('option');
    opt.value = r; opt.textContent = r;
    sel.appendChild(opt);
  });
  if (!document.getElementById('unavail-reason-other-wrap')) {
    ...
    sel.addEventListener('change', onUnavailReasonChange);
  }
}
// 変更後
if (sel) {
  while (sel.options.length > 1) sel.remove(1); // 既存オプションをクリア（重複防止）
  UNAVAIL_REASONS.forEach(key => {
    const opt = document.createElement('option');
    opt.value = key; opt.textContent = t(key);
    sel.appendChild(opt);
  });
  sel.onchange = onUnavailReasonChange; // addEventListenerでなくonchangeで重複を防ぐ
  if (!document.getElementById('unavail-reason-other-wrap')) {
    const wrap = document.createElement('div');
    wrap.id = 'unavail-reason-other-wrap';
    wrap.style.cssText = 'display:none;margin-top:6px;';
    wrap.innerHTML = `<input type="text" class="form-input" id="unavail-reason-other-input" placeholder="${t('reason_other_placeholder') ?? 'Please describe the reason'}" maxlength="100">`;
    sel.parentNode.appendChild(wrap);
  }
}
```

`I18N` 辞書に `reason_other_placeholder` を追加する（Task 1 のコードに追記）：
```javascript
// ja
reason_other_placeholder: '理由を具体的にご記入ください',
// en
reason_other_placeholder: 'Please describe the reason',
```

- [ ] **Step 3: renderUnavailRequest() のサマリーカードを翻訳対応にする**

`sumEl.innerHTML = \`` のブロック（line ~2861）内の日本語文字列を `t()` に変更：

```javascript
// 変更前
<div style="font-size:11px;color:var(--text-muted);">残り申請回数</div>
// 変更後
<div style="font-size:11px;color:var(--text-muted);">${t('unavail_remaining')}</div>
```

同様に「申請済み（承認待含む）」→ `${t('unavail_used')}`、「月の上限」→ `${t('unavail_limit')}`、「締切：」→ `${t('unavail_deadline')}：`、「（締切済み）」→ `${t('unavail_deadline_passed')}` に変更。

- [ ] **Step 4: 「その他」チェックを新キーに合わせて変更する**

```javascript
// 変更前（line ~3052）
const reason = reasonSel === '📋 その他' ? `📋 その他：${otherTxt}` : reasonSel;
// 変更後
const reason = reasonSel === 'reason_other' ? `reason_other:${otherTxt}` : reasonSel;
```

`onUnavailReasonChange` 関数内の「その他」チェックも同様：

```javascript
// 変更前
if (sel.value === '📋 その他') {
// 変更後
if (sel.value === 'reason_other') {
```

- [ ] **Step 5: renderUnavailManage()（管理者側）の理由表示を t_ja() 対応にする**

管理者の申請一覧で reason を表示している箇所（line ~3116 付近）：

```javascript
// 変更前
<div style="font-size:12px;color:var(--text-muted);margin-top:2px;">${r.reason || ''}</div>
// 変更後
<div style="font-size:12px;color:var(--text-muted);margin-top:2px;">${_displayReason(r)}</div>
```

`renderUnavailManage()` より前に `_displayReason()` 関数を追加する：

```javascript
function _displayReason(r) {
  if (!r.reason) return '';
  // キー形式 (reason_other:text) の場合
  if (r.reason.startsWith('reason_other:')) {
    const text = r.reason.slice('reason_other:'.length);
    return `${t_ja('reason_other')}：${text}`;
  }
  // I18N.jaに存在するキー
  if (I18N.ja[r.reason]) return t_ja(r.reason);
  // 旧形式（テキストそのまま）→ as-is
  return r.reason;
}
```

- [ ] **Step 6: コミット**

```bash
git add index.html
git commit -m "feat: 勤務希望申請の理由をキー管理に変更・翻訳対応"
```

---

## Task 6: 申請備考欄の自動翻訳（Google Translate API）

**Files:**
- Modify: `index.html` — `submitUnavailRequest()` (line ~3047)

- [ ] **Step 1: submitUnavailRequest() を async に変更し、備考翻訳を組み込む**

関数全体を以下に置き換える（既存ロジックを維持しつつ async 化・翻訳追加）：

```javascript
async function submitUnavailRequest() {
  const dateVal   = document.getElementById('unavail-date-input').value;
  const patId     = document.getElementById('unavail-pattern-input').value;
  const reasonSel = document.getElementById('unavail-reason-input').value;
  const otherTxt  = document.getElementById('unavail-reason-other-input')?.value.trim() || '';
  const noteRaw   = document.getElementById('unavail-note-input')?.value.trim() || '';
  const reason    = reasonSel === 'reason_other' ? `reason_other:${otherTxt}` : reasonSel;
  if (!dateVal)   { showToast(t('toast_select_date')); return; }
  if (!patId)     { showToast(t('toast_select_pattern')); return; }
  if (!reasonSel) { showToast(t('toast_select_reason')); return; }
  if (reasonSel === 'reason_other' && !otherTxt) { showToast(t('toast_other_required')); return; }
  const [y, m, d] = dateVal.split('-').map(Number);
  const today     = new Date();
  const todayNorm = new Date(today.getFullYear(), today.getMonth(), today.getDate());
  if (new Date(y, m-1, d) <= todayNorm) { showToast(t('toast_past_date')); return; }
  if (_unavailDeadlinePassed(y, m)) {
    const dm2 = m===1?12:m-1, dy2 = m===1?y-1:y;
    showToast(`${y}年${m}月分の申請は${dy2}年${dm2}月${UNAVAIL_DEADLINE_DAY}日で締め切られました`);
    return;
  }
  const used = _countMonthlyUnavail(currentStaffId, y, m);
  if (used >= MAX_UNAVAIL_PER_MONTH) { showToast(`${y}年${m}月の申請は上限（${MAX_UNAVAIL_PER_MONTH}回）に達しています`); return; }
  const dup = Object.values(unavailRequests).find(r =>
    r.staffId === currentStaffId && r.year===y && r.month===m && r.day===d && r.status !== 'rejected'
  );
  if (dup) { showToast(t('toast_already_applied')); return; }

  // 備考の翻訳処理
  let note = noteRaw;
  let noteOriginal = null;
  let noteTranslateError = false;
  if (currentLang === 'en' && noteRaw) {
    showToast(t('toast_translating'));
    const translated = await translateToJa(noteRaw);
    if (translated) {
      note = translated;
      noteOriginal = noteRaw;
    } else {
      note = noteRaw;
      noteTranslateError = true;
    }
  }

  const reqId = 'ur_' + Date.now() + '_' + Math.random().toString(36).substr(2,6);
  const req = {
    staffId: currentStaffId, staffName: currentStaffName,
    year: y, month: m, day: d,
    patternId: patId, patternName: patName(patId),
    reason, status: 'pending',
    submittedAt: Date.now(), decidedAt: null, decidedBy: null,
    ...(note ? { note } : {}),
    ...(noteOriginal ? { noteOriginal } : {}),
    ...(noteTranslateError ? { noteTranslateError: true } : {}),
  };
  unavailRequests[reqId] = req;
  fbSave(`/unavailRequests/${reqId}`, req);
  document.getElementById('unavail-date-input').value = '';
  document.getElementById('unavail-pattern-input').value = '';
  document.getElementById('unavail-reason-input').value = '';
  if (document.getElementById('unavail-reason-other-input')) document.getElementById('unavail-reason-other-input').value = '';
  if (document.getElementById('unavail-note-input')) document.getElementById('unavail-note-input').value = '';
  const otherWrap = document.getElementById('unavail-reason-other-wrap');
  if (otherWrap) otherWrap.style.display = 'none';
  onUnavailPatternChange();
  showToast(t('toast_submitted'));
  renderUnavailRequest();
  _updateUnavailDot();
}
```

- [ ] **Step 2: 備考欄HTMLが存在しない場合、勤務希望申請フォームに追加する**

`id="page-unavail-request"` 内のフォームで、申請ボタンの直前に備考欄を追加する（`id="unavail-form-area"` ブロック内）：

```html
<!-- 申請ボタンの直前に追加 -->
<div class="form-group">
  <label data-i18n="unavail_note_label">備考（任意・日本語で記入）</label>
  <input type="text" class="form-input" id="unavail-note-input" maxlength="200">
</div>
```

- [ ] **Step 3: 管理者の申請詳細で noteTranslateError と noteOriginal を表示する**

`renderUnavailManage()` の申請詳細レンダリング箇所（`r.reason || ''` の表示の直後）に以下を追加：

```javascript
${r.note ? `<div style="font-size:12px;color:var(--text-muted);margin-top:2px;">📝 ${r.note}${r.noteTranslateError ? ' <span style="color:var(--warning);">⚠️ 未翻訳（原文）</span>' : ''}</div>` : ''}
${r.noteOriginal ? `<div style="font-size:11px;color:var(--text-muted);">原文: ${r.noteOriginal}</div>` : ''}
```

- [ ] **Step 4: コミット**

```bash
git add index.html
git commit -m "feat: 勤務希望申請の備考欄を追加・英語モード時にGoogle翻訳APIで自動翻訳"
```

---

## Task 7: 休み希望申請・全体シフト表・その他ページのタイトル翻訳

**Files:**
- Modify: `index.html` — 各ページタイトル

- [ ] **Step 1: 休み希望申請ページのタイトルに `data-i18n` を付与する**

`id="page-shift-req-form"` 内のページタイトル：
```html
<!-- 変更前 -->
<div class="page-title">✏️ 休み希望申請</div>
<!-- 変更後 -->
<div class="page-title">✏️ <span data-i18n="shift_req_title">休み希望申請</span></div>
```

- [ ] **Step 2: 全体シフト表ページのタイトルに `data-i18n` を付与する**

`id="page-all-shift"` 内のページタイトル：
```html
<!-- 変更前 -->
<div class="page-title">👥 全体シフト表</div>
<!-- 変更後 -->
<div class="page-title">👥 <span data-i18n="all_shift_title">全体シフト表</span></div>
```

- [ ] **Step 3: 全体シフト表の曜日ヘッダーを翻訳対応にする**

`renderAllShiftTable()` 関数内で WEEKDAYS を参照している箇所を `weekdayLabel()` に変更する：
```javascript
// 変更前
WEEKDAYS[w]
// 変更後
weekdayLabel(w)
```

- [ ] **Step 4: コミット**

```bash
git add index.html
git commit -m "feat: 休み希望申請・全体シフト表のタイトル・曜日を翻訳対応"
```

---

## Task 8: デプロイ

**Files:**
- `index.html`

- [ ] **Step 1: GOOGLE_TRANSLATE_API_KEY を設定する**

Google CloudコンソールでTranslation APIに限定したAPIキーを発行し、Task 1 で追加した以下の行に設定する：

```javascript
const GOOGLE_TRANSLATE_API_KEY = ''; // ← ここにキーを入力
```

APIキーをまだ発行していない場合は空のままでOK（翻訳なしで動作する）。

- [ ] **Step 2: git push してデプロイ**

```bash
git push origin main
```

GitHub Pages が自動的にビルドしてデプロイされる（約30秒〜1分）。

- [ ] **Step 3: 動作確認（クマリのスマホ または PC）**

1. ページをリロードしてヘッダーにJA/ENボタンが表示されることを確認
2. ENをタップして全ナビ・ログイン画面が英語に切り替わることを確認
3. スタッフでログインし、マイシフトのシフト種別が英語で表示されることを確認
4. 勤務希望申請画面で理由の選択肢が英語で表示されることを確認
5. 備考欄に英語を入力して申請し、管理者画面で日本語訳が表示されることを確認（APIキー設定時）
6. JAに切り替えると日本語に戻ることを確認
7. ページリロード後も言語設定が保持されることを確認
