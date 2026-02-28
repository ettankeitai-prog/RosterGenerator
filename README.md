# RosterGenerator
RosterGenerator

```
<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>シフト表作成（2段ヘッダー / 日夜左右分割 / 折り返し / 希望休・祝日=日付のみ / 微調整 / 印刷A4横 / CSV入出力）</title>
  <style>
    :root { --bg:#0b0f14; --panel:#121925; --muted:#7f8ea3; --text:#e7eefb; --bad:#ff5c5c; }
    :root{
      /* 画面用：日夜の色 */
      --dayBg: rgba(255, 105, 180, .25);      /* 日勤ピンク */
      --nightBg: rgba(120, 190, 255, .30);    /* 夜勤薄青 */
      --akeBg: rgba(0,0,0,.04);
      --offBg: rgba(0,0,0,.05);               /* 公休 薄灰（左右） */
      --reqOffBg: rgba(0,0,0,.10);            /* 希望休 灰（左右） */
      --paidBg: rgba(0,0,0,.16);              /* 有給 濃灰（左右） */
      --dMark: rgba(255, 215, 0, .18);
      --dMarkOff: rgba(255, 215, 0, .10);

      /* ★日毎の太枠（前半/後半統一） */
      --dayBorder: 2px solid rgba(0,0,0,.35);
      --gridThin: 1px solid rgba(0,0,0,.10);
      --headerBg: #f6f8fb;
      --tableBg: #ffffff;
      --cellText: #0b0f14;

      /* ★日列の固定幅（前半/後半で揃える） */
      --dayColW: 62px;
      --rowH: 24px;

      /* ★社員名列（デザイン優先：広がりすぎ防止） */
      --nameColMin: 110px;
      --nameColW: 130px;
      --nameColMax: 150px;
    }

    body{ margin:0; font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Noto Sans JP", sans-serif; background:var(--bg); color:var(--text); }
    header{ padding:16px 18px; border-bottom:1px solid rgba(255,255,255,.08); display:flex; gap:12px; align-items:center; justify-content:space-between; }
    header h1{ font-size:16px; margin:0; letter-spacing:.2px; }
    .wrap{ display:grid; grid-template-columns: 460px 1fr; gap:12px; padding:12px; }
    .panel{ background:var(--panel); border:1px solid rgba(255,255,255,.08); border-radius:14px; padding:12px; }
    .panel h2{ font-size:13px; margin:0 0 8px; color:#cfe0ff; }
    .row{ display:flex; gap:8px; align-items:center; margin:8px 0; flex-wrap:wrap; }
    .row label{ font-size:12px; color:var(--muted); min-width:160px; }
    input, textarea, button{ background:#0e1522; border:1px solid rgba(255,255,255,.12); color:var(--text); border-radius:10px; padding:8px 10px; font-size:12px; }
    input[type="number"]{ width:120px; }
    input[type="month"]{ width:170px; }
    textarea{ width:100%; min-height:58px; resize:vertical; line-height:1.4; }
    button{ cursor:pointer; }
    button.primary{ background:linear-gradient(180deg, rgba(87,166,255,.22), rgba(87,166,255,.10)); border-color: rgba(87,166,255,.45); }
    button.secondary{ background:linear-gradient(180deg, rgba(255,255,255,.10), rgba(255,255,255,.04)); }
    button:disabled{ opacity:.55; cursor:not-allowed; }

    .chips{ display:flex; flex-wrap:wrap; gap:6px; margin-top:8px; }
    .chip{ font-size:11px; padding:5px 8px; border-radius:999px; border:1px solid rgba(255,255,255,.12); background:#0e1522; color:var(--muted); }
    .chip.ok{ border-color: rgba(67,209,122,.45); color:#bff7d3; }
    .chip.warn{ border-color: rgba(255,204,102,.45); color:#ffe6b3; }
    .chip.bad{ border-color: rgba(255,92,92,.45); color:#ffb8b8; }

    .gridWrap{ overflow:auto; max-height: calc(100vh - 120px); border-radius:14px; border:1px solid rgba(255,255,255,.08); background:rgba(255,255,255,.03); padding:10px; }

    .sectionTitle{
      position:sticky; top:0;
      z-index:20;
      background:linear-gradient(180deg, rgba(18,25,37,1), rgba(18,25,37,.88));
      border:1px solid rgba(255,255,255,.10);
      border-radius:12px;
      padding:8px 10px;
      margin:0 0 10px 0;
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:10px;
    }
    .sectionTitle .left{ font-size:12px; color:#d6e6ff; }
    .sectionTitle .right{ font-size:11px; color:var(--muted); }

    /* ===== 表：白基調 + 罫線を「縦に繋ぐ」ため collapse ===== */
    table{
      border-collapse:collapse;   /* ★列の太枠を上下で繋ぐ */
      width:max-content;          /* ★デザイン優先：内容幅（ページ幅に無理に合わせない） */
      min-width:unset;            /* ★100%で伸びて社員名が広がるのを防止 */
      margin:0 0 14px 0;
      background:var(--tableBg);
      table-layout:fixed;         /* ★列幅のブレを抑える */
    }
    th, td{
      border:var(--gridThin);
      white-space:nowrap;
      box-sizing:border-box;
    }

    thead th{
      position:sticky;
      top:44px; /* sectionTitleの下 */
      background:var(--headerBg);
      z-index:12;
      text-align:center;
      padding:6px 6px;
      font-size:12px;
      color:var(--cellText);
    }

    /* ★社員名列（角・行ヘッダ）幅固定＋省略表示 */
    th.corner,
    tbody th{
      width:var(--nameColW);
      min-width:var(--nameColMin);
      max-width:var(--nameColMax);
      overflow:hidden;
      text-overflow:ellipsis;
    }

    tbody th{
      position:sticky; left:0;
      background:var(--headerBg);
      z-index:11;
      text-align:left;
      padding:6px 8px;
      font-size:12px;
      color:var(--cellText);
    }
    th.corner{ left:0; z-index:13; }

    td{
      text-align:center;
      background:#fff;           /* ★基本背景を白 */
      padding:0;
      height:var(--rowH);
      font-size:12px;
      color:var(--cellText);
    }

    /* ★日列の固定幅（thead/th/td 全て揃える） */
    th.dayCol, td.dayCol{
      width:var(--dayColW);
      min-width:var(--dayColW);
      max-width:var(--dayColW);
    }

    /* weekend/holiday はほんのり */
    td.weekend{ box-shadow: inset 0 0 0 999px rgba(0,0,0,.02); }
    td.holiday{ box-shadow: inset 0 0 0 999px rgba(255,204,102,.18); }

    /* bad */
    td.bad{ outline:2px solid rgba(255,92,92,.75); outline-offset:-2px; }

    /* fixed (前月末夜勤→1日目 明け固定) */
    td.locked{ outline:2px dashed rgba(0,0,0,.25); outline-offset:-2px; }

    /* ===== 日ごとの縦枠（一本線に見せる） ===== */
    .dayCol{
      border-left: var(--dayBorder) !important;
      border-right: var(--dayBorder) !important;
    }
    .dayColTop{
      border-top: var(--dayBorder) !important;
    }
    .dayColBottom{
      border-bottom: var(--dayBorder) !important;
    }

    /* ===== 日夜左右分割（間の線は不要） ===== */
    .cellWrap{ position:relative; width:100%; height:var(--rowH); }
    .cell2{
      display:grid;
      grid-template-columns: 1fr 1fr;
      height:var(--rowH);
      align-items:stretch;
    }
    .cell2 .half{
      display:flex;
      align-items:center;
      justify-content:center;
      font-size:12px;
      letter-spacing:.5px;
      background:#fff; /* ★反対側は白 */
    }
    .cell2 .half.left{ border-right:none; } /* ★日夜の間の薄線なし */

    /* 日勤＝左だけ色 */
    td.shift-day .cell2 .half.left{ background: var(--dayBg); }
    /* 夜勤＝右だけ色 */
    td.shift-night .cell2 .half.right{ background: var(--nightBg); }

    /* 明け */
    td.shift-ake .cell2 .half.left,
    td.shift-ake .cell2 .half.right{ background: var(--akeBg); }

    /* 休み（左右薄灰） */
    td.shift-off .cell2 .half.left,
    td.shift-off .cell2 .half.right{ background: var(--offBg); }

    /* 希望休（左右濃灰） */
    td.reqoff .cell2 .half.left,
    td.reqoff .cell2 .half.right{ background: var(--reqOffBg); }
    /* 有給（左右さらに濃灰） */
    td.paid .cell2 .half.left,
    td.paid .cell2 .half.right{ background: var(--paidBg); }


    /* Dマーク（上から薄い金：枠線は border なので残る） */
    td.d-earned{ box-shadow: inset 0 0 0 999px var(--dMark); }
    td.d-off{ box-shadow: inset 0 0 0 999px var(--dMarkOff); }

    .centerBadge{
      position:absolute;
      inset:0;
      display:flex;
      align-items:center;
      justify-content:center;
      font-size:11px;
      color:rgba(0,0,0,.70);
      pointer-events:none;
    }

    .legend{ display:flex; flex-wrap:wrap; gap:6px; margin-top:6px; }
    .legend span{ font-size:11px; color:var(--muted); border:1px solid rgba(255,255,255,.10); border-radius:999px; padding:4px 8px; background:#0e1522; }
    .small{ font-size:11px; color:var(--muted); }

    .caseList{ display:grid; gap:8px; }
    .caseCard{ border:1px solid rgba(255,255,255,.10); background:#0e1522; border-radius:12px; padding:10px; }
    .caseCard h3{ margin:0 0 6px; font-size:12px; color:#d6e6ff; display:flex; justify-content:space-between; gap:10px; }
    .caseCard .meta{ font-size:11px; color:var(--muted); display:flex; flex-wrap:wrap; gap:10px; }
    .caseCard button{ margin-top:8px; width:100%; }
    .hr{ height:1px; background:rgba(255,255,255,.08); margin:10px 0; }

    .empBlock{ border:1px solid rgba(255,255,255,.10); background:#0e1522; border-radius:12px; padding:10px; margin:10px 0; }
    .empTop{ display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
    .empTop input[type="text"]{ width:140px; }
    .empGrid{ display:grid; grid-template-columns: 1fr; gap:8px; margin-top:8px; }
    .empGrid .hint{ font-size:11px; color:var(--muted); }
    .empGrid textarea{ min-height:32px; height:32px; }

    /* 後半のカウント列の縦ズレ対策 */
    td.metaCell{
      height:var(--rowH);
      padding:0 8px;
      background:#fff;
      color:var(--cellText);
      vertical-align:middle;
      font-size:12px;
    }
    td.metaCell > .metaIn{
      height:var(--rowH);
      display:flex;
      align-items:center;
      justify-content:center;
      white-space:nowrap;
    }

    /* ★カウント列の左に太い区切り線 */
    .countDivider{
      border-left: var(--dayBorder) !important;
    }

    /* 編集ポップ */
    .editPop{
      position:fixed;
      inset:auto 16px 16px auto;
      width:320px;
      max-width: calc(100vw - 32px);
      background:#0e1522;
      border:1px solid rgba(255,255,255,.14);
      border-radius:14px;
      padding:12px;
      box-shadow: 0 10px 30px rgba(0,0,0,.45);
      z-index:9999;
      display:none;
    }
    .editPop .title{
      font-size:12px; color:#d6e6ff; margin:0 0 8px 0;
      display:flex; justify-content:space-between; gap:8px; align-items:center;
    }
    .editPop .title .x{ cursor:pointer; color:var(--muted); }
    .editPop .btns{ display:grid; grid-template-columns: repeat(5, 1fr); gap:6px; }
    .editPop button{ padding:8px 0; border-radius:10px; font-size:12px; }
    .editPop .note{ margin-top:8px; font-size:11px; color:var(--muted); line-height:1.4; }

    /* =========================
       印刷（PDF）：A4 横向き
    ========================= */
    @media print {
      @page{
        size: A4 landscape;
        margin: 6mm;
      }
      *{
        -webkit-print-color-adjust: exact;
        print-color-adjust: exact;
      }

      body{ background:#fff !important; }
      .wrap{ display:block; }
      .wrap > .panel:first-child{ display:none !important; } /* 左パネル非表示 */
      header{ border:none; padding:0 0 4mm 0; }
      header h1{ font-size:12px; color:#000; }
      header .small{ color:#000; }

      .gridWrap{
        max-height:none;
        overflow:visible;
        border:none;
        padding:0;
        background:#fff;
        transform: scale(0.82);
        transform-origin: top left;
        width: calc(100% / 0.82);
      }

      .sectionTitle{
        position:static !important;
        background:#fff !important;
        border:none !important;
        color:#000 !important;
        padding:0 0 2mm 0 !important;
        margin:0 0 2mm 0 !important;
      }
      .sectionTitle .left{ font-size:11px; color:#000 !important; }
      .sectionTitle .right{ font-size:10px; color:#000 !important; }

      thead th, tbody th{
        position:static !important; /* sticky解除 */
        background:#f6f8fb !important;
        color:#000 !important;
      }

      :root{
        --rowH: 24px;
        --dayColW: 40px;
        --nameColMin: 90px;
        --nameColW: 110px;
        --nameColMax: 130px;
      }
      thead th{ padding:2px 2px !important; font-size:9px !important; }
      tbody th{ padding:2px 4px !important; font-size:9px !important; }
      td{ font-size:9px !important; }
      .centerBadge{ font-size:9px !important; }
      td.metaCell{ font-size:9px !important; padding:0 4px !important; }

      table{ page-break-inside: avoid; background:#fff !important; margin-bottom:3mm !important; }

      .dayCol{ border-left:2px solid #000 !important; border-right:2px solid #000 !important; }
      .dayColTop{ border-top:2px solid #000 !important; }
      .dayColBottom{ border-bottom:2px solid #000 !important; }
      .countDivider{ border-left:2px solid #000 !important; }

      td.shift-day .cell2 .half.left{ background:rgba(0,0,0,.08) !important; }
      td.shift-night .cell2 .half.right{ background:rgba(0,0,0,.15) !important; }
      td.shift-off .cell2 .half.left,
      td.shift-off .cell2 .half.right{ background:rgba(0,0,0,.05) !important; }
      td.reqoff .cell2 .half.left,
      td.reqoff .cell2 .half.right{ background:rgba(0,0,0,.12) !important; }
      td.paid .cell2 .half.left,
      td.paid .cell2 .half.right{ background:rgba(0,0,0,.18) !important; }
      td.shift-ake .cell2 .half.left,
      td.shift-ake .cell2 .half.right{ background:rgba(0,0,0,.04) !important; }

      td.metaCell,
      th.metaTh{
        display:none !important;
      }
    }
  </style>
</head>

<body>
  <header>
    <h1>シフト表作成（ヘッダー2段 / 日夜左右 / 折り返し / 微調整 / 印刷A4横 / CSV入出力）</h1>
    <div class="small">祝日・希望休は「日」だけ入力（年・月は対象月で補完）</div>
  </header>

  <div class="wrap">
    <div class="panel">

      <h2>生成</h2>

      <div class="row">
        <button id="btnGenerate" class="primary">自動生成（上位4案）</button>
        <button id="btnStop">停止</button>
        <button id="btnPrint" title="A4横で印刷→PDF保存（Ctrl+P / ⌘+P）">印刷（PDF）</button>
      </div>

      <div class="row">
        <button id="btnEmpExport" class="secondary" title="社員リスト・フラグ・希望休/希望勤務をCSVで保存">社員CSV出力</button>
        <button id="btnEmpImport" class="secondary" title="社員CSVを読み込んで復元">社員CSV読込</button>
        <button id="btnResultExport" class="secondary" title="表示中のシフト結果をCSVで出力">結果CSV出力</button>
        <button id="btnResultExportStyled" class="secondary" title="表示中のシフト表をExcelで開ける見た目付き（.xls）で出力">見た目Excel出力</button>
        <input id="empCsvFile" type="file" accept=".csv,text/csv" style="display:none" />
      </div>

      <div class="small" id="progressText">待機中</div>
      
      <div class="hr"></div>
      <h2>設定</h2>

      <div class="row">
        <label>対象月</label>
        <input id="month" type="month" />
      </div>

      <div class="row">
        <label>標準労働時間(時間)</label>
        <input id="stdHr" type="number" min="0" step="0.5" value="160" />
      </div>
      <div class="row">
        <label>最低勤務時間(時間)</label>
        <input id="minHr" type="number" min="0" step="0.5" value="140" />
      </div>
      <div class="row">
        <label>最高勤務可能(時間)</label>
        <input id="maxHr" type="number" min="0" step="0.5" value="180" />
      </div>

      <div class="row">
        <label>探索時間(秒)</label>
        <input id="limitSec" type="number" min="1" max="600" value="25" />
      </div>

      <div class="row">
        <label>希望休をハード適用</label>
        <label style="min-width:auto; display:flex; align-items:center; gap:6px; color:var(--muted);">
          <input id="hardReqOffEnabled" type="checkbox" />
          ONで希望休は必ず「休」
        </label>
      </div>

      
<div class="row">
  <label>有給モード</label>
  <label style="min-width:auto; display:flex; align-items:center; gap:10px; color:var(--muted); flex-wrap:wrap;">
    <span style="display:flex; align-items:center; gap:6px;">
      <input type="radio" name="paidMode" id="paidModeNone" value="none" checked />
      なし
    </span>
    <span style="display:flex; align-items:center; gap:6px;">
      <input type="radio" name="paidMode" id="paidMode1" value="pay1" />
      有給1（見かけのみ）
    </span>
    <span style="display:flex; align-items:center; gap:6px;">
      <input type="radio" name="paidMode" id="paidMode2" value="pay2" />
      有給2（8h加算）
    </span>
  </label>
</div>

<div class="row">
        <label>編集モード（微調整）</label>
        <label style="min-width:auto; display:flex; align-items:center; gap:6px; color:var(--muted);">
          <input id="editMode" type="checkbox" />
          ONでセルクリック編集
        </label>
      </div>

      <div class="row">
        <label>祝日（“日”のみ）</label>
      </div>
      <div class="small">例: 11,23（カンマ区切り）</div>
      <div class="row">
        <textarea id="holidays" placeholder="11,23"></textarea>
      </div>

      <div class="hr"></div>

      <h2>社員 <span id="empCount">(9名)</span></h2>
      <div class="small">希望休: 3,15,28 / 希望勤務: 5=日,12=夜（どちらも“日”だけ）</div>
      <div class="row">
        <button id="btnAddEmp">＋社員追加</button>
        <button id="btnRemoveEmp">－最後を削除</button>
        <span class="small">（目安：最大12名程度）</span>
      </div>
      <div id="empList"></div>
      <div class="legend">
        <span>上段: 日にち / 下段: 曜日</span>
        <span>日=左</span>
        <span>夜=右</span>
        <span>休=左右薄灰</span>
        <span>希望休=左右濃灰</span>
        <span>カウント列は後半の最後（印刷では非表示）</span>
        <span>日ごとに太枠（縦に繋がる）</span>
      </div>

      <div class="hr"></div>

      <h2>候補（上位4件）</h2>
      <div class="caseList" id="caseList"></div>

      <div class="chips" id="statusChips"></div>
    </div>

    <div class="panel">
      <h2>シフト表</h2>
      <div class="gridWrap" id="gridWrap"></div>
    </div>
  </div>

  <!-- 編集ポップ -->
  <div class="editPop" id="editPop">
    <div class="title">
      <div id="editTitle">編集</div>
      <div class="x" id="editClose">✕</div>
    </div>
    <div class="btns">
      <button data-sh="DAY" class="primary">日</button>
      <button data-sh="NIGHT" class="primary">夜</button>
      <button data-sh="AKE" class="primary">明</button>
      <button data-sh="OFF" class="primary">休</button>
      <button data-sh="D_OFF" class="primary">休(D)</button>
    </div>
    <div class="note">
      ・「夜」を入れると翌日は自動で「明」にします（可能な場合）。<br/>
      ・夜勤の翌日は「明」が必須です（編集しても戻します）。<br/>
      ・希望休ハードON時、希望休当日は「休」固定です。
    </div>
  </div>

<script id="workerCode" type="text/plain">

(() => {
  const SHIFT = { OFF:"OFF", DAY:"DAY", NIGHT:"NIGHT", AKE:"AKE", D_OFF:"D_OFF" };
  let paidMode = "none";

  const deepCopySchedule = (s) => ({ assign: s.assign.map(r => r.slice()) });
  const scheduleKey = (s) => s.assign.map(r => r.join("|")).join("||");
  const emptySchedule = (nEmp, nDay) => ({ assign: Array.from({length:nEmp}, () => Array(nDay).fill(SHIFT.OFF)) });

  const empTotalMin = (schedule, e, workMin, paidMode, employees) => {
    let t = 0;
    const row = schedule.assign[e];
    const paidArr = employees && employees[e] ? employees[e].paidIdxSet : null;

    for (let i = 0; i < row.length; i++) {
      const sh = row[i];
      let add = (workMin[sh] ?? 0);

      // 有給モード2: 休み(OFF/D_OFF)だが 8h(=日勤) を労働時間に加算
      if (paidMode === 2 && paidArr && paidArr[i] && (sh === SHIFT.OFF || sh === SHIFT.D_OFF)) {
        add += 480;
      }
      t += add;
    }
    return t;
  };
  const empNightCount = (schedule, e) => schedule.assign[e].filter(x => x===SHIFT.NIGHT).length;

  const empOffNoAkeCount = (schedule, e) => {
    const row = schedule.assign[e];
    let c=0;
    for (let i=0;i<row.length;i++){
      const sh = row[i];
      if (sh === SHIFT.OFF || sh === SHIFT.D_OFF) c++;
    }
    return c;
  };

  function computeDCounts(schedule, days, employees) {
    const nEmp = employees.length;
    const nDay = days.length;
    const dEarn = Array(nEmp).fill(0);
    const dOff = Array(nEmp).fill(0);

    for (let e=0; e<nEmp; e++) for (let i=0; i<nDay; i++) {
      if (schedule.assign[e][i] === SHIFT.D_OFF) dOff[e]++;
    }

    for (let e=0; e<nEmp; e++) {
      if (!employees[e].proper) continue;
      for (let i=0; i<nDay; i++) {
        const sh = schedule.assign[e][i];
        const day = days[i];
        if (sh === SHIFT.DAY && day.isHolOrWeekend) dEarn[e]++;
        if (sh === SHIFT.NIGHT && day.isHoliday) dEarn[e]++;
        if (sh === SHIFT.NIGHT && i+1 < nDay) {
          const endDay = days[i+1];
          if (endDay.isHolOrWeekend) dEarn[e]++;
        }
      }
    }
    return { dEarn, dOff };
  }

  function isHardOffDay(employees, e, i, hardReqOffEnabled) {
    const isReq = !!(hardReqOffEnabled && employees[e].reqOffIdxSet && employees[e].reqOffIdxSet[i]);
    const isPaid = !!(employees[e].paidIdxSet && employees[e].paidIdxSet[i]);
    return isReq || isPaid;
  }

  function isValidLocal(schedule, days, employees, e, i, sh, cfg, hardReqOffEnabled) {
    const cur = schedule.assign[e][i];

    // 前月末夜勤 → 当月1日は明け固定（変更不可）
    if (i === 0 && employees[e].prevMonthNight) {
      return sh === SHIFT.AKE;
    }

    if (isHardOffDay(employees, e, i, hardReqOffEnabled)) {
      return (sh === SHIFT.OFF || sh === SHIFT.D_OFF);
    }

    if (cur === SHIFT.AKE && (sh === SHIFT.DAY || sh === SHIFT.NIGHT)) return false;

    if (cfg.noDayAfterAke && sh === SHIFT.DAY && i>0 && schedule.assign[e][i-1] === SHIFT.AKE) return false;

    if (sh === SHIFT.NIGHT) {
      // ★日勤5連続の翌日は「休」必須（夜勤も禁止）
      let run = 0;
      for (let k=i-1; k>=0 && schedule.assign[e][k]===SHIFT.DAY; k--) run++;
      if (run >= cfg.maxConsecDay) return false;

      if (employees[e].proper && days[i].isHolOrWeekend) return false;

      if (i+1 < days.length) {
        const nxt = schedule.assign[e][i+1];
        if (nxt === SHIFT.DAY || nxt === SHIFT.NIGHT) return false;
        if (isHardOffDay(employees, e, i+1, hardReqOffEnabled)) return false;
      }
      if (i>0 && schedule.assign[e][i-1] === SHIFT.NIGHT) return false;
    }

    if (sh === SHIFT.DAY) {
      let c=1;
      for (let k=i-1; k>=0 && schedule.assign[e][k]===SHIFT.DAY; k--) c++;
      for (let k=i+1; k<days.length && schedule.assign[e][k]===SHIFT.DAY; k++) c++;
      if (c > cfg.maxConsecDay) return false;
    }

    return true;
  }

  function enforceHardOff(schedule, employees, hardReqOffEnabled) {
    if (!hardReqOffEnabled) return;
    const nEmp = employees.length, nDay = schedule.assign[0].length;

    for (let e=0;e<nEmp;e++){
      const reqArr = employees[e].reqOffIdxSet;
      const paidArr = employees[e].paidIdxSet;
      if (!reqArr && !paidArr) continue;

      for (let i=0; i<nDay; i++){
        const hard = (!!(reqArr && reqArr[i] && hardReqOffEnabled)) || (!!(paidArr && paidArr[i]));
        if (!hard) continue;
        if (i>0 && schedule.assign[e][i-1] === SHIFT.NIGHT) schedule.assign[e][i-1] = SHIFT.OFF;
        schedule.assign[e][i] = SHIFT.OFF;
        if (i+1 < nDay && schedule.assign[e][i+1] === SHIFT.AKE) schedule.assign[e][i+1] = SHIFT.OFF;
      }
    }
  }

  function enforcePrevMonthAke(schedule, employees) {
    if (!schedule || !schedule.assign || !schedule.assign.length) return;
    const nEmp = employees.length;
    const nDay = schedule.assign[0].length;
    if (nDay <= 0) return;
    for (let e=0;e<nEmp;e++){
      if (!employees[e].prevMonthNight) continue;
      schedule.assign[e][0] = SHIFT.AKE;
    }
  }

  function normalizeAkeConsistency(schedule, days, employees, cfg, hardReqOffEnabled) {
    const nEmp = employees.length;
    const nDay = days.length;

    // 1) 先月末が夜勤なら当月1日は「明」で固定（表示/カウントはUI側で調整）
    for (let e=0; e<nEmp; e++) {
      if (employees[e].prevMonthNight) schedule.assign[e][0] = SHIFT.AKE;
    }

    // 2) 孤立した「明」（前日が夜勤でない明）はOFFに戻す（※1日固定の明は除外）
    for (let e=0; e<nEmp; e++) {
      for (let i=0; i<nDay; i++) {
        if (i === 0 && employees[e].prevMonthNight) continue;
        if (schedule.assign[e][i] === SHIFT.AKE) {
          if (i === 0 || schedule.assign[e][i-1] !== SHIFT.NIGHT) {
            schedule.assign[e][i] = SHIFT.OFF;
          }
        }
      }
    }

    // 3) 夜勤は必ず翌日「明」にする（希望休ハードで翌日OFF固定なら夜勤自体をOFFに戻す）
    for (let e=0; e<nEmp; e++) {
      for (let i=0; i<nDay-1; i++) {
        if (schedule.assign[e][i] !== SHIFT.NIGHT) continue;

        if (hardReqOffEnabled && employees[e].reqOffIdxSet && employees[e].reqOffIdxSet[i+1]) {
          schedule.assign[e][i] = SHIFT.OFF;
          continue;
        }
        schedule.assign[e][i+1] = SHIFT.AKE;
      }
    }

    // 4) 希望休ハード日の「明」は前日の夜勤を解除してOFFにする
    if (hardReqOffEnabled) {
      for (let e=0; e<nEmp; e++) {
        const arr = employees[e].reqOffIdxSet;
        if (!arr) continue;
        for (let i=0; i<nDay; i++) {
          if (!arr[i]) continue;
          if (schedule.assign[e][i] === SHIFT.AKE && i > 0 && schedule.assign[e][i-1] === SHIFT.NIGHT) {
            schedule.assign[e][i-1] = SHIFT.OFF;
            schedule.assign[e][i] = SHIFT.OFF;
          }
        }
      }
    }
  }


  function allocateDOff(schedule, days, employees, cfg, hardReqOffEnabled) {
    const nEmp = employees.length;
    const nDay = days.length;

    for (let e=0;e<nEmp;e++) for (let i=0;i<nDay;i++){
      if (schedule.assign[e][i]===SHIFT.D_OFF) schedule.assign[e][i]=SHIFT.OFF;
    }

    const dayCount = (i) => {
      let c=0;
      for (let e=0;e<nEmp;e++) if (schedule.assign[e][i]===SHIFT.DAY) c++;
      return c;
    };

    const canPutDay = (f, i) => {
      const cur = schedule.assign[f][i];
      if (cur === SHIFT.NIGHT || cur === SHIFT.AKE) return false;
      if (cur === SHIFT.DAY) return false;
      return isValidLocal(schedule, days, employees, f, i, SHIFT.DAY, cfg, hardReqOffEnabled);
    };

    const { dEarn } = computeDCounts(schedule, days, employees);

    for (let e=0; e<nEmp; e++) {
      if (!employees[e].proper) continue;

      let need = dEarn[e];
      if (need <= 0) continue;

      const offSlots = [];
      for (let i=0;i<nDay;i++){
        if (days[i].isHolOrWeekend) continue;
        if (schedule.assign[e][i] === SHIFT.OFF) {
          if (hardReqOffEnabled && isHardOffDay(employees, e, i, hardReqOffEnabled)) continue;
          offSlots.push(i);
        }
      }
      for (let k=0; k<offSlots.length && need>0; k++){
        schedule.assign[e][offSlots[k]] = SHIFT.D_OFF;
        need--;
      }
      if (need === 0) continue;

      const daySlots = [];
      for (let i=0;i<nDay;i++){
        if (days[i].isHolOrWeekend) continue;
        if (schedule.assign[e][i] === SHIFT.DAY) daySlots.push(i);
      }
      for (let i=daySlots.length-1;i>0;i--){
        const j = (Math.random()*(i+1))|0;
        [daySlots[i],daySlots[j]]=[daySlots[j],daySlots[i]];
      }

      for (let idx=0; idx<daySlots.length && need>0; idx++){
        const i = daySlots[idx];
        const dc = dayCount(i);

        if (dc - 1 >= cfg.minDay) {
          schedule.assign[e][i] = SHIFT.D_OFF;
          need--;
          continue;
        }

        let filled = false;
        for (let f=0; f<nEmp; f++){
          if (f===e) continue;
          if (!canPutDay(f, i)) continue;

          schedule.assign[f][i] = SHIFT.DAY;
          schedule.assign[e][i] = SHIFT.D_OFF;
          filled = true;
          need--;
          break;
        }
        if (!filled) schedule.assign[e][i] = SHIFT.DAY;
      }

      if (need > 0) return false;
    }

    return true;
  }

  function hardViolationsCount(schedule, days, employees, cfg, monthRules, hardReqOffEnabled) {
    const nEmp = employees.length;
    const nDay = days.length;
    let issues = 0;

    for (let i=0;i<nDay;i++){
      let dayCnt=0, nightCnt=0;
      for (let e=0;e<nEmp;e++){
        const sh = schedule.assign[e][i];
        if (sh===SHIFT.DAY) dayCnt++;
        if (sh===SHIFT.NIGHT) nightCnt++;
      }
      if (nightCnt !== cfg.requiredNight) issues++;
      if (dayCnt < cfg.minDay) issues++;
      if (dayCnt > cfg.maxDay) issues++;
    }

    for (let e=0;e<nEmp;e++){
      let consecDay=0;
      let nightSetRun=0;

      for (let i=0;i<nDay;i++){
        const sh = schedule.assign[e][i];

        if (isHardOffDay(employees, e, i, hardReqOffEnabled)) {
          if (sh !== SHIFT.OFF && sh !== SHIFT.D_OFF) issues++;
        }

        if (sh===SHIFT.DAY) consecDay++; else consecDay=0;
        // ★日勤5連続の翌日は「休」必須（夜勤も禁止）
        if (sh===SHIFT.NIGHT) {
          let run=0;
          for (let k=i-1; k>=0 && schedule.assign[e][k]===SHIFT.DAY; k--) run++;
          if (run >= cfg.maxConsecDay) issues++;
        }
        if (consecDay > cfg.maxConsecDay) issues++;

        if (employees[e].proper && sh===SHIFT.NIGHT && days[i].isHolOrWeekend) issues++;
        if (sh===SHIFT.NIGHT && i+1 < nDay && schedule.assign[e][i+1] !== SHIFT.AKE) issues++;
        if (i>0 && schedule.assign[e][i-1]===SHIFT.NIGHT && sh===SHIFT.NIGHT) issues++;
        if (cfg.noDayAfterAke && i>0 && schedule.assign[e][i-1]===SHIFT.AKE && sh===SHIFT.DAY) issues++;

        if (sh===SHIFT.NIGHT) {
          const continues = (i>=2 && schedule.assign[e][i-1]===SHIFT.AKE && schedule.assign[e][i-2]===SHIFT.NIGHT);
          nightSetRun = continues ? (nightSetRun+1) : 1;
          if (nightSetRun > cfg.maxNightSetRun) issues++;
        }
        if (sh !== SHIFT.NIGHT && sh !== SHIFT.AKE) nightSetRun = 0;
      }
    }

    for (let e=0;e<nEmp;e++){
      const total = empTotalMin(schedule,e,cfg.workMin,paidMode,employees);
      if (monthRules.minMin>0 && total < monthRules.minMin) issues++;
      if (monthRules.maxMin>0 && total > monthRules.maxMin) issues++;
    }

    const { dEarn, dOff } = computeDCounts(schedule, days, employees);
    for (let e=0;e<nEmp;e++){
      if (!employees[e].proper) continue;
      if (dEarn[e] > cfg.maxDDays) issues++;
      if (dEarn[e] !== dOff[e]) issues++;
    }

    return issues;
  }

  function scoreSchedule(schedule, days, employees, cfg, monthRules) {
    const nEmp = employees.length;
    const nDay = days.length;
    let score = 0;

    for (let i=0;i<nDay;i++){
      let dayCnt=0, nightCnt=0;
      for (let e=0;e<nEmp;e++){
        const sh = schedule.assign[e][i];
        if (sh===SHIFT.DAY) dayCnt++;
        if (sh===SHIFT.NIGHT) nightCnt++;
      }

      score -= Math.abs(nightCnt - cfg.requiredNight) * 5000;
      if (dayCnt < cfg.minDay) score -= (cfg.minDay - dayCnt) * 5000;
      if (dayCnt > cfg.maxDay) score -= (dayCnt - cfg.maxDay) * 5000;

      score -= Math.abs(dayCnt - cfg.targetDay) * 60;
    }

    for (let e=0;e<nEmp;e++){
      const total = empTotalMin(schedule,e,cfg.workMin,paidMode,employees);
      score -= Math.abs(total - monthRules.stdMin) * 0.6;
    }

    // 平日日勤優先（強化：ほぼ固定 + 夜勤は月1〜3回想定）
    for (let e=0;e<nEmp;e++){
      if (!employees[e].weekdayDayPriority) continue;

      let nightCnt = 0;
      for (let i=0;i<nDay;i++){
        const sh = schedule.assign[e][i];
        if (sh===SHIFT.NIGHT) nightCnt++;

        // 平日（祝日/土日以外）は「日」寄せ
        if (!days[i].isHolOrWeekend) {
          if (sh===SHIFT.DAY) score += 200;
          else if (sh===SHIFT.OFF || sh===SHIFT.D_OFF) score -= 220;
          else if (sh===SHIFT.NIGHT) score -= 320;
          else if (sh===SHIFT.AKE) score -= 120;
        } else {
          // 土日祝は「休」寄せ（ただし必須人数の都合で出勤は許容）
          if (sh===SHIFT.OFF || sh===SHIFT.D_OFF) score += 40;
          else if (sh===SHIFT.DAY) score -= 25;
          else if (sh===SHIFT.NIGHT) score -= 10;
          else if (sh===SHIFT.AKE) score -= 5;
        }
      }

      // 夜勤回数の目安（ソフト）
      const MIN_N = 1, MAX_N = 3;
      if (nightCnt < MIN_N) score -= (MIN_N - nightCnt) * 800;
      if (nightCnt > MAX_N) score -= (nightCnt - MAX_N) * 800;
    }

    const W_MATCH = 25;
    const W_MISS  = 30;
    for (let e=0;e<nEmp;e++){
      const wish = employees[e].wishShiftIdxMap;
      if (!wish) continue;
      for (let i=0;i<nDay;i++){
        const w = wish[i];
        if (!w) continue;
        const sh = schedule.assign[e][i];
        if (sh === w) score += W_MATCH;
        else score -= W_MISS;
      }
    }

    const { dEarn, dOff } = computeDCounts(schedule, days, employees);
    for (let e=0;e<nEmp;e++){
      if (!employees[e].proper) continue;
      score -= dEarn[e] * 5;
      score -= Math.abs(dEarn[e]-dOff[e]) * 200;
    }

    const offCounts = new Array(nEmp);
    let sumOff = 0;
    for (let e=0;e<nEmp;e++){
      const c = empOffNoAkeCount(schedule, e);
      offCounts[e] = c;
      sumOff += c;
    }
    const meanOff = sumOff / Math.max(1, nEmp);
    const OFF_STD_WEIGHT = 200;
    for (let e=0;e<nEmp;e++){
      score -= Math.abs(offCounts[e] - meanOff) * OFF_STD_WEIGHT;
    }

    return score;
  }

  function repairStaffing(s, days, employees, cfg, monthRules, hardReqOffEnabled) {
    const nEmp = employees.length, nDay = days.length;

    enforcePrevMonthAke(s, employees);
    enforceHardOff(s, employees, hardReqOffEnabled);
    normalizeAkeConsistency(s, days, employees, cfg, hardReqOffEnabled);

    for (let i=0;i<nDay;i++){
      const nightEs = [];
      for (let e=0;e<nEmp;e++) if (s.assign[e][i]===SHIFT.NIGHT) nightEs.push(e);

      if (nightEs.length > cfg.requiredNight) {
        while (nightEs.length > cfg.requiredNight) {
          const e = nightEs.pop();
          s.assign[e][i] = SHIFT.OFF;
        }
      } else if (nightEs.length < cfg.requiredNight) {
        const need = cfg.requiredNight - nightEs.length;
        const cand = [];
        for (let e=0;e<nEmp;e++){
          if (s.assign[e][i]===SHIFT.NIGHT) continue;
          if (!isValidLocal(s, days, employees, e, i, SHIFT.NIGHT, cfg, hardReqOffEnabled)) continue;
          cand.push({e, n: empNightCount(s,e), t: empTotalMin(s,e,cfg.workMin,paidMode,employees), r: Math.random()});
        }
        cand.sort((a,b)=> (a.n-b.n)||(a.t-b.t)||(a.r-b.r));
        for (let k=0;k<need && k<cand.length;k++){
          const ePick = cand[k].e;
          s.assign[ePick][i] = SHIFT.NIGHT;
          if (i+1 < nDay) s.assign[ePick][i+1] = SHIFT.AKE;
        }
      }

      if (i+1 < nDay) {
        for (let e=0;e<nEmp;e++){
          if (s.assign[e][i]===SHIFT.NIGHT) s.assign[e][i+1] = SHIFT.AKE;
        }
      }
    }

    for (let i=0;i<nDay;i++){
      let dayCnt=0;
      for (let e=0;e<nEmp;e++) if (s.assign[e][i]===SHIFT.DAY) dayCnt++;
      if (dayCnt >= cfg.minDay) continue;

      const need = cfg.minDay - dayCnt;
      const cand = [];
      for (let e=0;e<nEmp;e++){
        const cur = s.assign[e][i];
        if (cur===SHIFT.NIGHT || cur===SHIFT.AKE || cur===SHIFT.DAY) continue;
        if (!isValidLocal(s, days, employees, e, i, SHIFT.DAY, cfg, hardReqOffEnabled)) continue;
        const t = empTotalMin(s,e,cfg.workMin,paidMode,employees);
        const pref = (employees[e].weekdayDayPriority && !days[i].isHolOrWeekend) ? -1 : 0;
        cand.push({e, pref, t, r: Math.random()});
      }
      cand.sort((a,b)=> (a.pref-b.pref)||(a.t-b.t)||(a.r-b.r));
      for (let k=0;k<need && k<cand.length;k++) s.assign[cand[k].e][i] = SHIFT.DAY;
    }

    for (let i=0;i<nDay;i++){
      let dayEs = [];
      for (let e=0;e<nEmp;e++) if (s.assign[e][i]===SHIFT.DAY) dayEs.push(e);
      if (dayEs.length <= cfg.maxDay) continue;

      const cand = dayEs.map(e => ({
        e,
        pref: (employees[e].weekdayDayPriority && !days[i].isHolOrWeekend) ? 1 : 0,
        t: empTotalMin(s,e,cfg.workMin,paidMode,employees),
        r: Math.random()
      }));
      cand.sort((a,b)=> (b.pref-a.pref) || (a.t-b.t) || (a.r-b.r));

      for (let k=cfg.maxDay; k<cand.length; k++){
        s.assign[cand[k].e][i] = SHIFT.OFF;
      }
    }

    allocateDOff(s, days, employees, cfg, hardReqOffEnabled);
    enforceHardOff(s, employees, hardReqOffEnabled);
    normalizeAkeConsistency(s, days, employees, cfg, hardReqOffEnabled);
    enforcePrevMonthAke(s, employees);
  }

  function buildInitialSchedule(days, employees, cfg, monthRules, hardReqOffEnabled) {
    const s = emptySchedule(employees.length, days.length);

    enforcePrevMonthAke(s, employees);
    enforceHardOff(s, employees, hardReqOffEnabled);
    enforcePrevMonthAke(s, employees);

    for (let i=0;i<days.length;i++){
      const cand = [];
      for (let e=0;e<employees.length;e++){
        if (!isValidLocal(s, days, employees, e, i, SHIFT.NIGHT, cfg, hardReqOffEnabled)) continue;
        cand.push({e, n: empNightCount(s,e), t: empTotalMin(s,e,cfg.workMin,paidMode,employees), r:Math.random()});
      }
      cand.sort((a,b)=> (a.n-b.n)||(a.t-b.t)||(a.r-b.r));
      if (cand.length < cfg.requiredNight) return null;
      for (let k=0;k<cfg.requiredNight;k++){
        const ePick = cand[k].e;
        s.assign[ePick][i] = SHIFT.NIGHT;
        if (i+1 < days.length) s.assign[ePick][i+1] = SHIFT.AKE;
      }
    }

    for (let i=0;i<days.length;i++){
      const cand = [];
      for (let e=0;e<employees.length;e++){
        const cur = s.assign[e][i];
        if (cur===SHIFT.NIGHT || cur===SHIFT.AKE) continue;
        if (!isValidLocal(s, days, employees, e, i, SHIFT.DAY, cfg, hardReqOffEnabled)) continue;
        const t = empTotalMin(s,e,cfg.workMin,paidMode,employees);
        const pref = (employees[e].weekdayDayPriority && !days[i].isHolOrWeekend) ? -1 : 0;
        cand.push({e, pref, t, r:Math.random()});
      }
      cand.sort((a,b)=> (a.pref-b.pref)||(a.t-b.t)||(a.r-b.r));
      if (cand.length < cfg.minDay) return null;

      for (let k=0; k<cfg.minDay; k++) s.assign[cand[k].e][i] = SHIFT.DAY;

      const extra = Math.min(cfg.maxDay, cfg.targetDay) - cfg.minDay;
      for (let k=0; k<extra; k++){
        const idx = cfg.minDay + k;
        if (idx < cand.length) s.assign[cand[idx].e][i] = SHIFT.DAY;
      }
    }

    if (!allocateDOff(s, days, employees, cfg, hardReqOffEnabled)) return null;
    repairStaffing(s, days, employees, cfg, monthRules, hardReqOffEnabled);
    return s;
  }

  function mutate(schedule, days, employees, cfg, monthRules, hardReqOffEnabled) {
    const s = deepCopySchedule(schedule);
    const nEmp = employees.length;
    const nDay = days.length;

    const op = Math.random();

    if (op < 0.45) {
      let i = (Math.random()*nDay)|0;
      if (nDay > 1 && i === 0) i = 1;
      const a = (Math.random()*nEmp)|0;
      let b = (Math.random()*nEmp)|0;
      if (a===b) b = (b+1)%nEmp;

      if (!isHardOffDay(employees, a, i, hardReqOffEnabled) && !isHardOffDay(employees, b, i, hardReqOffEnabled)) {
        const sa = s.assign[a][i], sb = s.assign[b][i];
        if (sa === SHIFT.AKE || sb === SHIFT.AKE) {
          if (s.assign[a][i] !== SHIFT.NIGHT && s.assign[a][i] !== SHIFT.AKE) {
            s.assign[a][i] = (s.assign[a][i] === SHIFT.DAY ? SHIFT.OFF : SHIFT.DAY);
            if (sa === SHIFT.D_OFF) s.assign[a][i] = SHIFT.OFF;
          }
        } else {
          s.assign[a][i] = sb;
          s.assign[b][i] = sa;
        }
      }
    } else if (op < 0.75) {
      const e = (Math.random()*nEmp)|0;
      let i = (Math.random()*nDay)|0;
      if (nDay > 1 && i === 0) i = 1;
      if (!isHardOffDay(employees, e, i, hardReqOffEnabled)) {
        const cur = s.assign[e][i];
        if (cur !== SHIFT.NIGHT && cur !== SHIFT.AKE) {
          s.assign[e][i] = (cur === SHIFT.DAY ? SHIFT.OFF : SHIFT.DAY);
          if (cur === SHIFT.D_OFF) s.assign[e][i] = SHIFT.OFF;
        }
      }
    } else {
      let i = (Math.random()*nDay)|0;
      if (nDay > 1 && i === 0) i = 1;
      for (let e=0;e<nEmp;e++) if (s.assign[e][i]===SHIFT.NIGHT) s.assign[e][i]=SHIFT.OFF;
      if (i+1<nDay) for (let e=0;e<nEmp;e++) if (s.assign[e][i+1]===SHIFT.AKE) s.assign[e][i+1]=SHIFT.OFF;

      const cand = [];
      for (let e=0;e<nEmp;e++){
        if (!isValidLocal(s, days, employees, e, i, SHIFT.NIGHT, cfg, hardReqOffEnabled)) continue;
        cand.push({e, n: empNightCount(s,e), t: empTotalMin(s,e,cfg.workMin,paidMode,employees), r:Math.random()});
      }
      cand.sort((a,b)=> (a.n-b.n)||(a.t-b.t)||(a.r-b.r));
      if (cand.length >= cfg.requiredNight) {
        for (let k=0;k<cfg.requiredNight;k++){
          const ePick = cand[k].e;
          s.assign[ePick][i] = SHIFT.NIGHT;
          if (i+1<nDay) s.assign[ePick][i+1]=SHIFT.AKE;
        }
      }
    }

    repairStaffing(s, days, employees, cfg, monthRules, hardReqOffEnabled);
    return s;
  }

  function buildSummary(schedule, employees, cfg, monthRules, days) {
    const nEmp = employees.length;
    const totals = employees.map((_,e)=> empTotalMin(schedule,e,cfg.workMin,paidMode,employees));
    const avg = totals.reduce((a,b)=>a+b,0)/nEmp;

    const { dEarn, dOff } = computeDCounts(schedule, days, employees);
    const properInfo = employees
      .map((emp,e)=> emp.proper ? `\${emp.name}:D勤\${dEarn[e]}/D休\${dOff[e]}` : null)
      .filter(Boolean)
      .slice(0,6)
      .join(" / ");

    return {
      avgHr:(avg/60).toFixed(1),
      minHr:(Math.min(...totals)/60).toFixed(1),
      maxHr:(Math.max(...totals)/60).toFixed(1),
      properInfo
    };
  }

  function search(payload) {
    const start = performance.now();
    const limitMs = payload.limitMs;

    const cfg = {
      requiredNight: payload.requiredNight,
      minDay: payload.minDay,
      maxDay: payload.maxDay,
      targetDay: payload.targetDay,
      maxConsecDay: payload.maxConsecDay,
      maxNightSetRun: payload.maxNightSetRun,
      noDayAfterAke: payload.noDayAfterAke,
      maxDDays: payload.maxDDays,
      workMin: payload.workMin,
    };

    const employees = payload.employees;
    const days = payload.days;
    const monthRules = payload.monthRules;
    const hardReqOffEnabled = !!payload.hardReqOffEnabled;
    paidMode = payload.paidMode || "none";

    // --- A案：初期解（init）を内部でリトライしてから探索開始 ---
    let init = null;
    const INIT_TRIES_MAX = 120;
    for (let t=0; t<INIT_TRIES_MAX; t++){
      init = buildInitialSchedule(days, employees, cfg, monthRules, hardReqOffEnabled);
      if (init) break;
      if (performance.now() - start > limitMs * 0.20) break;
    }
    if (!init) return { iter:0, candidates:[] };

    const seen = new Set();
    const best = [];

    function consider(s) {
      const hv = hardViolationsCount(s, days, employees, cfg, monthRules, hardReqOffEnabled);
      if (hv !== 0) return;

      const sc = scoreSchedule(s, days, employees, cfg, monthRules);
      const key = scheduleKey(s);
      if (seen.has(key)) return;
      seen.add(key);

      best.push({ score: sc, schedule: s, summary: buildSummary(s, employees, cfg, monthRules, days) });
      best.sort((a,b)=> b.score - a.score);
      if (best.length > 4) best.pop();
    }

    let cur = init;
    consider(cur);

    let iter = 0;
    let lastProgress = 0;

    while (performance.now() - start < limitMs) {
      iter++;
      const nxt = mutate(cur, days, employees, cfg, monthRules, hardReqOffEnabled);

      const hvCur = hardViolationsCount(cur, days, employees, cfg, monthRules, hardReqOffEnabled);
      const hvNxt = hardViolationsCount(nxt, days, employees, cfg, monthRules, hardReqOffEnabled);

      const scCur = scoreSchedule(cur, days, employees, cfg, monthRules) - hvCur * 10000;
      const scNxt = scoreSchedule(nxt, days, employees, cfg, monthRules) - hvNxt * 10000;

      const elapsed = performance.now() - start;
      const t = Math.max(0.05, 1.0 - elapsed/limitMs);
      const accept = (scNxt >= scCur) || (Math.random() < Math.exp((scNxt - scCur) / (220 * t)));
      if (accept) cur = nxt;

      consider(nxt);
      if (best.length === 4 && iter % 250 === 0) cur = deepCopySchedule(best[0].schedule);

      if (elapsed - lastProgress > 150) {
        lastProgress = elapsed;
        postMessage({ type:"progress", iter, elapsedMs: elapsed, bestCount: best.length });
      }
    }

    return { iter, candidates: best };
  }

  onmessage = (ev) => {
    try {
      const msg = ev.data;
      if (!msg || msg.type !== "start") return;
      const result = search(msg);
      postMessage({ type:"done", iter: result.iter, candidates: result.candidates });
    } catch (e) {
      postMessage({ type:"error", error: (e && e.stack) ? e.stack : String(e) });
    }
  };
})();

</script>

<script>
(() => {
  // ===== Constants =====
  const SHIFT = { OFF:"OFF", DAY:"DAY", NIGHT:"NIGHT", AKE:"AKE", D_OFF:"D_OFF" };
  const WORK_MIN = { DAY: 480, NIGHT: 840, AKE: 0, OFF: 0, D_OFF: 0 };

  // Staffing
  const REQUIRED_NIGHT = 2;
  const MIN_DAY = 2;
  const MAX_DAY = 3;
  const TARGET_DAY = 3;

  // Hard rules
  const MAX_CONSEC_DAY = 5;
  const MAX_NIGHT_SET_RUN = 3;
  const NO_DAY_AFTER_AKE = true;

  // Proper
  const MAX_D_DAYS = 5;

  const state = {
    employees: [],
    year: 0,
    month: 0,
    days: [],
    holidaysIdx: new Set(), // 0-based day index
    monthRules: { stdMin: 160*60, minMin: 140*60, maxMin: 180*60 },
    currentSchedule: null,
    candidates: [],
    parsedEmployeesForView: null,
    editTarget: null, // {e,i}
  };

  const pad2 = n => String(n).padStart(2,"0");
  const ymd = (y,m,d) => `${y}-${pad2(m)}-${pad2(d)}`;
  const parseMonthInput = (v) => {
    const [yy, mm] = v.split("-").map(Number);
    return { yy, mm };
  };
  const clamp = (x,a,b) => Math.max(a, Math.min(b,x));

  function getPaidMode() {
    const sel = document.querySelector('input[name="paidMode"]:checked');
    const v = sel ? sel.value : "none";
    if (v === "pay1") return 1;
    if (v === "pay2") return 2;
    return 0; // none
  }

  // ===== CSV helpers（Excelでも壊れにくい）=====
  const CSV_BOM = "\ufeff";
  function csvEscape(v) {
    const s = String(v ?? "");
    if (/[",\n\r]/.test(s)) return `"${s.replace(/"/g,'""')}"`;
    return s;
  }
  function toCSV(rows) {
    return rows.map(r => r.map(csvEscape).join(",")).join("\r\n");
  }
  function downloadText(filename, text, addBom=true) {
    const blob = new Blob([addBom ? (CSV_BOM + text) : text], { type: "text/csv;charset=utf-8" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    a.remove();
    setTimeout(()=>URL.revokeObjectURL(url), 4000);
  }
  function normalizeBool(v) {
    const s = String(v ?? "").trim().toLowerCase();
    if (!s) return false;
    return ["1","true","t","yes","y","on","はい","有","○"].includes(s);
  }
  function parseCSV(text) {
    // RFC4180-ish parser (quotes + commas + CRLF)
    const rows = [];
    let row = [];
    let cur = "";
    let i = 0;
    let inQ = false;

    // remove BOM
    if (text && text.charCodeAt(0) === 0xFEFF) text = text.slice(1);

    while (i < text.length) {
      const ch = text[i];

      if (inQ) {
        if (ch === '"') {
          const nxt = text[i+1];
          if (nxt === '"') { cur += '"'; i += 2; continue; }
          inQ = false; i++; continue;
        } else {
          cur += ch; i++; continue;
        }
      } else {
        if (ch === '"') { inQ = true; i++; continue; }
        if (ch === ",") { row.push(cur); cur = ""; i++; continue; }
        if (ch === "\r") {
          if (text[i+1] === "\n") i++;
          row.push(cur); rows.push(row);
          row = []; cur = ""; i++; continue;
        }
        if (ch === "\n") {
          row.push(cur); rows.push(row);
          row = []; cur = ""; i++; continue;
        }
        cur += ch; i++; continue;
      }
    }
    row.push(cur);
    // ignore trailing empty line
    const isAllEmpty = row.every(x => String(x).trim() === "");
    if (!isAllEmpty) rows.push(row);
    return rows;
  }

  function downloadBlob(filename, blob) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  }


  // ===== “日だけ”入力パース =====
  function parseDayList(text, lastDay) {
    const t = (text||"").trim();
    if (!t) return [];
    const parts = t.split(",").map(s => s.trim()).filter(Boolean);
    const out = [];
    for (const p of parts) {
      const num = parseInt(String(p).replace(/[^\d]/g,""), 10);
      if (!Number.isFinite(num)) continue;
      if (num >= 1 && num <= lastDay) out.push(num);
    }
    return Array.from(new Set(out));
  }

  // 例: "5=日,12=夜" -> idxMap[4]="DAY", idxMap[11]="NIGHT"
  function parseWishShift(text, lastDay) {
    const t = (text||"").trim();
    const idxMap = Array(lastDay).fill(null);
    if (!t) return idxMap;
    const parts = t.split(",").map(s=>s.trim()).filter(Boolean);
    for (const p of parts) {
      const [lhs, rhsRaw] = p.split("=").map(x => (x||"").trim());
      if (!lhs || !rhsRaw) continue;
      const dayNum = parseInt(String(lhs).replace(/[^\d]/g,""), 10);
      if (!Number.isFinite(dayNum) || dayNum < 1 || dayNum > lastDay) continue;
      const rhs = rhsRaw.replace(/\s/g,"");
      if (rhs === "日") idxMap[dayNum-1] = SHIFT.DAY;
      else if (rhs === "夜") idxMap[dayNum-1] = SHIFT.NIGHT;
    }
    return idxMap;
  }

  function buildDays(year, month1to12, holidaysIdxSet) {
    const m = month1to12;
    const last = new Date(year, m, 0).getDate();
    const days = [];
    for (let d=1; d<=last; d++) {
      const dt = new Date(year, m-1, d);
      const dow = dt.getDay();
      const dateStr = ymd(year,m,d);
      const isWeekend = (dow === 0 || dow === 6);
      const isHoliday = holidaysIdxSet.has(d-1);
      const isHolOrWeekend = isWeekend || isHoliday;
      days.push({ dateStr, d, dow, isWeekend, isHoliday, isHolOrWeekend });
    }
    return days;
  }
  function deepCopySchedule(s) { return { assign: s.assign.map(row => row.slice()) }; }

  // ===== D勤/D休（表示用） =====
  function computeDMarks(schedule, days, employees) {
    const nEmp = employees.length;
    const nDay = days.length;
    const dEarnMark = Array.from({length:nEmp}, ()=> Array(nDay).fill(false));
    const dOffMark  = Array.from({length:nEmp}, ()=> Array(nDay).fill(false));

    for (let e=0; e<nEmp; e++) for (let i=0; i<nDay; i++) {
      if (schedule.assign[e][i] === SHIFT.D_OFF) dOffMark[e][i] = true;
    }

    for (let e=0; e<nEmp; e++) {
      if (!employees[e].proper) continue;
      for (let i=0; i<nDay; i++) {
        const sh = schedule.assign[e][i];
        const day = days[i];
        if (sh === SHIFT.DAY && day.isHolOrWeekend) dEarnMark[e][i] = true;
        if (sh === SHIFT.NIGHT && day.isHoliday) dEarnMark[e][i] = true;
        if (sh === SHIFT.NIGHT && i+1 < nDay) {
          const endDay = days[i+1];
          if (endDay.isHolOrWeekend) dEarnMark[e][i+1] = true;
        }
      }
    }
    return { dEarnMark, dOffMark };
  }

  function computeDCounts(schedule, days, employees) {
    const nEmp = employees.length;
    const nDay = days.length;
    const dEarn = Array(nEmp).fill(0);
    const dOff  = Array(nEmp).fill(0);

    for (let e=0; e<nEmp; e++) for (let i=0; i<nDay; i++) {
      if (schedule.assign[e][i] === SHIFT.D_OFF) dOff[e]++;
    }

    for (let e=0; e<nEmp; e++) {
      if (!employees[e].proper) continue;
      for (let i=0; i<nDay; i++) {
        const sh = schedule.assign[e][i];
        const day = days[i];
        if (sh === SHIFT.DAY && day.isHolOrWeekend) dEarn[e]++;
        if (sh === SHIFT.NIGHT && day.isHoliday) dEarn[e]++;
        if (sh === SHIFT.NIGHT && i+1 < nDay) {
          const endDay = days[i+1];
          if (endDay.isHolOrWeekend) dEarn[e]++;
        }
      }
    }
    return { dEarn, dOff };
  }

  // ===== ハード違反（表示用） =====
  function hardViolations(schedule, days, employees, rules, hardReqOffEnabled) {
    const nEmp = employees.length;
    const nDay = days.length;
    const issues = [];
    const cellFlags = Array.from({length:nEmp}, () => Array(nDay).fill(null));

    // staffing
    for (let i=0; i<nDay; i++) {
      let dayCnt = 0, nightCnt = 0;
      for (let e=0; e<nEmp; e++) {
        const sh = schedule.assign[e][i];
        if (sh === SHIFT.DAY) dayCnt++;
        if (sh === SHIFT.NIGHT) nightCnt++;
      }
      if (nightCnt !== REQUIRED_NIGHT) issues.push({type:"staff", day:i});
      if (dayCnt < MIN_DAY) issues.push({type:"staff", day:i});
      if (dayCnt > MAX_DAY) issues.push({type:"staff", day:i});
    }

    // per-emp
    for (let e=0; e<nEmp; e++) {
      let consecDay = 0;
      let nightSetRun = 0;

      for (let i=0; i<nDay; i++) {
        const sh = schedule.assign[e][i];

        // 希望休ハード + 有給（常にハード）
        const isReqOffHard = !!(hardReqOffEnabled && employees[e].reqOffIdxSet && employees[e].reqOffIdxSet[i]);
        const isPaidHard = !!(employees[e].paidIdxSet && employees[e].paidIdxSet[i]);
        if (isReqOffHard || isPaidHard) {
          if (sh !== SHIFT.OFF && sh !== SHIFT.D_OFF) {
            issues.push({type:"reqoff", emp:e, day:i});
            cellFlags[e][i] = "bad";
          }
        }

        if (sh === SHIFT.DAY) consecDay++; else consecDay = 0;
        // ★日勤5連続の翌日は「休」必須（夜勤も禁止）
        if (sh === SHIFT.NIGHT) {
          let run = 0;
          for (let k=i-1; k>=0 && schedule.assign[e][k]===SHIFT.DAY; k--) run++;
          if (run >= MAX_CONSEC_DAY) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }
        }
        if (consecDay > MAX_CONSEC_DAY) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }

        if (employees[e].proper && sh === SHIFT.NIGHT && days[i].isHolOrWeekend) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }

        if (sh === SHIFT.NIGHT) {
          if (i + 1 < nDay && schedule.assign[e][i+1] !== SHIFT.AKE) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }
        }

        if (i > 0 && schedule.assign[e][i-1] === SHIFT.NIGHT && sh === SHIFT.NIGHT) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }

        if (NO_DAY_AFTER_AKE && i > 0 && schedule.assign[e][i-1] === SHIFT.AKE && sh === SHIFT.DAY) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }

        if (sh === SHIFT.NIGHT) {
          const continues = (i >= 2 && schedule.assign[e][i-1] === SHIFT.AKE && schedule.assign[e][i-2] === SHIFT.NIGHT);
          nightSetRun = continues ? (nightSetRun + 1) : 1;
          if (nightSetRun > MAX_NIGHT_SET_RUN) { issues.push({type:"rule", emp:e, day:i}); cellFlags[e][i]="bad"; }
        }
        if (sh !== SHIFT.NIGHT && sh !== SHIFT.AKE) nightSetRun = 0;
      }
    }

    // hours
    for (let e=0; e<nEmp; e++) {
      let total = 0;
      for (let i=0; i<nDay; i++) total += (WORK_MIN[schedule.assign[e][i]] ?? 0);
      if (rules.minMin > 0 && total < rules.minMin) issues.push({type:"hours", emp:e});
      if (rules.maxMin > 0 && total > rules.maxMin) issues.push({type:"hours", emp:e});
    }

    const { dEarn, dOff } = computeDCounts(schedule, days, employees);
    for (let e=0; e<nEmp; e++) {
      if (!employees[e].proper) continue;
      if (dEarn[e] > MAX_D_DAYS) issues.push({type:"proper", emp:e});
      if (dOff[e] !== dEarn[e]) issues.push({type:"proper", emp:e});
    }

    return { issues, cellFlags };
  }

  function renderStatusChips(issues) {
    const chips = document.getElementById("statusChips");
    chips.innerHTML = "";
    const counts = {
      staff: issues.filter(x=>x.type==="staff").length,
      rule: issues.filter(x=>x.type==="rule").length,
      hours: issues.filter(x=>x.type==="hours").length,
      proper: issues.filter(x=>x.type==="proper").length,
      reqoff: issues.filter(x=>x.type==="reqoff").length,
    };
    const total = issues.length;

    const mk = (text, cls) => {
      const c = document.createElement("div");
      c.className = `chip ${cls||""}`;
      c.textContent = text;
      return c;
    };

    if (total===0) { chips.appendChild(mk("ハード違反なし（OK）","ok")); return; }
    chips.appendChild(mk(`ハード違反: ${total}`,"bad"));
    chips.appendChild(mk(`人数: ${counts.staff}` , counts.staff? "bad":""));
    chips.appendChild(mk(`勤務ルール: ${counts.rule}` , counts.rule? "bad":""));
    chips.appendChild(mk(`希望休: ${counts.reqoff}` , counts.reqoff? "bad":""));
    chips.appendChild(mk(`労働時間: ${counts.hours}` , counts.hours? "warn":""));
    chips.appendChild(mk(`D勤/消化: ${counts.proper}` , counts.proper? "bad":""));
  }

  // ===== 社員入力UI =====
  function renderEmployees() {
    const root = document.getElementById("empList");
    root.innerHTML = "";



    const cntEl = document.getElementById("empCount");
    if (cntEl) cntEl.textContent = `(${state.employees.length}名)`;
    state.employees.forEach((emp, idx) => {
      const block = document.createElement("div");
      block.className = "empBlock";

      const top = document.createElement("div");
      top.className = "empTop";

      const name = document.createElement("input");
      name.type = "text";
      name.value = emp.name;
      name.addEventListener("input", () => {
        emp.name = name.value.trim() || `社員${idx+1}`;
        renderGrid();
        renderCases();
      });

      const wdp = document.createElement("label");
      wdp.style.minWidth = "auto";
      wdp.style.display = "flex";
      wdp.style.alignItems = "center";
      wdp.style.gap = "6px";
      wdp.style.color = "var(--muted)";
      const wdpCb = document.createElement("input");
      wdpCb.type = "checkbox";
      wdpCb.checked = emp.weekdayDayPriority;
      wdpCb.addEventListener("change", () => { emp.weekdayDayPriority = wdpCb.checked; });
      wdp.appendChild(wdpCb);
      wdp.appendChild(document.createTextNode("平日日勤優先"));

      const proper = document.createElement("label");
      proper.style.minWidth = "auto";
      proper.style.display = "flex";
      proper.style.alignItems = "center";
      proper.style.gap = "6px";
      proper.style.color = "var(--muted)";
      const properCb = document.createElement("input");
      properCb.type = "checkbox";
      properCb.checked = emp.proper;
      properCb.addEventListener("change", () => { emp.proper = properCb.checked; });
      proper.appendChild(properCb);
      proper.appendChild(document.createTextNode("プロパー"));

      top.appendChild(name);
      top.appendChild(wdp);
      top.appendChild(proper);

      const prev = document.createElement("label");
      prev.style.minWidth = "auto";
      prev.style.display = "flex";
      prev.style.alignItems = "center";
      prev.style.gap = "6px";
      prev.style.color = "var(--muted)";
      const prevCb = document.createElement("input");
      prevCb.type = "checkbox";
      prevCb.checked = !!emp.prevMonthNight;
      prevCb.addEventListener("change", () => { emp.prevMonthNight = prevCb.checked; if (state.currentSchedule) renderGrid(); });
      prev.appendChild(prevCb);
      prev.appendChild(document.createTextNode("前月末夜勤→1日明け固定"));

      top.appendChild(prev);

      const grid = document.createElement("div");
      grid.className = "empGrid";

      const offHint = document.createElement("div");
      offHint.className = "hint";
      offHint.textContent = "希望休（“日”のみ / カンマ区切り）例: 3,15,28";
      const offTa = document.createElement("textarea");
      offTa.value = emp.requestsOffText || "";
      offTa.placeholder = "3,15,28";
      offTa.addEventListener("input", () => { emp.requestsOffText = offTa.value; if (state.currentSchedule) renderGrid(); });

      const wishHint = document.createElement("div");
      wishHint.className = "hint";
      wishHint.textContent = "希望勤務（“日”のみ）例: 5=日,12=夜";
      const wishTa = document.createElement("textarea");
      wishTa.value = emp.wishShiftText || "";
      wishTa.placeholder = "5=日,12=夜";
      wishTa.addEventListener("input", () => { emp.wishShiftText = wishTa.value; });

      grid.appendChild(offHint);
      grid.appendChild(offTa);
      
const paidHint = document.createElement("div");
paidHint.className = "hint";
paidHint.textContent = "有給（“日”のみ / カンマ区切り）例: 7,18";
const paidTa = document.createElement("textarea");
paidTa.value = emp.paidLeaveText || "";
paidTa.placeholder = "7,18";
paidTa.addEventListener("input", () => { emp.paidLeaveText = paidTa.value; if (state.currentSchedule) renderGrid(); });

grid.appendChild(paidHint);
grid.appendChild(paidTa);
grid.appendChild(wishHint);
      grid.appendChild(wishTa);

      block.appendChild(top);
      block.appendChild(grid);
      root.appendChild(block);
    });
  }

  // ===== 入力反映 =====
  function readInputs() {
    const { yy, mm } = parseMonthInput(document.getElementById("month").value);
    state.year = yy; state.month = mm;

    const stdHr = Number(document.getElementById("stdHr").value || 0);
    const minHr = Number(document.getElementById("minHr").value || 0);
    const maxHr = Number(document.getElementById("maxHr").value || 0);
    state.monthRules.stdMin = Math.round(stdHr * 60);
    state.monthRules.minMin = Math.round(minHr * 60);
    state.monthRules.maxMin = Math.round(maxHr * 60);

    const last = new Date(state.year, state.month, 0).getDate();
    const holDays = parseDayList(document.getElementById("holidays").value.trim(), last);
    state.holidaysIdx = new Set(holDays.map(d => d-1));
    state.days = buildDays(state.year, state.month, state.holidaysIdx);
  }

  // ===== Worker に渡す社員配列を作る（reqOffIdxSet / wishShiftIdxMap を含める） =====
  function buildEmployeesPayloadAndView() {
    const lastDay = state.days.length;

    const view = state.employees.map(e => {
      const reqDays = parseDayList(e.requestsOffText || "", lastDay);
      const reqArr = Array(lastDay).fill(false);
      for (const d of reqDays) reqArr[d-1] = true;

            const paidDays = parseDayList(e.paidLeaveText || "", lastDay);
      const paidArr = Array(lastDay).fill(false);
      for (const d of paidDays) paidArr[d-1] = true;

      const wishIdxMap = parseWishShift(e.wishShiftText || "", lastDay);

      return {
        id: e.id,
        name: e.name,
        weekdayDayPriority: !!e.weekdayDayPriority,
        proper: !!e.proper,
        prevMonthNight: !!e.prevMonthNight,
        reqOffIdxSet: reqArr,
        wishShiftIdxMap: wishIdxMap,
        paidIdxSet: paidArr,
      };
    });

    state.parsedEmployeesForView = view;

    return view.map(v => ({
      id: v.id,
      name: v.name,
      weekdayDayPriority: v.weekdayDayPriority,
      proper: v.proper,
      prevMonthNight: v.prevMonthNight,
      reqOffIdxSet: v.reqOffIdxSet,
      wishShiftIdxMap: v.wishShiftIdxMap,
      paidIdxSet: v.paidIdxSet
    }));
  }

  // ===== 微調整（編集） =====
  function canEditSet(e, i, sh) {
    const hardReqOffEnabled = document.getElementById("hardReqOffEnabled").checked;
    const paidMode = getPaidMode();
    const empV = state.parsedEmployeesForView?.[e];
    if (!empV) return true;
    // 前月末夜勤 → 当月1日目は「明け」固定（編集不可）
    if (i === 0 && empV.prevMonthNight) return false;

    // ★日勤5連続の翌日は「休」必須（夜勤も禁止）
    if (sh === SHIFT.NIGHT && state.currentSchedule) {
      let run = 0;
      for (let k=i-1; k>=0 && state.currentSchedule.assign[e][k]===SHIFT.DAY; k--) run++;
      if (run >= MAX_CONSEC_DAY) return false;
    }

    const isPaid = !!(empV.paidIdxSet && empV.paidIdxSet[i]);
    if ((hardReqOffEnabled && empV.reqOffIdxSet && empV.reqOffIdxSet[i]) || isPaid) {
      return (sh === SHIFT.OFF || sh === SHIFT.D_OFF);
    }
    if (sh === SHIFT.NIGHT && i+1 < state.days.length) {
      const nxtIsReqOff = !!(hardReqOffEnabled && empV.reqOffIdxSet && empV.reqOffIdxSet[i+1]);
      const nxtIsPaid = !!(empV.paidIdxSet && empV.paidIdxSet[i+1]);
      if (nxtIsReqOff || nxtIsPaid) return false;
    }
    return true;
  }

  function normalizeAfterEdit(e, i) {
    const s = state.currentSchedule;
    if (!s) return;
    const nDay = state.days.length;

    const empView = state.parsedEmployeesForView?.[e];
    if (i === 0 && empView?.prevMonthNight) {
      s.assign[e][0] = SHIFT.AKE;
      return;
    }

    if (s.assign[e][i] === SHIFT.NIGHT && i+1 < nDay) {
      s.assign[e][i+1] = SHIFT.AKE;
    }

    if (i > 0 && s.assign[e][i-1] === SHIFT.NIGHT) {
      s.assign[e][i] = SHIFT.AKE;
    }

    const hardReqOffEnabled = document.getElementById("hardReqOffEnabled").checked;
    // empView is already defined above
    const isReqOff = !!(hardReqOffEnabled && empView?.reqOffIdxSet?.[i]);
    const isPaid = !!(empView?.paidIdxSet?.[i]);
    if (isReqOff || isPaid) {
      s.assign[e][i] = SHIFT.OFF;
      if (i+1 < nDay && s.assign[e][i+1] === SHIFT.AKE) s.assign[e][i+1] = SHIFT.OFF;
      if (i>0 && s.assign[e][i-1] === SHIFT.NIGHT) s.assign[e][i-1] = SHIFT.OFF;
    }
  }

  function openEditPop(e, i) {
    if (!state.currentSchedule) return;
    const pop = document.getElementById("editPop");
    const t = document.getElementById("editTitle");
    const d = state.days[i];
    t.textContent = `${state.employees[e].name} / ${d.d}日`;
    state.editTarget = { e, i };
    pop.style.display = "block";
  }
  function closeEditPop() {
    document.getElementById("editPop").style.display = "none";
    state.editTarget = null;
  }

  // ===== 描画（折り返し：前半/後半） =====
  function renderGrid() {
    const wrap = document.getElementById("gridWrap");
    wrap.innerHTML = "";
    if (!state.currentSchedule) {
      wrap.innerHTML = `<div class="small" style="padding:12px;">左の「自動生成」を押すとシフト表が出ます。</div>`;
      return;
    }

    readInputs();
    buildEmployeesPayloadAndView();

    const rules = state.monthRules;
    const hardReqOffEnabled = document.getElementById("hardReqOffEnabled").checked;

    const empsView = state.parsedEmployeesForView;
    const hv = hardViolations(state.currentSchedule, state.days, empsView, rules, hardReqOffEnabled);
    const { dEarn, dOff } = computeDCounts(state.currentSchedule, state.days, empsView);
    const { dEarnMark, dOffMark } = computeDMarks(state.currentSchedule, state.days, empsView);

    const lastDay = state.days.length;
    const split = 15;
    const ranges = [
      { title: "前半（1〜15）", start: 0, end: Math.min(split, lastDay) },
      { title: "後半（16〜末）", start: Math.min(split, lastDay), end: lastDay }
    ];
    const dowJP = ["日","月","火","水","木","金","土"];

    function makeSectionTitle(text, sub) {
      const div = document.createElement("div");
      div.className = "sectionTitle";
      const l = document.createElement("div");
      l.className = "left";
      l.textContent = text;
      const r = document.createElement("div");
      r.className = "right";
      r.textContent = sub || "";
      div.appendChild(l);
      div.appendChild(r);
      return div;
    }

    function make2RowHeader(tr1, tr2, range, withCounts) {
      const corner = document.createElement("th");
      corner.className = "corner";
      corner.rowSpan = 2;
      corner.textContent = "社員";
      tr1.appendChild(corner);

      for (let i = range.start; i < range.end; i++) {
        const d = state.days[i];

        const thDate = document.createElement("th");
        thDate.textContent = `${d.d}`;
        thDate.classList.add("dayCol","dayColTop");
        tr1.appendChild(thDate);

        const thDow = document.createElement("th");
        thDow.textContent = `${dowJP[d.dow]}`;
        thDow.classList.add("dayCol");
        tr2.appendChild(thDow);
      }

      if (withCounts) {
        const thCount = document.createElement("th");
        thCount.rowSpan = 2;
        thCount.textContent = "日/夜/休(明除)/明";
        thCount.classList.add("metaTh","countDivider");
        const thSum = document.createElement("th");
        thSum.rowSpan = 2;
        thSum.textContent = "合計(時間)";
        thSum.classList.add("metaTh");
        const thD = document.createElement("th");
        thD.rowSpan = 2;
        thD.textContent = "D勤/D休";
        thD.classList.add("metaTh");
        tr1.appendChild(thCount);
        tr1.appendChild(thSum);
        tr1.appendChild(thD);
      }
    }

    function makeMetaCell(text, extraClass) {
      const td = document.createElement("td");
      td.className = `metaCell ${extraClass || ""}`.trim();
      const inner = document.createElement("div");
      inner.className = "metaIn";
      inner.textContent = text;
      td.appendChild(inner);
      return td;
    }

    function makeTable(range, withCounts) {
      const paidMode = getPaidMode(); // ★有給モード（表示/時間加算）
      const tbl = document.createElement("table");
      const thead = document.createElement("thead");
      const trDate = document.createElement("tr");
      const trDow  = document.createElement("tr");
      make2RowHeader(trDate, trDow, range, withCounts);
      thead.appendChild(trDate);
      thead.appendChild(trDow);
      tbl.appendChild(thead);

      const tbody = document.createElement("tbody");

      state.employees.forEach((emp, e) => {
        const tr = document.createElement("tr");
        const th = document.createElement("th");
        th.textContent = emp.name;
        tr.appendChild(th);

        // カウント（全期間）
        let totalMinAll = 0;
        let cntDayAll = 0, cntNightAll = 0, cntOffNoAkeAll = 0, cntAkeAll = 0;
        for (let i=0;i<lastDay;i++){
          const shAll = state.currentSchedule.assign[e][i];

          // 前月末夜勤 → 当月1日は「明け」固定（当月のカウンタ対象外）
          if (i === 0 && empsView[e].prevMonthNight && shAll === SHIFT.AKE) continue;

          {
            let addMin = (WORK_MIN[shAll] ?? 0);
            // 有給モード2: 休み扱いだが 8h(=日勤) を労働時間に加算
            if (paidMode === 2 && empsView[e].paidIdxSet && empsView[e].paidIdxSet[i] && (shAll === SHIFT.OFF || shAll === SHIFT.D_OFF)) {
              addMin += 480;
            }
            totalMinAll += addMin;
          }
          if (shAll === SHIFT.DAY) cntDayAll++;
          else if (shAll === SHIFT.NIGHT) cntNightAll++;
          else if (shAll === SHIFT.AKE) cntAkeAll++;
          else cntOffNoAkeAll++;
        }

        for (let i = range.start; i < range.end; i++) {
          const d = state.days[i];
          const td = document.createElement("td");
          const sh = state.currentSchedule.assign[e][i];

          td.classList.add("dayCol");

          if (i === 0 && empsView[e].prevMonthNight) {
            td.classList.add("locked");
            td.title = "前月末夜勤のため、当月1日は明け固定（カウント対象外）";
          }

          if (sh === SHIFT.DAY) td.classList.add("shift-day");
          else if (sh === SHIFT.NIGHT) td.classList.add("shift-night");
          else if (sh === SHIFT.AKE) td.classList.add("shift-ake");
          else td.classList.add("shift-off");

          if (empsView[e].reqOffIdxSet && empsView[e].reqOffIdxSet[i]) td.classList.add("reqoff");

          if (d.isWeekend) td.classList.add("weekend");
          if (d.isHoliday) td.classList.add("holiday");

          const isDEarned = dEarnMark[e][i];
          const isDOff = dOffMark[e][i];

          if (isDEarned) td.classList.add("d-earned");
          if (isDOff) td.classList.add("d-off");

          const flag = hv.cellFlags[e][i];
          if (flag==="bad") td.classList.add("bad");

          const wrapCell = document.createElement("div");
          wrapCell.className = "cellWrap";

          const cell2 = document.createElement("div");
          cell2.className = "cell2";

          const left = document.createElement("div");
          left.className = "half left";
          const right = document.createElement("div");
          right.className = "half right";

          if (sh === SHIFT.DAY) left.textContent = "日";
          if (sh === SHIFT.NIGHT) right.textContent = "夜";

          const badge = document.createElement("div");
          badge.className = "centerBadge";
          if (sh === SHIFT.AKE) badge.textContent = "明";
          else if (sh === SHIFT.D_OFF) badge.textContent = "休(D)";
          else if (paidMode !== "none" && empsView[e].paidIdxSet && empsView[e].paidIdxSet[i]) badge.textContent = "有";
          else badge.textContent = "";

          cell2.appendChild(left);
          cell2.appendChild(right);
          wrapCell.appendChild(cell2);
          wrapCell.appendChild(badge);
          td.appendChild(wrapCell);

          td.addEventListener("click", () => {
            if (!document.getElementById("editMode").checked) return;
            openEditPop(e, i);
          });

          tr.appendChild(td);
        }

        if (withCounts) {
          const tdCount = makeMetaCell(`日${cntDayAll}/夜${cntNightAll}/休${cntOffNoAkeAll}/明${cntAkeAll}`, "countDivider");
          tr.appendChild(tdCount);

          const tdSum = makeMetaCell((totalMinAll / 60).toFixed(1), "");
          if (rules.minMin>0 && totalMinAll<rules.minMin) tdSum.classList.add("bad");
          if (rules.maxMin>0 && totalMinAll>rules.maxMin) tdSum.classList.add("bad");
          tr.appendChild(tdSum);

          const tdD = makeMetaCell(emp.proper ? `D勤${dEarn[e]}/D休${dOff[e]}` : "-", "");
          if (emp.proper && (dEarn[e]!==dOff[e] || dEarn[e]>MAX_D_DAYS)) tdD.classList.add("bad");
          tr.appendChild(tdD);
        }

        tbody.appendChild(tr);
      });

      const trStaff = document.createElement("tr");
      const thStaff = document.createElement("th");
      thStaff.textContent = "人数";
      trStaff.appendChild(thStaff);

      for (let i = range.start; i < range.end; i++) {
        let dayCnt=0, nightCnt=0;
        for (let e=0;e<state.employees.length;e++){
          if (state.currentSchedule.assign[e][i]===SHIFT.DAY) dayCnt++;
          if (state.currentSchedule.assign[e][i]===SHIFT.NIGHT) nightCnt++;
        }
        const td = document.createElement("td");
        td.classList.add("dayCol","dayColBottom");
        td.style.padding = "0 6px";
        td.textContent = `日${dayCnt}/夜${nightCnt}`;
        if (nightCnt!==REQUIRED_NIGHT || dayCnt<MIN_DAY || dayCnt>MAX_DAY) td.classList.add("bad");
        trStaff.appendChild(td);
      }

      if (withCounts) {
        trStaff.appendChild(makeMetaCell("", "countDivider"));
        trStaff.appendChild(makeMetaCell("", ""));
        trStaff.appendChild(makeMetaCell("", ""));
      }

      tbody.appendChild(trStaff);

      tbl.appendChild(tbody);
      return tbl;
    }

    wrap.appendChild(makeSectionTitle(ranges[0].title, "（前半テーブル）"));
    wrap.appendChild(makeTable(ranges[0], false));

    if (ranges[1].start < ranges[1].end) {
      wrap.appendChild(makeSectionTitle(ranges[1].title, "（後半テーブル：ここにカウント列）"));
      wrap.appendChild(makeTable(ranges[1], true));
    } else {
      wrap.innerHTML = "";
      wrap.appendChild(makeSectionTitle("（この月は15日までのため1表表示）", ""));
      wrap.appendChild(makeTable(ranges[0], true));
    }

    renderStatusChips(hv.issues);
  }

  function renderCases() {
    const list = document.getElementById("caseList");
    list.innerHTML = "";

    if (!state.candidates.length) {
      list.innerHTML = `<div class="small">まだ候補がありません。左の「自動生成」を押してください。</div>`;
      return;
    }

    state.candidates.forEach((c, idx) => {
      const card = document.createElement("div");
      card.className = "caseCard";

      const h3 = document.createElement("h3");
      const left = document.createElement("span");
      left.textContent = `候補 ${idx+1}`;
      const right = document.createElement("span");
      right.textContent = `score ${Math.round(c.score)}`;
      h3.appendChild(left); h3.appendChild(right);

      const meta = document.createElement("div");
      meta.className = "meta";
      meta.innerHTML = `
        <span>平均: ${c.summary.avgHr}h</span>
        <span>最小: ${c.summary.minHr}h</span>
        <span>最大: ${c.summary.maxHr}h</span>
      `;
      const meta2 = document.createElement("div");
      meta2.className = "meta";
      meta2.textContent = c.summary.properInfo ? `プロパー: ${c.summary.properInfo}` : "プロパー: -";

      const btn = document.createElement("button");
      btn.textContent = "この候補を表示";
      btn.className = "primary";
      btn.addEventListener("click", () => {
        state.currentSchedule = deepCopySchedule(c.schedule);
        renderGrid();
      });

      card.appendChild(h3);
      card.appendChild(meta);
      card.appendChild(meta2);
      card.appendChild(btn);
      list.appendChild(card);
    });
  }

  // ===== CSV：社員 出力/読込 =====
  function exportEmployeesCSV() {
    // 保存したいのは「入力値」なので state.employees をそのまま吐く
    const rows = [];
    rows.push(["id","name","weekdayDayPriority","proper","prevMonthNight","requestsOff","wishShift","paidLeave"]);
    state.employees.forEach((e) => {
      rows.push([
        e.id ?? "",
        e.name ?? "",
        e.weekdayDayPriority ? "1" : "0",
        e.proper ? "1" : "0",
        e.prevMonthNight ? "1" : "0",
        (e.requestsOffText ?? "").trim(),
        (e.wishShiftText ?? "").trim(),
        (e.paidLeaveText ?? "").trim(),
      ]);
    });
    const csv = toCSV(rows);
    const ym = (document.getElementById("month").value || "employees").replace("-","");
    downloadText(`employees_${ym}.csv`, csv, true);
  }

  function importEmployeesCSVText(text) {
    const rows = parseCSV(text);
    if (!rows.length) throw new Error("CSVが空です");

    // header
    const head = rows[0].map(x => String(x ?? "").trim());
    const idx = (name) => head.findIndex(h => h.toLowerCase() === name.toLowerCase());
    const idxAny = (...names) => {
      for (const n of names) {
        const k = idx(n);
        if (k >= 0) return k;
      }
      return -1;
    };

    const prevIdx = idxAny("prevMonthNight","prev_month_night","prevNight","先月末夜勤","先月最終日夜勤");

    const paidIdx0 = idxAny("paidLeave","paidLeaveText","paid_leave","paid","有給","有給日","有給休暇","paidLeaveDays");

    const idI = idx("id");
    const nameI = idx("name");
    const wdpI = idx("weekdayDayPriority");
    const properI = idx("proper");
    const offI = idx("requestsOff");
    const wishI = idx("wishShift");

    // fallback (日本語ヘッダでも読めるように)
    const alt = {
      "社員id": idI, "社員": nameI, "社員名": nameI,
      "平日日勤優先": wdpI, "プロパー": properI,
      "希望休": offI, "希望勤務": wishI, "有給": paidIdx0, "有給日": paidIdx0
    };

    function pickI(keys, current) {
      if (current !== -1) return current;
      for (const k of keys) {
        const j = head.findIndex(h => h === k);
        if (j !== -1) return j;
      }
      return -1;
    }

    const idIdx = pickI(["社員id","ID"], idI);
    const nameIdx = pickI(["社員","社員名","Name"], nameI);
    const wdpIdx = pickI(["平日日勤優先","weekday","weekdayDay"], wdpI);
    const properIdx = pickI(["プロパー","proper"], properI);
    const offIdx = pickI(["希望休","requestsOff"], offI);
    const wishIdx = pickI(["希望勤務","wishShift"], wishI);
    const paidIdx = pickI(["有給","有給日","paidLeave","paidLeaveText","paid_leave"], paidIdx0);

    const emps = [];
    for (let r=1; r<rows.length; r++) {
      const row = rows[r];
      if (!row || row.every(x => String(x ?? "").trim() === "")) continue;

      const id = (idIdx !== -1 ? row[idIdx] : "") || `E${emps.length+1}`;
      const name = (nameIdx !== -1 ? row[nameIdx] : "") || `社員${emps.length+1}`;
      const weekdayDayPriority = normalizeBool(wdpIdx !== -1 ? row[wdpIdx] : "");
      const proper = normalizeBool(properIdx !== -1 ? row[properIdx] : "");
      const prevMonthNight = normalizeBool(prevIdx !== -1 ? row[prevIdx] : "");
      const requestsOffText = String(offIdx !== -1 ? row[offIdx] : "").trim();
      const wishShiftText = String(wishIdx !== -1 ? row[wishIdx] : "").trim();
      const paidLeaveText = String(paidIdx !== -1 ? row[paidIdx] : "").trim();

      emps.push({
        id: String(id).trim() || `E${emps.length+1}`,
        name: String(name).trim() || `社員${emps.length+1}`,
        weekdayDayPriority,
        proper,
        prevMonthNight,
        requestsOffText,
        wishShiftText,
        paidLeaveText,
      });
    }
    if (!emps.length) throw new Error("社員データ行が見つかりませんでした");

    state.employees = emps;

    // 既存の候補/結果は入力条件が変わるのでリセット（必要ならコメントアウト可）
    state.candidates = [];
    state.currentSchedule = null;

    renderEmployees();
    readInputs();
    buildEmployeesPayloadAndView();
    renderCases();
    renderGrid();
  }

  // ===== CSV：結果 出力 =====
  function shiftToJP(sh) {
    if (sh === SHIFT.DAY) return "日";
    if (sh === SHIFT.NIGHT) return "夜";
    if (sh === SHIFT.AKE) return "明";
    if (sh === SHIFT.D_OFF) return "休(D)";
    return "休";
  }
  
  // ===== 結果CSV出力（テキスト：簡易） =====
  function exportResultCSV() {
    if (!state.currentSchedule) { alert("先にシフトを生成してください"); return; }
    readInputs();
    buildEmployeesPayloadAndView();

    const days = state.days;
    const empsView = state.parsedEmployeesForView || [];
    const paidMode = getPaidMode(); // 0/1/2

    const dowJP = ["日","月","火","水","木","金","土"];
    const header = ["社員"].concat(days.map(d => `${d.d}(${dowJP[d.dow]})`));

    const rows = [header];

    for (let e=0; e<state.employees.length; e++){
      const empName = state.employees[e].name;
      const row = [empName];
      for (let i=0;i<days.length;i++){
        const sh = state.currentSchedule.assign[e][i];
        const v = empsView[e] || {};
        const isReq = !!(v.reqOffIdxSet && v.reqOffIdxSet[i]);
        const isPaid = !!(v.paidIdxSet && v.paidIdxSet[i]);

        // 表示寄せ（Webの見え方優先）
        let out = "";
        if (isPaid) out = (paidMode === 1 ? "有(見)" : (paidMode === 2 ? "有" : "有"));
        else if (isReq) out = "休(希)";
        else if (sh === SHIFT.DAY) out = "日";
        else if (sh === SHIFT.NIGHT) out = "夜";
        else if (sh === SHIFT.AKE) out = "明";
        else if (sh === SHIFT.D_OFF) out = "休(D)";
        else out = "休";

        row.push(out);
      }
      rows.push(row);
    }

    const csv = rows.map(r => r.map(x => `"${String(x).replaceAll('"','""')}"`).join(",")).join("\r\n");
    const blob = new Blob(["\uFEFF" + csv], { type: "text/csv;charset=utf-8" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = `shift_${state.year}${pad2(state.month)}.csv`;
    a.click();
    URL.revokeObjectURL(a.href);
  }

  // ===== 見た目付きエクスポート（Excel互換 .xls / 1日を半分に分ける / 前半+後半を全再現） =====
  function exportStyledExcel() {
    if (!state.currentSchedule) { alert("先にシフトを生成してください"); return; }
    readInputs();
    buildEmployeesPayloadAndView();

    const days = state.days;
    const empsView = state.parsedEmployeesForView || [];
    const paidMode = getPaidMode(); // 0/1/2

    const dowJP = ["日","月","火","水","木","金","土"];
    const lastDay = days.length;
    const split = 15;
    const ranges = [
      { title: `${state.month}月 上旬`, start: 0, end: Math.min(split, lastDay), withCounts: false },
      { title: `${state.month}月 下旬`, start: Math.min(split, lastDay), end: lastDay, withCounts: true },
    ];

    // 罫線・色（Web寄せ）
    const CSS = `
      <style>
        table{ border-collapse:collapse; font-family: "Segoe UI","Noto Sans JP",sans-serif; font-size:11px; }
        th,td{ border:1px solid #777; padding:2px 4px; text-align:center; height:18px; }
        th.name, td.name{ text-align:left; white-space:nowrap; min-width:90px; }
        th.secTitle{ background:#f6f8fb; font-weight:700; border:2px solid #000; }
        th.topHead{ background:#f6f8fb; font-weight:700; }
        .dayGroupL{ border-left:2px solid #000 !important; }
        .dayGroupR{ border-right:2px solid #000 !important; }
        .dayTop{ border-top:2px solid #000 !important; }
        .dayBottom{ border-bottom:2px solid #000 !important; }

        /* halves */
        td.halfL, th.halfL{ min-width:22px; }
        td.halfR, th.halfR{ min-width:22px; }

        .bgDay{ background:#ffd1e6; }     /* 日勤ピンク */
        .bgNight{ background:#cfe8ff; }   /* 夜勤薄青 */
        .bgAke{ background:#f1f1f1; }     /* 明け */
        .bgOff{ background:#eeeeee; }     /* 公休（薄灰） */
        .bgReq{ background:#dcdcdc; }     /* 希望休（濃灰） */
        .bgPaid{ background:#bfbfbf; }    /* 有給（さらに濃い灰） */

        .countHead{ background:#f6f8fb; font-weight:700; border-left:2px solid #000 !important; min-width:88px; }
        .countCell{ border-left:2px solid #000 !important; min-width:88px; }
        .sumCell{ min-width:70px; }
        .dCell{ min-width:80px; }
        .staffRow th{ background:#f6f8fb; font-weight:700; }
      </style>
    `;

    const esc = (s) => String(s ?? "").replaceAll("&","&amp;").replaceAll("<","&lt;").replaceAll(">","&gt;").replaceAll('"',"&quot;");

    const shiftLabel = (e,i) => {
      const sh = state.currentSchedule.assign[e][i];
      const v = empsView[e] || {};
      const isReq = !!(v.reqOffIdxSet && v.reqOffIdxSet[i]);
      const isPaid = !!(v.paidIdxSet && v.paidIdxSet[i]);

      if (isPaid) return { kind:"PAID", text:(paidMode===1 ? "有" : "有") };
      if (isReq)  return { kind:"REQ",  text:"希" };
      if (sh === SHIFT.DAY)   return { kind:"DAY", text:"日" };
      if (sh === SHIFT.NIGHT) return { kind:"NIGHT", text:"夜" };
      if (sh === SHIFT.AKE)   return { kind:"AKE", text:"明け" };
      if (sh === SHIFT.D_OFF) return { kind:"D_OFF", text:"D休" };
      return { kind:"OFF", text:"" };
    };

    const bgClassForKind = (kind) => {
      if (kind==="DAY") return "bgDay";
      if (kind==="NIGHT") return "bgNight";
      if (kind==="AKE") return "bgAke";
      if (kind==="REQ") return "bgReq";
      if (kind==="PAID") return "bgPaid";
      return "bgOff";
    };

    // counts (全期間)
    const calcCountsAll = (e) => {
      let totalMinAll = 0;
      let cntDayAll=0, cntNightAll=0, cntOffNoAkeAll=0, cntAkeAll=0;
      for (let i=0;i<lastDay;i++){
        const sh = state.currentSchedule.assign[e][i];
        const v = empsView[e] || {};
        const isPaid = !!(v.paidIdxSet && v.paidIdxSet[i]);
        const isPrevMonthAke = !!(v.prevMonthAkeIdxSet && v.prevMonthAkeIdxSet[i]); // 1日固定の明け（当月カウント除外）
        if (!isPrevMonthAke) {
          // 有給モード2：休み扱いだが8h加算
          if (isPaid && paidMode===2) totalMinAll += 480;
          else totalMinAll += (WORK_MIN[sh] ?? 0);
          if (sh === SHIFT.DAY) cntDayAll++;
          else if (sh === SHIFT.NIGHT) cntNightAll++;
          else if (sh === SHIFT.AKE) cntAkeAll++;
          else cntOffNoAkeAll++;
        }
      }
      return { totalMinAll, cntDayAll, cntNightAll, cntOffNoAkeAll, cntAkeAll };
    };

    const buildSectionTable = (range) => {
      if (range.start >= range.end) return "";
      const colSpanDays = (range.end - range.start) * 2;
      let html = "";

      // section title row
      html += `<table><thead>`;
      html += `<tr><th class="secTitle name" colspan="${1 + colSpanDays + (range.withCounts ? 3 : 0)}">${esc(range.title)}</th></tr>`;

      // header row 1: date (colspan=2 each)
      html += `<tr>`;
      html += `<th class="topHead name" rowspan="2">社員</th>`;
      for (let i=range.start;i<range.end;i++){
        const clsL = "dayGroupL dayTop";
        const clsR = "dayGroupR dayTop";
        html += `<th class="topHead ${clsL}" colspan="2">${esc(days[i].d)}</th>`;
      }
      if (range.withCounts){
        html += `<th class="countHead" rowspan="2">日/夜/休(明除)/明</th>`;
        html += `<th class="topHead sumCell" rowspan="2">合計(時間)</th>`;
        html += `<th class="topHead dCell" rowspan="2">D勤/D休</th>`;
      }
      html += `</tr>`;

      // header row 2: dow (colspan=2 each)
      html += `<tr>`;
      for (let i=range.start;i<range.end;i++){
        html += `<th class="topHead dayGroupL" colspan="2">${esc(dowJP[days[i].dow])}</th>`;
      }
      html += `</tr></thead><tbody>`;

      // employee rows
      for (let e=0; e<state.employees.length; e++){
        html += `<tr>`;
        html += `<td class="name">${esc(state.employees[e].name)}</td>`;

        for (let i=range.start;i<range.end;i++){
          const info = shiftLabel(e,i);

          // day group borders
          const baseL = "dayGroupL";
          const baseR = "dayGroupR";

          if (info.kind === "DAY"){
            html += `<td class="halfL ${baseL} ${bgClassForKind("DAY")}">${esc(info.text)}</td>`;
            html += `<td class="halfR ${baseR}"></td>`;
          } else if (info.kind === "NIGHT"){
            html += `<td class="halfL ${baseL}"></td>`;
            html += `<td class="halfR ${baseR} ${bgClassForKind("NIGHT")}">${esc(info.text)}</td>`;
          } else {
            const bg = bgClassForKind(info.kind);
            html += `<td class="${baseL} ${bg}" colspan="2">${esc(info.text)}</td>`;
          }
        }

        if (range.withCounts){
          const { totalMinAll, cntDayAll, cntNightAll, cntOffNoAkeAll, cntAkeAll } = calcCountsAll(e);
          html += `<td class="countCell">${esc(`日${cntDayAll}/夜${cntNightAll}/休${cntOffNoAkeAll}/明${cntAkeAll}`)}</td>`;
          html += `<td class="sumCell">${esc((totalMinAll/60).toFixed(1))}</td>`;
          const { dEarn, dOff } = computeDCounts(state.currentSchedule, days, empsView);
          html += `<td class="dCell">${esc(state.employees[e].proper ? `D勤${dEarn[e]}/D休${dOff[e]}` : "-")}</td>`;
        }

        html += `</tr>`;
      }

      // staff row
      html += `<tr class="staffRow"><th class="name">人数</th>`;
      for (let i=range.start;i<range.end;i++){
        let dayCnt=0, nightCnt=0;
        for (let e=0;e<state.employees.length;e++){
          const sh = state.currentSchedule.assign[e][i];
          if (sh===SHIFT.DAY) dayCnt++;
          if (sh===SHIFT.NIGHT) nightCnt++;
        }
        html += `
          <td class="dayGroupL dayBottom">${esc(dayCnt)}</td>
          <td class="dayGroupR dayBottom">${esc(nightCnt)}</td>
        `;
      }
      if (range.withCounts){
        html += `<td class="countCell"></td><td class="sumCell"></td><td class="dCell"></td>`;
      }
      html += `</tr>`;

      html += `</tbody></table><br/>`;
      return html;
    };

    const body = ranges.map(buildSectionTable).join("");

    const html = `
      <html><head><meta charset="utf-8">${CSS}</head>
      <body>${body}</body></html>
    `;

    const blob = new Blob([html], { type: "application/vnd.ms-excel;charset=utf-8" });
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = `shift_${state.year}${pad2(state.month)}_styled_half.xls`;
    a.click();
    URL.revokeObjectURL(a.href);
  }

const WORKER_CODE = document.getElementById("workerCode").textContent;

  let solverWorker = null;

  function setProgress(text) {
    const el = document.getElementById("progressText");
    if (el) el.textContent = text;
  }

  function stopWorker() {
    if (solverWorker) {
      solverWorker.terminate();
      solverWorker = null;
    }
    const btn = document.getElementById("btnGenerate");
    btn.disabled = false;
    btn.textContent = "自動生成（上位4案）";
  }

  function createSolverWorker() {
    const blob = new Blob([WORKER_CODE], { type: "text/javascript" });
    const url = URL.createObjectURL(blob);
    const w = new Worker(url);
    URL.revokeObjectURL(url);
    return w;
  }

  async function onGenerate() {
    const btn = document.getElementById("btnGenerate");
    btn.disabled = true;
    btn.textContent = "生成中…";
    setProgress("初期化中…");

    try {
      readInputs();

      state.candidates = [];
      renderCases();

      stopWorker();
      solverWorker = createSolverWorker();

      const limitSec = clamp(Number(document.getElementById("limitSec").value || 10), 1, 600);
      const limitMs = limitSec * 1000;

      const hardReqOffEnabled = !!document.getElementById("hardReqOffEnabled").checked;
      const paidMode = getPaidMode();

      const employeesPayload = buildEmployeesPayloadAndView();

      const payload = {
        type: "start",
        limitMs,
        requiredNight: REQUIRED_NIGHT,
        minDay: MIN_DAY,
        maxDay: MAX_DAY,
        targetDay: TARGET_DAY,
        maxConsecDay: MAX_CONSEC_DAY,
        maxNightSetRun: MAX_NIGHT_SET_RUN,
        noDayAfterAke: NO_DAY_AFTER_AKE,
        maxDDays: MAX_D_DAYS,
        workMin: WORK_MIN,
        monthRules: state.monthRules,
        days: state.days,
        employees: employeesPayload,
        hardReqOffEnabled,
        paidMode
      };

      solverWorker.onmessage = (ev) => {
        const msg = ev.data;
        if (!msg || !msg.type) return;

        if (msg.type === "progress") {
          setProgress(`探索中… ${msg.iter}回 / ${Math.round(msg.elapsedMs/1000)}秒経過 / 有効候補 ${msg.bestCount}件`);
          return;
        }

        if (msg.type === "done") {
          state.candidates = msg.candidates || [];
          state.currentSchedule = state.candidates.length ? deepCopySchedule(state.candidates[0].schedule) : null;

          renderCases();
          renderGrid();

          setProgress(`完了：有効候補 ${state.candidates.length}件（試行 ${msg.iter}回）`);
          stopWorker();

          if (!state.candidates.length) {
            alert(
              "有効な候補（ハード違反なし）が見つかりませんでした。\\n" +
              "探索時間を増やす / 月の最低・最高勤務時間を緩める、などを試してください。"
            );
          }
          return;
        }

        if (msg.type === "error") {
          console.error(msg.error);
          alert("Workerでエラーが発生しました:\\n" + msg.error);
          setProgress("エラー");
          stopWorker();
        }
      };

      solverWorker.onerror = (e) => {
        console.error(e);
        alert("Workerエラー:\\n" + (e.message || e));
        setProgress("エラー");
        stopWorker();
      };

      solverWorker.postMessage(payload);
    } catch (e) {
      console.error(e);
      alert("生成開始に失敗:\\n" + (e?.message || e));
      setProgress("エラー");
      stopWorker();
    }
  }

  // ===== 編集ポップ操作 =====
  function bindEditor() {
    const pop = document.getElementById("editPop");
    document.getElementById("editClose").addEventListener("click", closeEditPop);

    pop.querySelectorAll("button[data-sh]").forEach(btn => {
      btn.addEventListener("click", () => {
        if (!state.currentSchedule || !state.editTarget) return;

        const sh = btn.dataset.sh;
        const { e, i } = state.editTarget;

        readInputs();
        buildEmployeesPayloadAndView();

        if (!canEditSet(e, i, sh)) {
          alert("希望休ハード条件によりこの変更はできません。（夜の翌日が希望休など）");
          return;
        }

        state.currentSchedule.assign[e][i] = sh;
        normalizeAfterEdit(e, i);

        renderGrid();
        closeEditPop();
      });
    });

    document.addEventListener("keydown", (ev) => {
      if (ev.key === "Escape") closeEditPop();
    });
  }

  // ===== 社員追加/削除 =====
  function nextEmployeeId() {
    // 既存ID（E数字）から最大を拾う。なければ employees.length から推定
    let maxN = 0;
    for (const e of state.employees) {
      const m = String(e.id || "").match(/E(\d+)/);
      if (m) maxN = Math.max(maxN, Number(m[1]) || 0);
    }
    return `E${maxN + 1}`;
  }

  function addEmployee(prefill = {}) {
    const idx = state.employees.length;
    state.employees.push({
      id: prefill.id || nextEmployeeId(),
      name: prefill.name || `社員${idx + 1}`,
      weekdayDayPriority: !!prefill.weekdayDayPriority,
      proper: !!prefill.proper,
      requestsOffText: prefill.requestsOffText || "",
      wishShiftText: prefill.wishShiftText || "",
      prevMonthNight: !!prefill.prevMonthNight,
    });
    renderEmployees();
    if (state.currentSchedule) renderGrid();
  }

  function removeLastEmployee() {
    if (state.employees.length <= 1) {
      alert("これ以上削除できません。");
      return;
    }
    state.employees.pop();
    state.candidates = [];
    state.currentSchedule = null;
    renderEmployees();
    renderCases();
    renderGrid();
  }

  function initEmployees() {
    state.employees = Array.from({length:9}, (_,i)=>({
      id: `E${i+1}`,
      name: `社員${i+1}`,
      weekdayDayPriority: (i<4),
      proper: false,
      requestsOffText: "",
      wishShiftText: "",
      prevMonthNight: false,
    }));
  }

  function initMonthDefault() {
    const now = new Date();
    const y = now.getFullYear();
    const m = now.getMonth()+1;
    document.getElementById("month").value = `${y}-${pad2(m)}`;
  }

  function bind() {
    document.getElementById("btnGenerate").addEventListener("click", onGenerate);
    document.getElementById("btnStop").addEventListener("click", () => {
      stopWorker();
      setProgress("停止しました");
    });
    document.getElementById("btnPrint").addEventListener("click", () => window.print());

    // CSV buttons
    document.getElementById("btnEmpExport").addEventListener("click", exportEmployeesCSV);
    document.getElementById("btnEmpImport").addEventListener("click", () => document.getElementById("empCsvFile").click());
    document.getElementById("btnResultExport").addEventListener("click", exportResultCSV);
    document.getElementById("btnResultExportStyled").addEventListener("click", exportStyledExcel);

    // Employee add/remove
    document.getElementById("btnAddEmp").addEventListener("click", () => {
      addEmployee();
    });
    document.getElementById("btnRemoveEmp").addEventListener("click", () => {
      removeLastEmployee();
    });

    document.getElementById("empCsvFile").addEventListener("change", async (ev) => {
      const file = ev.target.files && ev.target.files[0];
      ev.target.value = "";
      if (!file) return;
      try {
        const text = await file.text();
        importEmployeesCSVText(text);
        alert("社員CSVを読み込みました。");
      } catch (e) {
        alert("社員CSVの読み込みに失敗しました:\\n" + (e?.message || e));
      }
    });

    ["month","stdHr","minHr","maxHr","holidays","hardReqOffEnabled","paidModeNone","paidMode1","paidMode2"].forEach(id => {
      document.getElementById(id).addEventListener("change", () => {
        readInputs();
        if (state.currentSchedule) {
          buildEmployeesPayloadAndView();
          renderGrid();
        }
      });
    });

    document.getElementById("editMode").addEventListener("change", (e) => {
      if (!e.target.checked) closeEditPop();
    });

    bindEditor();
  }

  function main() {
    initMonthDefault();
    initEmployees();
    renderEmployees();
    readInputs();
    buildEmployeesPayloadAndView();
    bind();
    renderGrid();
    renderCases();
  }

  main();
})();
</script>
</body>
</html>
```
