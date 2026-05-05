# 言語切替（JP/EN）機能 設計書

## 概要

外国人スタッフ（3名）向けに、シフト管理システムのスタッフ画面を英語で表示できる機能を追加する。ヘッダー右上のJA/ENボタンで切替し、設定はデバイスごとにlocalStorageに保存する。

---

## 決定事項

| 項目 | 決定内容 |
|---|---|
| 対象ロール | スタッフのみ（管理者画面は翻訳しない） |
| 言語保存 | デバイスごと（localStorage） |
| トグル配置 | ヘッダー右上 |
| トグルデザイン | `JA / EN` テキスト切替（現在の言語を太字） |
| シフト種別 | 翻訳対象（英語名を辞書に定義） |
| 申請理由 | キーで保存し、表示時に言語に応じて変換 |
| 備考欄（自由入力） | 英語モード時に送信するとGoogle Translate APIで自動翻訳してから保存。原文も `noteOriginal` フィールドに保存 |
| 管理者画面 | 翻訳対象外。申請理由キーは常に日本語テキストに変換して表示 |

---

## アーキテクチャ

### i18n コアシステム

```javascript
// 翻訳辞書（I18N）
const I18N = {
  ja: { my_shift: 'マイシフト', ... },
  en: { my_shift: 'My Shift',   ... }
};

// 現在の言語
let currentLang = localStorage.getItem('ymLang') || 'ja';

// 翻訳関数（キーが存在しない場合はja辞書にフォールバック、それもなければキーそのまま）
function t(key) {
  return I18N[currentLang]?.[key] ?? I18N.ja[key] ?? key;
}

// 言語切替
function setLang(lang) {
  currentLang = lang;
  localStorage.setItem('ymLang', lang);
  applyI18n();      // data-i18n 属性を持つ要素を一括更新
  rerenderStaff();  // スタッフ画面を再描画（マイシフト・申請画面）
  updateLangBtn();  // ヘッダーボタンの強調表示を更新
}
```

### ヘッダーボタン

```html
<!-- ログアウトボタンの左隣に追加 -->
<button class="lang-toggle-btn" onclick="setLang(currentLang==='ja'?'en':'ja')">
  <span id="lang-label-ja">JA</span> / <span id="lang-label-en">EN</span>
</button>
```

現在の言語側を `font-weight: bold`、非選択側を `opacity: 0.5` で表示。

### 静的要素の翻訳（data-i18n）

HTMLの静的テキストノードには `data-i18n="key"` 属性を付与する。

```html
<div class="login-role-label" data-i18n="select_role">ログイン区分を選択してください</div>
```

`applyI18n()` はこれらを走査してテキストを書き換える。

```javascript
function applyI18n() {
  document.querySelectorAll('[data-i18n]').forEach(el => {
    el.textContent = t(el.dataset.i18n);
  });
}
```

### 動的要素の翻訳（再描画）

マイシフト・申請画面などJavaScriptで描画する箇所は、既存の描画関数内で `t('key')` を使う。言語切替時は `rerenderStaff()` が当該ページを再描画する。

---

## データ保存のキー化

### 申請理由

現在は表示テキストをそのままFirebaseに保存しているが、キーに変更する。

| キー | 日本語 | 英語 |
|---|---|---|
| `illness` | 🤒 病気・体調不良 | 🤒 Illness / Poor Health |
| `family_care` | 👨‍👩‍👦 家族の介護・育児 | 👨‍👩‍👦 Family Care / Childcare |
| `ceremony` | 💒 冠婚葬祭 | 💒 Wedding / Funeral |
| `school_event` | 🏫 子の学校行事 | 🏫 Child's School Event |
| `other` | 📋 その他 | 📋 Other |

管理者画面では `t_ja(key)` でキーを常に日本語に変換して表示する（管理者の言語設定に関わらず）。

```javascript
function t_ja(key) { return I18N.ja[key] ?? key; }
```

既存データの後方互換：キーに一致しない古いデータ（テキストがそのまま入っているもの）はテキストをそのまま表示する。

### シフト種別

シフト種別はPATTERNSの `name` フィールドで管理されている。保存値は変更せず、表示時のみ翻訳する。

翻訳対象は休暇系4種（スタッフが自分のシフトで目にするもの）と半休フラグのみ。部署固有の勤務パターン（フ早・ル早など）は略称のため翻訳しない。

```javascript
const SHIFT_TYPE_I18N = {
  '公休':          { en: 'Day Off' },
  '有休':          { en: 'Paid Leave' },
  '希望休':        { en: 'Requested Off' },
  '欠勤':          { en: 'Absent' },
};
// 半休フラグの表示
const HALF_DAY_I18N = {
  'AM_kokyu': { ja: '半休AM（公休）', en: 'Half Day AM (Day Off)' },
  'PM_kokyu': { ja: '半休PM（公休）', en: 'Half Day PM (Day Off)' },
  'AM_yukyu': { ja: '半休AM（有休）', en: 'Half Day AM (Paid Leave)' },
  'PM_yukyu': { ja: '半休PM（有休）', en: 'Half Day PM (Paid Leave)' },
};
function shiftTypeLabel(name) {
  if (currentLang === 'en' && SHIFT_TYPE_I18N[name]) return SHIFT_TYPE_I18N[name].en;
  return name;
}
```

---

## 備考欄の自動翻訳

### フロー

```
英語モードで備考入力 → 送信ボタン押下
  → currentLang === 'en' かつ備考が空でない？
    → YES: Google Translate API (EN→JA) を呼ぶ
      → 成功: 翻訳テキストをFirebaseに保存 + noteOriginal に原文保存
      → 失敗: 原文をそのまま保存 + noteTranslateError: true フラグを保存
    → NO: そのまま保存
```

### API呼び出し

```javascript
async function translateToJa(text) {
  const res = await fetch(
    `https://translation.googleapis.com/language/translate/v2?key=${GOOGLE_TRANSLATE_API_KEY}`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ q: text, target: 'ja', source: 'en' })
    }
  );
  const data = await res.json();
  return data.translations?.[0]?.translatedText ?? null;
}
```

### APIキー管理

- HTMLに直接記載する（内部ツールのため許容）
- Google Cloudコンソールで **Cloud Translation APIのみ**に利用制限をかけたキーを発行する
- キーの定数名: `GOOGLE_TRANSLATE_API_KEY`

### 管理者画面での表示

- `noteTranslateError: true` のデータには「⚠️ 未翻訳（原文）」を表示
- `noteOriginal` フィールドがある場合は管理者画面の申請詳細に原文を併記する

---

## 翻訳対象ページ一覧

| ページ / 要素 | 翻訳方式 |
|---|---|
| ログイン画面のラベル・ボタン | `data-i18n` 静的置換 |
| スタッフ用ナビゲーション（下部・サイドバー） | `data-i18n` 静的置換 |
| ヘッダーのスタッフバッジ | `data-i18n` 静的置換 |
| マイシフト画面（曜日・シフト種別・ラベル類） | 再描画時に `t()` で出力 |
| 勤務不可申請画面（申請理由・ボタン・履歴） | 再描画 ＋ 申請理由キー化 |
| 勤務不可申請の備考欄 | 送信時にGoogle翻訳API（EN→JA） |
| トースト通知・エラーメッセージ | `showToast(t('...'))` に変更 |
| 管理者画面 | **対象外**（常に日本語） |

---

## 翻訳しないもの

- スタッフ名・部署名
- 日付・数字・記号
- 管理者画面全般
- シフト種別の記号（◯ △ 等）

---

## エラーハンドリング

- 翻訳APIが失敗した場合：原文をそのまま保存し `noteTranslateError: true` をセット
- `t()` でキーが見つからない場合：日本語辞書にフォールバック、それもなければキー文字列をそのまま返す
- 言語設定が読めない場合（localStorageエラー等）：日本語をデフォルトとする

---

## 実装対象外（今回のスコープ外）

- 管理者画面の英語対応
- 英語以外の言語（ネパール語等）
- ログイン画面での言語選択UI（ヘッダーのみ）
- メール通知の多言語化
