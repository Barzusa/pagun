<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fact Finding + Tax Planning</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <script type="importmap">
      {
        "imports": {
          "react": "https://esm.sh/react@18.2.0",
          "react-dom/client": "https://esm.sh/react-dom@18.2.0/client",
          "lucide-react": "https://esm.sh/lucide-react@0.292.0"
        }
      }
    </script>
</head>
<body class="bg-gray-50 text-gray-800 font-sans">
    <div id="root"></div>
    <script type="text/babel" data-type="module">
        import React, { useState, useMemo, useEffect, useRef, useCallback } from 'react';
        import { createRoot } from 'react-dom/client';
        import { 
            AlertCircle, ShieldCheck, HeartPulse, CheckCircle2, Circle, 
            Table, ClipboardList, GraduationCap, Sunset, User, TrendingUp, 
            Target, Home, Shield, Banknote, Receipt, Lightbulb 
        } from 'lucide-react';

        // ============================================================
        // TAX PLANNING DATA & HELPERS
        // ============================================================
        const TAX_BRACKETS = [
          { min: 0,        max: 150000,  rate: 0 },
          { min: 150001,   max: 300000,  rate: 0.05 },
          { min: 300001,   max: 500000,  rate: 0.10 },
          { min: 500001,   max: 750000,  rate: 0.15 },
          { min: 750001,   max: 1000000, rate: 0.20 },
          { min: 1000001,  max: 2000000, rate: 0.25 },
          { min: 2000001,  max: 5000000, rate: 0.30 },
          { min: 5000001,  max: Infinity, rate: 0.35 },
        ];

        function calcTax(taxableIncome) {
          if (taxableIncome <= 0) return 0;
          let tax = 0;
          for (const b of TAX_BRACKETS) {
            if (taxableIncome > b.min - 1) {
              const upper = Math.min(taxableIncome, b.max);
              tax += (upper - (b.min - 1)) * b.rate;
            }
          }
          return Math.max(0, tax);
        }

        function getMarginalRate(taxableIncome) {
          for (let i = TAX_BRACKETS.length - 1; i >= 0; i--) {
            if (taxableIncome > TAX_BRACKETS[i].min - 1) return TAX_BRACKETS[i].rate;
          }
          return 0;
        }

        const INCOME_TYPE_OPTIONS = [
          { value: '40_1', label: 'ม.40(1) เงินเดือน/ค่าจ้าง', rate: 0.5, max: 100000 },
          { value: '40_2', label: 'ม.40(2) รับจ้างทั่วไป', rate: 0.5, max: 100000 },
          { value: '40_3', label: 'ม.40(3) ค่าลิขสิทธิ์', rate: 0.5, max: 100000 },
          { value: '40_5', label: 'ม.40(5) ค่าเช่าทรัพย์สิน', rate: 0.3, max: null },
          { value: '40_6', label: 'ม.40(6) วิชาชีพอิสระ', rate: 0.6, max: null },
          { value: '40_7', label: 'ม.40(7) รับเหมา', rate: 0.6, max: null },
          { value: '40_8', label: 'ม.40(8) รายได้อื่นๆ', rate: 0.6, max: null },
        ];

        // ============================================================
        // EDUCATION DATA
        // ============================================================
        const EDU_LEVELS = [
          { key:'K',      label:'🏫 ระดับก่อนประถมศึกษา (อนุบาล)',            startAge:4,  years:3, feeDefault:100000 },
          { key:'Pri',    label:'🎒 ระดับประถมศึกษา',          startAge:7,  years:6, feeDefault:80000 },
          { key:'SecLow', label:'📘 ระดับมัธยมศึกษาตอนต้น',    startAge:13, years:3, feeDefault:100000 },
          { key:'SecHigh',label:'📗 ระดับมัธยมศึกษาตอนปลาย / สายอาชีพ (ปวช.)',   startAge:16, years:3, feeDefault:100000 },
          { key:'Uni',    label:'🎓 ระดับปริญญาตรี (หลักสูตร 4 ปี)',           startAge:19, years:4, feeDefault:400000 },
          { key:'Master', label:'📜 ระดับปริญญาโท',            startAge:23, years:2, feeDefault:300000 },
        ];

        const EDU_LOCAL_TABLE = [
          { title: "🏫 อนุบาล (อ.1-3 / อายุ 4-6 ปี)", rows: [ { type: "รัฐบาล", min: 0, max: 5000, note: "มีเงินอุดหนุน" }, { type: "เอกชน (ทั่วไป)", min: 30000, max: 80000, note: "" }, { type: "เอกชน (นานาชาติ)", min: 200000, max: 500000, note: "" } ] },
          { title: "📚 ประถมศึกษา (ป.1-6 / อายุ 7-12 ปี)", rows: [ { type: "รัฐบาล", min: 0, max: 8000, note: "มีเงินอุดหนุน" }, { type: "เอกชน (ทั่วไป)", min: 40000, max: 100000, note: "" }, { type: "เอกชน (นานาชาติ)", min: 250000, max: 600000, note: "" } ] },
          { title: "📘 มัธยมต้น (ม.1-3 / อายุ 13-15 ปี)", rows: [ { type: "รัฐบาล", min: 0, max: 10000, note: "มีเงินอุดหนุน" }, { type: "เอกชน (ทั่วไป)", min: 50000, max: 120000, note: "" }, { type: "เอกชน (นานาชาติ)", min: 300000, max: 700000, note: "" } ] },
          { title: "📗 มัธยมปลาย (ม.4-6 / อายุ 16-18 ปี)", rows: [ { type: "รัฐบาล", min: 0, max: 10000, note: "มีเงินอุดหนุน" }, { type: "เอกชน (ทั่วไป)", min: 50000, max: 130000, note: "" }, { type: "เอกชน (นานาชาติ)", min: 350000, max: 800000, note: "" } ] },
          { title: "🎓 ปริญญาตรี (ปี 1-4 / อายุ 19-22 ปี)", rows: [ { type: "รัฐบาล (ภาคปกติ)", min: 15000, max: 50000, note: "ต่อเทอม" }, { type: "รัฐบาล (ภาคพิเศษ)", min: 30000, max: 80000, note: "ต่อเทอม" }, { type: "เอกชน (ทั่วไป)", min: 60000, max: 150000, note: "ต่อเทอม" }, { type: "แพทย์ / วิศวะ / สถาปัตย์", min: 100000, max: 300000, note: "ต่อเทอม" }, { type: "มหาวิทยาลัยนานาชาติ", min: 300000, max: 800000, note: "ต่อปี" } ] },
          { title: "📜 ปริญญาโท (ปี 1-2 / อายุ 23-24 ปี)", rows: [ { type: "รัฐบาล (ภาคปกติ)", min: 25000, max: 60000, note: "ต่อเทอม" }, { type: "รัฐบาล (ภาคพิเศษ)", min: 50000, max: 100000, note: "ต่อเทอม" }, { type: "เอกชน / MBA", min: 100000, max: 350000, note: "ต่อเทอม" }, { type: "ต่างประเทศ", min: 500000, max: 1500000, note: "ต่อปี" } ] }
        ];

        const EDU_ABROAD_TABLE = [
          { country: "🇬🇧 อังกฤษ", y68: 2267966, y78: 3694278, y88: 6017589 },
          { country: "🇺🇸 อเมริกา", y68: 2731797, y78: 4449810, y88: 7248271 },
          { country: "🇦🇺 ออสเตรเลีย", y68: 1690347, y78: 2753398, y88: 4484994 },
          { country: "🇳🇿 นิวซีแลนด์", y68: 1116930, y78: 1819362, y88: 2963548 },
          { country: "🇹🇭 ไทย (รัฐบาล)", y68: 270000, y78: 439802, y88: 716390 },
          { country: "🇹🇭 ไทย (เอกชน)", y68: 340000, y78: 553825, y88: 902122 },
        ];

        // ============================================================
        // SHARED HELPERS
        // ============================================================
        const fmt = (n) => Math.round(n).toLocaleString('en-US');

        const NumInput = ({ label, value, onChange, unit='บาท', className='', note, max }) => {
          const [localVal, setLocalVal] = useState(() => value === 0 ? '' : String(value));
          const isFocused = useRef(false);

          useEffect(() => {
            if (!isFocused.current) {
              setLocalVal(value === 0 ? '' : value.toLocaleString('en-US'));
            }
          }, [value]);

          return (
            <div className={`flex flex-col gap-1 ${className}`}>
              {label && (
                <div className="flex justify-between items-center">
                  <label className="text-xs font-semibold text-gray-600">{label}</label>
                  {max && <span className="text-[10px] text-gray-400">ไม่เกิน {fmt(max)} บ.</span>}
                </div>
              )}
              <div className="flex items-center gap-1">
                <input
                  type="text"
                  inputMode="numeric"
                  value={localVal}
                  onFocus={() => { isFocused.current = true; setLocalVal(value === 0 ? '' : String(value)); }}
                  onChange={e => {
                    const raw = e.target.value.replace(/[^0-9]/g, '');
                    setLocalVal(raw);
                    onChange(raw ? parseInt(raw, 10) : 0);
                  }}
                  onBlur={() => {
                    isFocused.current = false;
                    setLocalVal(value === 0 ? '' : value.toLocaleString('en-US'));
                  }}
                  className="w-full p-2 bg-white border border-gray-300 rounded-lg text-sm font-medium focus:border-emerald-500 focus:outline-none focus:ring-1 focus:ring-emerald-200"
                />
                {unit && <span className="text-xs text-gray-500 whitespace-nowrap">{unit}</span>}
              </div>
              {note && <p className="text-[10px] text-gray-400">{note}</p>}
            </div>
          );
        };

        const GapRow = ({ label, goal, ready }) => {
          const gap = Math.max(0, goal - ready);
          const pct = goal > 0 ? Math.min(100, Math.round((ready/goal)*100)) : 0;
          return (
            <div className="mb-3">
              <div className="flex justify-between text-sm mb-1">
                <span className="font-medium text-gray-700">{label}</span>
                <span className={gap>0?'text-red-600 font-bold':'text-green-600 font-bold'}>{gap>0?`ขาด ${fmt(gap)}`:'✓ เพียงพอ'} บ.</span>
              </div>
              <div className="w-full bg-gray-200 rounded-full h-2">
                <div className={`h-2 rounded-full transition-all ${gap>0?'bg-red-400':'bg-green-400'}`} style={{width:`${pct}%`}}></div>
              </div>
              <div className="flex justify-between text-xs text-gray-400 mt-1">
                <span>เป้าหมาย {fmt(goal)} บ.</span><span>เตรียมแล้ว {fmt(ready)} บ.</span>
              </div>
            </div>
          );
        };

        const SectionCard = ({ icon, title, color, children, priority, badge }) => (
          <div className={`bg-white rounded-2xl border-2 ${color} overflow-hidden shadow-sm`}>
            <div className={`p-4 flex justify-between items-center ${color.replace('border-','bg-').replace('-400','-50').replace('-500','-50')}`}>
              <div className="flex items-center gap-2">
                {priority && <span className={`w-6 h-6 rounded-full flex items-center justify-center text-xs font-bold text-white ${color.replace('border-','bg-').replace('-400','-500')}`}>{priority}</span>}
                <span className="font-bold text-gray-800 flex items-center gap-2">{icon} {title}</span>
              </div>
              {badge && <span className="text-xs bg-white border rounded-full px-2 py-0.5 text-gray-500">{badge}</span>}
            </div>
            <div className="p-5">{children}</div>
          </div>
        );

        // ============================================================
        // TAX PLANNING PAGE
        // ============================================================
        const TaxPlanningPage = ({ taxState, setTaxState }) => {
          const {
            incomes, spouseDeduct, childCount, childOld, pregnancyCost,
            parentDeduct, disabledCare, lifeInsurance, healthInsurance,
            annuityInsurance, parentInsurance, socialSecurity, rmf, pvd,
            govPension, natSaving, ssf, thaiesp, mortgage, politicalDonate,
            eduDonate, generalDonate, shopDee
          } = taxState;

          const s = (key) => (val) => setTaxState(prev => ({ ...prev, [key]: val }));
          const setIncomes = (updater) => setTaxState(prev => ({ ...prev, incomes: typeof updater === 'function' ? updater(prev.incomes) : updater }));

          const updateIncome = (id, field, val) => {
            setIncomes(prev => prev.map(r => r.id === id ? { ...r, [field]: val } : r));
          };

          const calc = useMemo(() => {
            let used12Cap = 0;
            const incomeRows = incomes.map(r => {
              const typeInfo = INCOME_TYPE_OPTIONS.find(t => t.value === r.type) || INCOME_TYPE_OPTIONS[0];
              let exp = r.amount * typeInfo.rate;
              if (r.type === '40_1' || r.type === '40_2') {
                const available = Math.max(0, 100000 - used12Cap);
                exp = Math.min(exp, available);
                used12Cap += exp;
              } else {
                if (typeInfo.max) exp = Math.min(exp, typeInfo.max);
              }
              return { ...r, typeInfo, expense: exp };
            });

            const totalGrossIncome = incomeRows.reduce((sum, r) => sum + r.amount, 0);
            const totalExpenseDeduct = incomeRows.reduce((sum, r) => sum + r.expense, 0);
            const totalWithholding = incomes.reduce((sum, r) => sum + r.withholding, 0);

            const spouseAllowance = spouseDeduct ? 60000 : 0;
            const childAllowance  = (childCount * 60000) + (childOld * 30000);
            const pregnancyAmt    = Math.min(pregnancyCost, 60000);
            const parentAllowance = Math.min(parentDeduct, 4) * 30000;
            const disabledAllowance = disabledCare * 60000;
            const totalPersonal = 60000 + spouseAllowance + childAllowance + pregnancyAmt + parentAllowance + disabledAllowance;

            const healthIns = Math.min(healthInsurance, 25000);
            const lifeIns   = Math.min(lifeInsurance, Math.max(0, 100000 - healthIns));
            const parentIns = Math.min(parentInsurance, 15000);
            const socialSecurityAmt = Math.min(socialSecurity, 9000);

            const cap30 = totalGrossIncome * 0.30;
            const cap15 = totalGrossIncome * 0.15;
            const annuityIns   = Math.min(annuityInsurance, Math.min(cap15, 200000));
            const ssfAmt       = Math.min(ssf, Math.min(cap30, 200000));
            const rmfAmt       = Math.min(rmf, Math.min(cap30, 500000));
            const pvdAmt       = Math.min(pvd, Math.min(cap15, 500000));
            const govPensionAmt= Math.min(govPension, Math.min(cap30, 500000));
            const natSavingAmt = Math.min(natSaving, 13200);
            const combinedRaw  = annuityIns + ssfAmt + rmfAmt + pvdAmt + govPensionAmt + natSavingAmt;
            const combinedInvestCap = Math.min(combinedRaw, 500000);
            const thaiespAmt = Math.min(thaiesp, Math.min(cap30, 300000));

            const mortgageAmt   = Math.min(mortgage, 100000);
            const politicalAmt  = Math.min(politicalDonate, 10000);
            const shopDeeAmt    = Math.min(shopDee, 50000);

            const beforeDonation = totalGrossIncome - totalExpenseDeduct - totalPersonal
              - lifeIns - healthIns - parentIns - socialSecurityAmt
              - combinedInvestCap - thaiespAmt - mortgageAmt - shopDeeAmt;

            const baseDonate    = Math.max(0, beforeDonation);
            const eduDonateAmt  = Math.min(eduDonate * 2, baseDonate * 0.10);
            const generalDonateAmt = Math.min(generalDonate, Math.max(0, baseDonate * 0.10 - eduDonateAmt));

            const taxableIncome = Math.max(0, beforeDonation - eduDonateAmt - generalDonateAmt - politicalAmt);
            const totalDeductions = totalGrossIncome - taxableIncome;
            const taxBeforeCredit = calcTax(taxableIncome);
            const taxDue   = Math.max(0, taxBeforeCredit - totalWithholding);
            const refund   = Math.max(0, totalWithholding - taxBeforeCredit);
            const effectiveRate = totalGrossIncome > 0 ? (taxBeforeCredit / totalGrossIncome) * 100 : 0;
            const marginalRate  = getMarginalRate(taxableIncome) * 100;

            return {
              totalGrossIncome, totalExpenseDeduct, totalWithholding, totalPersonal,
              lifeIns, healthIns, annuityIns, parentIns, socialSecurityAmt, pregnancyAmt,
              combinedInvestCap, thaiespAmt, natSavingAmt,
              mortgageAmt, shopDeeAmt, politicalAmt, eduDonateAmt, generalDonateAmt,
              taxableIncome, totalDeductions, taxBeforeCredit, taxDue, refund,
              effectiveRate, marginalRate,
              ssfAmt, rmfAmt, pvdAmt, govPensionAmt, combinedRaw,
              incomeRows,
            };
          }, [incomes, spouseDeduct, childCount, childOld, pregnancyCost, parentDeduct, disabledCare,
              lifeInsurance, healthInsurance, annuityInsurance, parentInsurance, socialSecurity,
              ssf, rmf, pvd, govPension, natSaving, thaiesp,
              mortgage, politicalDonate, eduDonate, generalDonate, shopDee]);

          const suggestions = useMemo(() => {
            const sList = [];
            const income = calc.totalGrossIncome;
            const marginal = getMarginalRate(calc.taxableIncome);
            if (marginal === 0) return sList;
            const roomLife = Math.max(0, 100000 - calc.lifeIns - calc.healthIns);
            if (roomLife > 0) sList.push({ icon:'🛡️', label:'ประกันชีวิต', detail:`ซื้อเพิ่มได้อีก ${fmt(roomLife)} บ. (ลดภาษี ~${fmt(roomLife * marginal)} บ.)`, color:'blue' });
            const roomHealth = Math.max(0, 25000 - calc.healthIns);
            if (roomHealth > 0) sList.push({ icon:'❤️', label:'ประกันสุขภาพ', detail:`ซื้อเพิ่มได้อีก ${fmt(roomHealth)} บ. (ลดภาษี ~${fmt(roomHealth * marginal)} บ.)`, color:'red' });
            const maxAnnuity = Math.min(income * 0.15, 200000);
            if (annuityInsurance < maxAnnuity && calc.combinedRaw < 500000) {
              const r = Math.min(maxAnnuity - calc.annuityIns, 500000 - calc.combinedRaw);
              if (r > 0) sList.push({ icon:'🌅', label:'ประกันบำนาญ', detail:`ซื้อเพิ่มได้อีก ${fmt(r)} บ. (ลดภาษี ~${fmt(r * marginal)} บ.)`, color:'orange' });
            }
            const maxSSF = Math.min(income * 0.30, 200000);
            if (ssf < maxSSF && calc.combinedRaw < 500000) {
              const r = Math.min(maxSSF - calc.ssfAmt, 500000 - calc.combinedRaw);
              if (r > 0) sList.push({ icon:'📈', label:'SSF', detail:`ลงทุนเพิ่มได้อีก ${fmt(r)} บ. (ลดภาษี ~${fmt(r * marginal)} บ.)`, color:'purple' });
            }
            const maxRMF = Math.min(income * 0.30, 500000);
            if (rmf < maxRMF && calc.combinedRaw < 500000) {
              const r = Math.min(maxRMF - calc.rmfAmt, 500000 - calc.combinedRaw);
              if (r > 0) sList.push({ icon:'📊', label:'RMF', detail:`ลงทุนเพิ่มได้อีก ${fmt(r)} บ. (ลดภาษี ~${fmt(r * marginal)} บ.)`, color:'indigo' });
            }
            const maxESG = Math.min(income * 0.30, 300000);
            if (thaiesp < maxESG) {
              const r = maxESG - calc.thaiespAmt;
              if (r > 0) sList.push({ icon:'🌿', label:'Thai ESG', detail:`ลงทุนเพิ่มได้อีก ${fmt(r)} บ. (ลดภาษี ~${fmt(r * marginal)} บ.)`, color:'green' });
            }
            if (shopDee < 50000) sList.push({ icon:'🧾', label:'Easy E-Receipt', detail:`ใช้จ่ายเพิ่มได้อีก ${fmt(50000 - shopDee)} บ. (ลดภาษี ~${fmt((50000-shopDee) * marginal)} บ.)`, color:'teal' });
            return sList;
          }, [calc, lifeInsurance, healthInsurance, annuityInsurance, ssf, rmf, thaiesp, shopDee]);

          return (
            <div className="space-y-6">
              <div className="bg-gradient-to-r from-emerald-800 to-teal-600 text-white rounded-3xl p-6 shadow-lg">
                <div className="flex items-center gap-3 mb-2">
                  <Receipt size={32} />
                  <div>
                    <h2 className="text-2xl font-bold">วางแผนภาษี</h2>
                    <p className="text-emerald-100 text-sm">คำนวณภาษีและหาช่องว่างในการลดหย่อน</p>
                  </div>
                </div>
              </div>
              <div className="grid grid-cols-1 xl:grid-cols-3 gap-6">
                <div className="xl:col-span-2 space-y-5">
                  <div className="bg-white rounded-2xl p-5 border-2 border-green-200 shadow-sm">
                    <div className="flex items-center gap-2 mb-4">
                      <Banknote size={20} className="text-green-600"/>
                      <h3 className="font-bold text-green-800">💰 รายได้ (สูงสุด 2 แหล่ง)</h3>
                    </div>
                    <div className="space-y-4">
                      {incomes.map((row, idx) => {
                        const rowCalc = calc.incomeRows.find(r => r.id === row.id);
                        const typeInfo = rowCalc.typeInfo;
                        const expDeduct = rowCalc.expense;
                        const is40_12 = row.type === '40_1' || row.type === '40_2';
                        return (
                          <div key={row.id} className="p-4 rounded-xl border border-green-100 bg-green-50/40 space-y-3">
                            <span className="text-xs font-bold bg-green-200 text-green-800 px-2 py-0.5 rounded-full">แหล่งที่ {idx+1}</span>
                            <div className="grid grid-cols-1 sm:grid-cols-2 gap-3 mt-2">
                              <div>
                                <label className="text-xs font-semibold text-gray-600 block mb-1">ประเภทรายได้</label>
                                <select value={row.type} onChange={e => updateIncome(row.id, 'type', e.target.value)}
                                  className="w-full p-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:border-green-500 bg-white">
                                  {INCOME_TYPE_OPTIONS.map(opt => <option key={opt.value} value={opt.value}>{opt.label}</option>)}
                                </select>
                              </div>
                              <NumInput label="รายได้ต่อปี" value={row.amount} onChange={v => updateIncome(row.id, 'amount', v)} />
                              <NumInput label="ภาษีหัก ณ ที่จ่าย (หนังสือ 50 ทวิ)" value={row.withholding} onChange={v => updateIncome(row.id, 'withholding', v)} note="ตรวจสอบจากหนังสือรับรองการหักภาษีฯ" />
                              <div className="bg-white rounded-xl p-3 border border-green-200 flex flex-col justify-center">
                                <p className="text-xs text-gray-500">ค่าใช้จ่ายหักได้ {(typeInfo.rate*100).toFixed(0)}% {is40_12 ? '(40(1),(2) รวมกันสูงสุด 1 แสน)' : typeInfo.max ? `(สูงสุด ${fmt(typeInfo.max)} บ.)` : ''}</p>
                                <p className="text-base font-bold text-green-700">{fmt(expDeduct)} บาท</p>
                              </div>
                            </div>
                          </div>
                        );
                      })}
                      <div className="flex gap-3 pt-2 border-t border-green-100">
                        <div className="flex-1 bg-green-50 rounded-xl p-3 border border-green-100">
                          <p className="text-xs text-gray-500">รายได้รวมทุกแหล่ง</p>
                          <p className="text-lg font-bold text-green-700">{fmt(calc.totalGrossIncome)} บาท</p>
                        </div>
                        <div className="flex-1 bg-blue-50 rounded-xl p-3 border border-blue-100">
                          <p className="text-xs text-gray-500">ภาษีหัก ณ ที่จ่าย รวม</p>
                          <p className="text-lg font-bold text-blue-700">{fmt(calc.totalWithholding)} บาท</p>
                        </div>
                      </div>
                    </div>
                  </div>
                  <div className="bg-white rounded-2xl p-5 border-2 border-blue-200 shadow-sm">
                    <div className="flex items-center gap-2 mb-4">
                      <User size={20} className="text-blue-600"/>
                      <h3 className="font-bold text-blue-800">👤 ค่าลดหย่อนส่วนบุคคลและครอบครัว</h3>
                    </div>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                      <div className="bg-blue-50 rounded-xl p-3 border border-blue-100">
                        <p className="text-xs text-gray-500">ค่าลดหย่อนส่วนตัว (คงที่)</p>
                        <p className="text-lg font-bold text-blue-700">60,000 บาท</p>
                      </div>
                      <div>
                        <label className="text-xs font-semibold text-gray-600 block mb-1">คู่สมรสไม่มีรายได้</label>
                        <div className="flex gap-3">
                          {[{v:true,l:'มี (60,000 บ.)'},{v:false,l:'ไม่มี'}].map(o=>(
                            <button key={String(o.v)} onClick={()=>s('spouseDeduct')(o.v)}
                              className={`flex-1 py-2 rounded-lg border-2 text-sm font-semibold transition-all ${spouseDeduct===o.v?'border-blue-500 bg-blue-50 text-blue-700':'border-gray-200 text-gray-500'}`}>
                              {o.l}
                            </button>
                          ))}
                        </div>
                      </div>
                      <div>
                        <label className="text-xs font-semibold text-gray-600 block mb-2">บุตร เกิดหลัง 2561 — 60,000 บ./คน</label>
                        <div className="flex items-center gap-3">
                          <button onClick={()=>s('childCount')(Math.max(0,childCount-1))} className="w-8 h-8 rounded-full bg-gray-100 font-bold text-gray-600 hover:bg-gray-200">-</button>
                          <span className="w-8 text-center font-bold text-blue-700 text-lg">{childCount}</span>
                          <button onClick={()=>s('childCount')(childCount+1)} className="w-8 h-8 rounded-full bg-blue-100 font-bold text-blue-600 hover:bg-blue-200">+</button>
                          <span className="text-sm text-gray-500">= {fmt(childCount*60000)} บ.</span>
                        </div>
                      </div>
                      <div>
                        <label className="text-xs font-semibold text-gray-600 block mb-2">บุตร เกิดก่อน 2561 — 30,000 บ./คน</label>
                        <div className="flex items-center gap-3">
                          <button onClick={()=>s('childOld')(Math.max(0,childOld-1))} className="w-8 h-8 rounded-full bg-gray-100 font-bold text-gray-600 hover:bg-gray-200">-</button>
                          <span className="w-8 text-center font-bold text-blue-700 text-lg">{childOld}</span>
                          <button onClick={()=>s('childOld')(childOld+1)} className="w-8 h-8 rounded-full bg-blue-100 font-bold text-blue-600 hover:bg-blue-200">+</button>
                          <span className="text-sm text-gray-500">= {fmt(childOld*30000)} บ.</span>
                        </div>
                      </div>
                      <NumInput label="ค่าฝากครรภ์/ค่าคลอดบุตร" value={pregnancyCost} onChange={s('pregnancyCost')} max={60000} note="ตามที่จ่ายจริง สูงสุด 60,000 บ./ครรภ์" />
                      <div>
                        <label className="text-xs font-semibold text-gray-600 block mb-2">บิดามารดา — 30,000 บ./คน (ไม่เกิน 4 คน)</label>
                        <div className="flex items-center gap-3">
                          <button onClick={()=>s('parentDeduct')(Math.max(0,parentDeduct-1))} className="w-8 h-8 rounded-full bg-gray-100 font-bold text-gray-600 hover:bg-gray-200">-</button>
                          <span className="w-8 text-center font-bold text-blue-700 text-lg">{Math.min(parentDeduct,4)}</span>
                          <button onClick={()=>s('parentDeduct')(Math.min(4,parentDeduct+1))} className="w-8 h-8 rounded-full bg-blue-100 font-bold text-blue-600 hover:bg-blue-200">+</button>
                          <span className="text-sm text-gray-500">= {fmt(Math.min(parentDeduct,4)*30000)} บ.</span>
                        </div>
                      </div>
                      <div>
                        <label className="text-xs font-semibold text-gray-600 block mb-2">ผู้พิการ/ทุพพลภาพที่อุปการะ — 60,000 บ./คน</label>
                        <div className="flex items-center gap-3">
                          <button onClick={()=>s('disabledCare')(Math.max(0,disabledCare-1))} className="w-8 h-8 rounded-full bg-gray-100 font-bold text-gray-600 hover:bg-gray-200">-</button>
                          <span className="w-8 text-center font-bold text-blue-700 text-lg">{disabledCare}</span>
                          <button onClick={()=>s('disabledCare')(disabledCare+1)} className="w-8 h-8 rounded-full bg-blue-100 font-bold text-blue-600 hover:bg-blue-200">+</button>
                          <span className="text-sm text-gray-500">= {fmt(disabledCare*60000)} บ.</span>
                        </div>
                      </div>
                      <div className="bg-blue-50 rounded-xl p-3 border border-blue-100 flex flex-col justify-center">
                        <p className="text-xs text-gray-500">ลดหย่อนส่วนบุคคล/ครอบครัว รวม</p>
                        <p className="text-lg font-bold text-blue-700">{fmt(calc.totalPersonal)} บาท</p>
                      </div>
                    </div>
                  </div>
                  <div className="bg-white rounded-2xl p-5 border-2 border-rose-200 shadow-sm">
                    <div className="flex items-center gap-2 mb-4">
                      <Shield size={20} className="text-rose-600"/>
                      <h3 className="font-bold text-rose-800">🛡️ ค่าลดหย่อนประกันภัย</h3>
                    </div>
                    <div className="bg-rose-50 rounded-xl p-3 border border-rose-100 mb-4 text-xs text-rose-700">
                      ⚠️ เบี้ยประกันชีวิต + เบี้ยประกันสุขภาพตนเอง รวมกันไม่เกิน <b>100,000 บาท</b> (ประกันสุขภาพเดี่ยว ไม่เกิน 25,000 บาท)
                    </div>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                      <NumInput label="ประกันชีวิตทั่วไป" value={lifeInsurance} onChange={s('lifeInsurance')} note="รวมกับสุขภาพไม่เกิน 100,000 บ." />
                      <NumInput label="ประกันสุขภาพตนเอง" value={healthInsurance} onChange={s('healthInsurance')} max={25000} />
                      <div className="sm:col-span-2">
                        <div className={`rounded-xl p-3 border text-sm flex justify-between ${(calc.lifeIns + calc.healthIns) >= 100000 ? 'bg-red-50 border-red-200 text-red-700' : 'bg-green-50 border-green-200 text-green-700'}`}>
                          <span>ประกันชีวิต + สุขภาพที่ใช้ได้จริง</span>
                          <span className="font-bold">{fmt(calc.lifeIns + calc.healthIns)} / 100,000 บาท</span>
                        </div>
                      </div>
                      <NumInput label="ประกันชีวิตแบบบำนาญ" value={annuityInsurance} onChange={s('annuityInsurance')} max={Math.min(calc.totalGrossIncome * 0.15, 200000)} note={`15% ของรายได้ สูงสุด ${fmt(Math.min(calc.totalGrossIncome*0.15,200000))} บ. (นับรวมกลุ่มเกษียณ)`} />
                      <NumInput label="ประกันสุขภาพบิดามารดา" value={parentInsurance} onChange={s('parentInsurance')} max={15000} />
                      <NumInput label="เงินสมทบประกันสังคม (ม.33/39/40)" value={socialSecurity} onChange={s('socialSecurity')} max={9000} note="สูงสุด 9,000 บ./ปี" />
                    </div>
                  </div>
                  <div className="bg-white rounded-2xl p-5 border-2 border-purple-200 shadow-sm">
                    <div className="flex items-center gap-2 mb-3">
                      <TrendingUp size={20} className="text-purple-600"/>
                      <h3 className="font-bold text-purple-800">📈 กลุ่มออมเพื่อเกษียณ</h3>
                    </div>
                    <div className="bg-purple-50 rounded-xl p-3 border border-purple-100 mb-4 text-xs text-purple-700 space-y-1">
                      <p>⚠️ <b>SSF + RMF + PVD + กบข. + กอช. + ประกันบำนาญ</b> รวมกันต้องไม่เกิน <b>500,000 บาท</b> และไม่เกิน <b>30%</b> ของรายได้</p>
                      <p>🌿 <b>Thai ESG</b> มีเพดานแยกต่างหาก ≤ 30% ของรายได้ ไม่เกิน 300,000 บาท</p>
                    </div>
                    <div className="mb-4">
                      <div className="flex justify-between text-xs text-gray-500 mb-1">
                        <span>ใช้ไปแล้ว: {fmt(calc.combinedInvestCap)} บ.</span>
                        <span>เพดาน: 500,000 บ.</span>
                      </div>
                      <div className="w-full bg-gray-200 rounded-full h-2">
                        <div className={`h-2 rounded-full transition-all ${calc.combinedRaw >= 500000 ? 'bg-red-500' : 'bg-purple-500'}`} style={{width:`${Math.min(100, (calc.combinedRaw/500000)*100)}%`}}></div>
                      </div>
                    </div>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                      <NumInput label="กองทุนรวม SSF" value={ssf} onChange={s('ssf')} max={Math.min(calc.totalGrossIncome*0.30, 200000)} note={`30% รายได้ สูงสุด ${fmt(Math.min(calc.totalGrossIncome*0.30,200000))} บ.`} />
                      <NumInput label="กองทุนรวม RMF" value={rmf} onChange={s('rmf')} max={Math.min(calc.totalGrossIncome*0.30, 500000)} note={`30% รายได้ สูงสุด ${fmt(Math.min(calc.totalGrossIncome*0.30,500000))} บ.`} />
                      <NumInput label="กองทุนสำรองเลี้ยงชีพ (PVD)" value={pvd} onChange={s('pvd')} max={Math.min(calc.totalGrossIncome*0.15, 500000)} note={`15% รายได้ สูงสุด ${fmt(Math.min(calc.totalGrossIncome*0.15,500000))} บ.`} />
                      <NumInput label="กบข. / กองทุนครูเอกชน / บำเหน็จบำนาญ" value={govPension} onChange={s('govPension')} max={Math.min(calc.totalGrossIncome*0.30, 500000)} />
                      <NumInput label="กองทุนการออมแห่งชาติ (กอช.)" value={natSaving} onChange={s('natSaving')} max={13200} note="สูงสุด 13,200 บ./ปี" />
                      <div className="border-t border-purple-100 pt-2 sm:col-span-2">
                        <p className="text-xs font-bold text-purple-700 mb-2">🌿 Thai ESG / Thai ESGX (เพดานแยก)</p>
                        <NumInput label="กองทุน Thai ESG" value={thaiesp} onChange={s('thaiesp')} max={Math.min(calc.totalGrossIncome*0.30, 300000)} note={`30% รายได้ สูงสุด 300,000 บ. — ไม่นับรวมเพดาน 500,000 บ.`} />
                      </div>
                    </div>
                  </div>
                  <div className="bg-white rounded-2xl p-5 border-2 border-amber-200 shadow-sm">
                    <div className="flex items-center gap-2 mb-4">
                      <Home size={20} className="text-amber-600"/>
                      <h3 className="font-bold text-amber-800">🏠 ค่าลดหย่อนอื่นๆ</h3>
                    </div>
                    <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                      <NumInput label="ดอกเบี้ยกู้ซื้อ/สร้าง/เช่าซื้อบ้าน" value={mortgage} onChange={s('mortgage')} max={100000} note="ตามที่จ่ายจริง ไม่เกิน 100,000 บ." />
                      <NumInput label="Easy E-Receipt / Shop Dee Mee Kuen" value={shopDee} onChange={s('shopDee')} max={50000} note="สินค้า/บริการที่ออก e-Tax Invoice ไม่เกิน 50,000 บ." />
                      <div className="sm:col-span-2 bg-amber-50 rounded-xl p-3 border border-amber-100">
                        <p className="text-xs font-bold text-amber-700 mb-2">💛 เงินบริจาค (รวมกันไม่เกิน 10% ของเงินได้หลังหักค่าใช้จ่ายและลดหย่อน)</p>
                        <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                          <NumInput label="บริจาคเพื่อการศึกษา/โรงพยาบาลรัฐ/กีฬา" value={eduDonate} onChange={s('eduDonate')} note="ลดหย่อนได้ 2 เท่า รวมอยู่ใน cap 10%" />
                          <NumInput label="บริจาคทั่วไป" value={generalDonate} onChange={s('generalDonate')} note="ไม่เกิน 10% ของเงินได้หลังหักลดหย่อน (รวมกับ 2 เท่า)" />
                          <NumInput label="บริจาคพรรคการเมือง" value={politicalDonate} onChange={s('politicalDonate')} max={10000} note="ไม่เกิน 10,000 บ. (แยกออกจากบริจาคทั่วไป)" />
                          <div className="bg-white rounded-xl p-3 border border-amber-200 flex flex-col justify-center">
                            <p className="text-xs text-gray-500">บริจาคที่ใช้ได้จริง (2เท่า+ทั่วไป)</p>
                            <p className="font-bold text-amber-700">{fmt(calc.eduDonateAmt + calc.generalDonateAmt)} บาท</p>
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </div>
                <div className="xl:col-span-1">
                  <div className="sticky top-6 space-y-4">
                    <div className="bg-gray-900 rounded-3xl p-6 shadow-xl text-white border border-gray-800">
                      <h3 className="text-lg font-bold mb-4 flex items-center gap-2 border-b border-gray-700 pb-3">
                        <Receipt className="text-emerald-400"/> สรุปภาษี
                      </h3>
                      <div className="flex items-center justify-center mb-4">
                        <div className="relative w-28 h-28">
                          <svg viewBox="0 0 36 36" className="w-28 h-28 -rotate-90">
                            <circle cx="18" cy="18" r="15.9" fill="none" stroke="#1f2937" strokeWidth="3.8"/>
                            <circle cx="18" cy="18" r="15.9" fill="none" stroke="#10b981" strokeWidth="3.8" strokeDasharray={`${calc.totalGrossIncome>0?Math.min(100,(calc.totalDeductions/calc.totalGrossIncome)*100):0} 100`} strokeLinecap="round"/>
                            <circle cx="18" cy="18" r="15.9" fill="none" stroke="#D31145" strokeWidth="3.8" strokeDasharray={`${calc.totalGrossIncome>0?Math.min(100,(calc.taxBeforeCredit/calc.totalGrossIncome)*100):0} 100`} strokeDashoffset={`${-(calc.totalGrossIncome>0?Math.min(100,(calc.totalDeductions/calc.totalGrossIncome)*100):0)}`} strokeLinecap="round"/>
                          </svg>
                          <div className="absolute inset-0 flex flex-col items-center justify-center">
                            <p className="text-xs text-gray-400">ภาษี</p>
                            <p className="text-sm font-bold text-white">{calc.effectiveRate.toFixed(1)}%</p>
                          </div>
                        </div>
                      </div>
                      <div className="flex gap-3 justify-center mb-4 text-xs">
                        <div className="flex items-center gap-1"><span className="w-2.5 h-2.5 rounded-full bg-emerald-500 inline-block"></span><span className="text-gray-300">ลดหย่อน</span></div>
                        <div className="flex items-center gap-1"><span className="w-2.5 h-2.5 rounded-full bg-rose-600 inline-block"></span><span className="text-gray-300">ภาษี</span></div>
                      </div>
                      <div className="space-y-2 text-sm">
                        {[
                          {label:'รายได้รวม', val:fmt(calc.totalGrossIncome)+' บ.', color:'text-white', bold:true},
                          {label:'หัก: ค่าใช้จ่าย', val:'-'+fmt(calc.totalExpenseDeduct)+' บ.', color:'text-emerald-400'},
                          {label:'หัก: ส่วนบุคคล/ครอบครัว', val:'-'+fmt(calc.totalPersonal)+' บ.', color:'text-emerald-400'},
                          {label:'หัก: ประกันชีวิต+สุขภาพ', val:'-'+fmt(calc.lifeIns+calc.healthIns)+' บ.', color:'text-emerald-400', show: (calc.lifeIns+calc.healthIns)>0},
                          {label:'หัก: ประกันบำนาญ', val:'-'+fmt(calc.annuityIns)+' บ.', color:'text-emerald-400', show: calc.annuityIns>0},
                          {label:'หัก: ประกันสุขภาพพ่อแม่', val:'-'+fmt(calc.parentIns)+' บ.', color:'text-emerald-400', show: calc.parentIns>0},
                          {label:'หัก: ประกันสังคม', val:'-'+fmt(calc.socialSecurityAmt)+' บ.', color:'text-emerald-400', show: calc.socialSecurityAmt>0},
                          {label:'หัก: กลุ่มเกษียณ (cap 500k)', val:'-'+fmt(calc.combinedInvestCap)+' บ.', color:'text-emerald-400', show: calc.combinedInvestCap>0},
                          {label:'หัก: Thai ESG', val:'-'+fmt(calc.thaiespAmt)+' บ.', color:'text-emerald-400', show: calc.thaiespAmt>0},
                          {label:'หัก: ดอกเบี้ยบ้าน', val:'-'+fmt(calc.mortgageAmt)+' บ.', color:'text-emerald-400', show: calc.mortgageAmt>0},
                          {label:'หัก: Easy E-Receipt', val:'-'+fmt(calc.shopDeeAmt)+' บ.', color:'text-emerald-400', show: calc.shopDeeAmt>0},
                          {label:'หัก: บริจาค', val:'-'+fmt(calc.eduDonateAmt+calc.generalDonateAmt+calc.politicalAmt)+' บ.', color:'text-emerald-400', show: (calc.eduDonateAmt+calc.generalDonateAmt+calc.politicalAmt)>0},
                        ].filter(r => r.show !== false).map((r,i) => (
                          <div key={i} className={`flex justify-between ${r.color||'text-gray-300'} ${r.bold?'font-bold':''}`}>
                            <span>{r.label}</span><span>{r.val}</span>
                          </div>
                        ))}
                        <div className="border-t border-gray-700 pt-2 flex justify-between font-bold">
                          <span>เงินได้สุทธิ</span><span className="text-yellow-300">{fmt(calc.taxableIncome)} บ.</span>
                        </div>
                        <div className="flex justify-between text-xs text-gray-400">
                          <span>อัตราภาษีสูงสุด (Marginal)</span><span>{calc.marginalRate.toFixed(0)}%</span>
                        </div>
                      </div>
                      <div className="mt-4 pt-4 border-t border-gray-700">
                        <div className="bg-rose-900/40 rounded-2xl p-4 border border-rose-800/50 mb-3">
                          <p className="text-xs text-rose-300 mb-1">ภาษีที่ต้องชำระ (ตามกฎหมาย)</p>
                          <p className="text-3xl font-bold text-white">{fmt(calc.taxBeforeCredit)} <span className="text-base font-medium text-gray-300">บาท</span></p>
                        </div>
                        <div className="grid grid-cols-2 gap-3">
                          <div className="bg-blue-900/30 rounded-xl p-3 border border-blue-800/40">
                            <p className="text-xs text-blue-300">ภาษีหัก ณ ที่จ่าย</p>
                            <p className="text-base font-bold text-blue-300">{fmt(calc.totalWithholding)} บ.</p>
                          </div>
                          <div className={`rounded-xl p-3 border ${calc.taxDue>0?'bg-red-900/30 border-red-800/40':'bg-green-900/30 border-green-800/40'}`}>
                            <p className={`text-xs ${calc.taxDue>0?'text-red-300':'text-green-300'}`}>{calc.taxDue>0?'ต้องชำระเพิ่ม':'✓ ได้รับคืน'}</p>
                            <p className={`text-base font-bold ${calc.taxDue>0?'text-red-300':'text-green-300'}`}>{fmt(calc.taxDue>0?calc.taxDue:calc.refund)} บ.</p>
                          </div>
                        </div>
                      </div>
                    </div>
                    {suggestions.length > 0 && (
                      <div className="bg-white rounded-2xl p-5 border-2 border-yellow-200 shadow-sm">
                        <div className="flex items-center gap-2 mb-3">
                          <Lightbulb size={18} className="text-yellow-500"/>
                          <h3 className="font-bold text-yellow-800">💡 ช่องว่างที่ยังลดหย่อนได้</h3>
                        </div>
                        <div className="space-y-2">
                          {suggestions.map((sg,i)=>(
                            <div key={i} className="flex items-start gap-3 p-3 rounded-xl bg-gray-50 border border-gray-100">
                              <span className="text-lg">{sg.icon}</span>
                              <div>
                                <p className="text-sm font-bold text-gray-800">{sg.label}</p>
                                <p className="text-xs text-gray-500">{sg.detail}</p>
                              </div>
                            </div>
                          ))}
                        </div>
                      </div>
                    )}
                    <div className="bg-white rounded-2xl p-5 border-2 border-gray-100 shadow-sm">
                      <h3 className="font-bold text-gray-700 mb-3 text-sm">📊 อัตราภาษีเงินได้บุคคลธรรมดา</h3>
                      <div className="space-y-1">
                        {TAX_BRACKETS.map((b,i)=>{
                          const active = calc.taxableIncome > b.min-1 && b.rate>0;
                          return (
                            <div key={i} className={`flex justify-between text-xs px-2 py-1.5 rounded-lg ${active?'bg-emerald-50 text-emerald-800 font-semibold':'text-gray-400'}`}>
                              <span>{b.min===0?'0 – 150,000':b.max===Infinity?'มากกว่า 5,000,000':`${fmt(b.min)} – ${fmt(b.max)}`}</span>
                              <span className="font-bold">{(b.rate*100).toFixed(0)}%</span>
                            </div>
                          );
                        })}
                      </div>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          );
        };

        // ============================================================
        // FACT FINDING PAGE
        // ============================================================
        const FactFindingPage = ({ ffState, setFF }) => {
          const { planEdu, planRetire, planHealth, planLife, clientName, gender, age, retireAge, lifeExpect, inflation, annualIncome, hasChild, childAge, eduEnabled, eduFees, educationReady, educationInsurance, monthlyExpenseRetire, retirementReady, retirementInsurance, currentHospital, roomRateNeeded, roomRateHave, annualHealthBudget, annualHealthHave, ciCoverageNeeded, ciCoverageHave, accidentNeeded, accidentHave, monthlyFamilyExpense, protectionYears, debtHome, debtCar, debtOther, liquidAssets, existingInsurance, budget } = ffState;
          
          const [showEduTable, setShowEduTable] = useState(false);

          const s = (key) => (val) => setFF(prev => ({ ...prev, [key]: val }));
          const sRaw = (key) => (val) => setFF(prev => ({ ...prev, [key]: val }));

          const calcEduCost = (levelKey, childAge, inflation, fee) => {
            const lv = EDU_LEVELS.find(l => l.key === levelKey);
            if (!lv) return { cost: 0, remainingYears: 0, yearsUntilStart: 0, passed: true };
            const passed = childAge > lv.startAge + lv.years - 1;
            const remainingYears = passed ? 0 : (childAge >= lv.startAge ? (lv.startAge + lv.years - childAge) : lv.years);
            const yearsUntilStart = Math.max(0, lv.startAge - childAge);
            let totalCost = 0;
            for (let i = 0; i < remainingYears; i++) {
              let inflYears = yearsUntilStart + i;
              totalCost += fee * Math.pow(1 + inflation / 100, inflYears);
            }
            return { cost: totalCost, remainingYears, yearsUntilStart, passed };
          };

          const levelDetails = {};
          EDU_LEVELS.forEach(lv => {
            const fee = eduFees[lv.key] || lv.feeDefault;
            levelDetails[lv.key] = calcEduCost(lv.key, childAge, inflation, fee);
          });
          
          const eduGoal = Object.keys(levelDetails).reduce((sum, k) => sum + (eduEnabled[k] ? levelDetails[k].cost : 0), 0);
          const eduGap = Math.max(0, eduGoal - educationReady - educationInsurance);
          const retireYears = Math.max(0, lifeExpect - retireAge);
          const yearlyExpenseRetire = monthlyExpenseRetire * 12;
          const retireGoal = yearlyExpenseRetire * retireYears;
          const retireAccumYears = Math.max(0, retireAge - age);
          const retireGap = Math.max(0, retireGoal - retirementReady - retirementInsurance);
          const roomGap = Math.max(0, roomRateNeeded - roomRateHave);
          const ciGap = Math.max(0, ciCoverageNeeded - ciCoverageHave);
          const familyNeeded = monthlyFamilyExpense * 12 * protectionYears;
          const totalDebt = debtHome + debtCar + debtOther;
          const lifeGoal = familyNeeded + totalDebt;
          const lifeReady = liquidAssets + existingInsurance;
          const lifeGap = Math.max(0, lifeGoal - lifeReady);
          
          const healthTotalGoal = annualHealthBudget + ciCoverageNeeded + accidentNeeded;
          const healthTotalHave = annualHealthHave + ciCoverageHave + accidentHave;
          const healthTotalGap = Math.max(0, healthTotalGoal - healthTotalHave);

          const priorities = [
            ...(planEdu && hasChild ? [{ no:1, name:'ทุนการศึกษาลูก', goal: Math.round(eduGoal), ready: educationReady+educationInsurance, gap: Math.round(eduGap) }] : []),
            ...(planHealth ? [{ no:2, name:'ความคุ้มครองสุขภาพ', sub:'(วงเงินรักษา + โรคร้ายแรง + อุบัติเหตุ)', goal: healthTotalGoal, ready: healthTotalHave, gap: healthTotalGap }] : []),
            ...(planRetire ? [{ no:3, name:'เงินออมเพื่อเกษียณ', goal: Math.round(retireGoal), ready: retirementReady+retirementInsurance, gap: Math.round(retireGap) }] : []),
            ...(planLife ? [{ no:4, name:'คุ้มครองรายได้ครอบครัว', goal: Math.round(lifeGoal), ready: lifeReady, gap: Math.round(lifeGap) }] : []),
          ];

          return (
            <div className="space-y-6">
              <div className="bg-gradient-to-r from-blue-800 to-blue-600 text-white rounded-3xl p-6 shadow-lg">
                <div className="flex items-center gap-3 mb-4">
                  <ClipboardList size={32} />
                  <div>
                    <h2 className="text-2xl font-bold">Fact Finding</h2>
                    <p className="text-blue-100 text-sm">วิเคราะห์ความต้องการทางการเงินและช่องว่างความคุ้มครอง</p>
                  </div>
                </div>
                <div className="grid grid-cols-2 md:grid-cols-4 gap-3">
                  <div>
                    <label className="text-blue-200 text-xs block mb-1">ชื่อลูกค้า</label>
                    <input value={clientName} onChange={e=>sRaw('clientName')(e.target.value)} placeholder="กรอกชื่อ..." className="w-full p-2 rounded-lg bg-white/20 text-white placeholder-blue-200 text-sm border border-blue-300/50 focus:outline-none focus:border-white" />
                  </div>
                  <div>
                    <label className="text-blue-200 text-xs block mb-1">เพศ</label>
                    <select value={gender} onChange={e=>sRaw('gender')(e.target.value)} className="w-full p-2 rounded-lg bg-white/20 text-white text-sm border border-blue-300/50 focus:outline-none">
                      <option value="M" className="text-gray-800">ชาย</option><option value="F" className="text-gray-800">หญิง</option>
                    </select>
                  </div>
                  <div>
                    <label className="text-blue-200 text-xs block mb-1">อายุ</label>
                    <input type="number" value={age} onChange={e=>sRaw('age')(Number(e.target.value))} className="w-full p-2 rounded-lg bg-white/20 text-white text-sm border border-blue-300/50 focus:outline-none" />
                  </div>
                  <div>
                    <label className="text-blue-200 text-xs block mb-1">อัตราเงินเฟ้อ (%)</label>
                    <input type="number" value={inflation} onChange={e=>sRaw('inflation')(Number(e.target.value))} className="w-full p-2 rounded-lg bg-white/20 text-white text-sm border border-blue-300/50 focus:outline-none" />
                  </div>
                </div>
              </div>

              <div className="bg-white rounded-2xl p-5 shadow-sm border-2 border-green-200">
                <div className="flex items-center gap-2 mb-3"><Banknote size={20} className="text-green-600"/><p className="text-sm font-bold text-green-800">💰 รายได้และฐานะทางการเงิน</p></div>
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                  <NumInput label="รายได้ต่อปี (บาท)" value={annualIncome} onChange={s('annualIncome')} />
                  <div className="bg-green-50 rounded-xl p-3 border border-green-100 flex flex-col justify-center">
                    <p className="text-xs text-gray-500 mb-1">รายได้ต่อเดือน (โดยประมาณ)</p>
                    <p className="text-lg font-bold text-green-700">{fmt(Math.round(annualIncome/12))} บาท</p>
                  </div>
                </div>
              </div>

              <div className="bg-white rounded-2xl p-5 shadow-sm border-2 border-blue-100">
                <p className="text-sm font-bold text-blue-800 mb-3">📋 เลือกหัวข้อที่ต้องการวางแผน</p>
                <div className="flex flex-wrap gap-3">
                  {[{key:'planEdu',val:planEdu,label:'🎓 ทุนการศึกษา',color:'blue'},{key:'planHealth',val:planHealth,label:'❤️ สุขภาพ',color:'red'},{key:'planRetire',val:planRetire,label:'🌅 เกษียณ',color:'orange'},{key:'planLife',val:planLife,label:'🛡️ คุ้มครองรายได้',color:'purple'}].map(p=>(
                    <button key={p.key} onClick={()=>sRaw(p.key)(!p.val)} className={`flex items-center gap-2 px-4 py-2 rounded-xl border-2 font-semibold text-sm transition-all ${p.val?`border-${p.color}-400 bg-${p.color}-50 text-${p.color}-700`:'border-gray-200 bg-white text-gray-400'}`}>
                      {p.val?<CheckCircle2 size={16}/>:<Circle size={16}/>} {p.label}
                    </button>
                  ))}
                </div>
              </div>

              <div className="grid grid-cols-1 xl:grid-cols-2 gap-6">
                {planEdu && (
                  <SectionCard icon={<GraduationCap size={20} className="text-blue-500"/>} title="Education Plan" color="border-blue-400" priority="1" badge="ทุนการศึกษาลูก">
                    <div className="flex items-center gap-2 mb-4">
                      <input type="checkbox" checked={hasChild} onChange={e=>sRaw('hasChild')(e.target.checked)} id="hasChild" className="w-4 h-4 accent-blue-500" />
                      <label htmlFor="hasChild" className="text-sm font-medium text-gray-700">มีบุตรที่ต้องวางแผนการศึกษา</label>
                    </div>
                    {hasChild && (
                      <div className="space-y-4">
                        <NumInput label="อายุลูกปัจจุบัน" value={childAge} onChange={s('childAge')} unit="ปี" className="mb-2" />
                        <div className="mb-2">
                          <button onClick={() => setShowEduTable(!showEduTable)} className="flex items-center gap-1.5 text-sm font-bold text-blue-600 hover:text-blue-800 transition-colors bg-blue-50 px-3 py-2 rounded-lg border border-blue-100 w-full sm:w-auto justify-center">
                            <Table size={16} /> {showEduTable ? 'ซ่อนตารางอ้างอิงค่าเล่าเรียน' : 'ดูตารางอ้างอิงค่าเล่าเรียนโดยประมาณ'}
                          </button>
                          {showEduTable && (
                            <div className="mt-3 p-4 bg-gray-50 border border-gray-200 rounded-xl space-y-4 animate-fade-in-up">
                              <p className="text-xs font-bold text-gray-700 border-b pb-2">🇹🇭 ค่าเล่าเรียนในไทย (บาท)</p>
                              <div className="overflow-x-auto no-scrollbar max-h-64 border rounded-lg bg-white">
                                <table className="w-full text-xs text-left whitespace-nowrap">
                                  <thead className="bg-blue-50 text-blue-800 sticky top-0 shadow-sm">
                                    <tr>
                                      <th className="p-2 font-bold border-b border-blue-100">ระดับชั้น</th>
                                      <th className="p-2 font-bold border-b border-blue-100">ประเภท</th>
                                      <th className="p-2 font-bold text-right border-b border-blue-100">ต่ำสุด</th>
                                      <th className="p-2 font-bold text-right border-b border-blue-100">สูงสุด</th>
                                      <th className="p-2 font-bold border-b border-blue-100 text-gray-500">หมายเหตุ</th>
                                    </tr>
                                  </thead>
                                  <tbody className="divide-y divide-gray-100">
                                    {EDU_LOCAL_TABLE.map((group, i) => (
                                      <React.Fragment key={i}>
                                        {group.rows.map((row, j) => (
                                          <tr key={`${i}-${j}`} className="hover:bg-gray-50">
                                            {j === 0 && <td rowSpan={group.rows.length} className="p-2 font-bold text-gray-700 bg-gray-50/50 align-top border-r border-gray-100 whitespace-normal min-w-[120px]">{group.title}</td>}
                                            <td className="p-2 text-gray-600">{row.type}</td>
                                            <td className="p-2 text-right text-gray-600">{row.min === 0 ? 'ฟรี' : fmt(row.min)}</td>
                                            <td className="p-2 text-right text-gray-600">{fmt(row.max)}</td>
                                            <td className="p-2 text-gray-400">{row.note}</td>
                                          </tr>
                                        ))}
                                      </React.Fragment>
                                    ))}
                                  </tbody>
                                </table>
                              </div>

                              <p className="text-xs font-bold text-gray-700 border-b pb-2 pt-2">🌏 เรียนต่างประเทศ ป.ตรี (รวมค่าครองชีพ / คิดเงินเฟ้อ 5%)</p>
                              <div className="overflow-x-auto no-scrollbar border rounded-lg bg-white">
                                <table className="w-full text-xs text-center whitespace-nowrap">
                                  <thead className="bg-orange-50 text-orange-800">
                                    <tr>
                                      <th className="p-2 font-bold border-b border-orange-100 text-left">ประเทศ</th>
                                      <th className="p-2 font-bold border-b border-orange-100">ปี 2568</th>
                                      <th className="p-2 font-bold border-b border-orange-100">ปี 2578</th>
                                      <th className="p-2 font-bold border-b border-orange-100">ปี 2588</th>
                                    </tr>
                                  </thead>
                                  <tbody className="divide-y divide-gray-100">
                                    {EDU_ABROAD_TABLE.map((row, i) => (
                                      <tr key={i} className="hover:bg-gray-50">
                                        <td className="p-2 font-bold text-gray-700 text-left">{row.country}</td>
                                        <td className="p-2 text-orange-600">{fmt(row.y68)}</td>
                                        <td className="p-2 text-orange-600">{fmt(row.y78)}</td>
                                        <td className="p-2 text-orange-600">{fmt(row.y88)}</td>
                                      </tr>
                                    ))}
                                  </tbody>
                                </table>
                              </div>
                            </div>
                          )}
                        </div>

                        {EDU_LEVELS.map(lv => {
                          const details = levelDetails[lv.key];
                          const isActive = eduEnabled[lv.key];
                          return (
                            <div key={lv.key} className={`rounded-xl border-2 transition-all overflow-hidden ${details.passed?'opacity-40 border-gray-100 bg-gray-50':isActive?'border-blue-300 bg-blue-50':'border-gray-100 bg-gray-50 opacity-70'}`}>
                              <div className="flex items-center justify-between px-4 py-3 cursor-pointer"
                                onClick={() => !details.passed && setFF(prev => ({ ...prev, eduEnabled: { ...prev.eduEnabled, [lv.key]: !prev.eduEnabled[lv.key] } }))}>
                                <div className="flex items-center gap-2">
                                  {isActive && !details.passed ? <CheckCircle2 size={18} className="text-blue-500"/> : <Circle size={18} className="text-gray-300"/>}
                                  <span className="font-semibold text-sm text-gray-800">{lv.label}</span>
                                  <span className="text-xs text-gray-500 bg-white px-2 py-0.5 rounded-full border">
                                    {details.passed ? 'ผ่านแล้ว' : details.yearsUntilStart === 0 ? `กำลังเรียน (เหลือ ${details.remainingYears} ปี)` : `เหลือ ${details.remainingYears} ปี · เริ่มอีก ${details.yearsUntilStart} ปี`}
                                  </span>
                                </div>
                                <span className="text-xs font-bold text-blue-700">{isActive && !details.passed ? fmt(details.cost) + ' บ.' : '-'}</span>
                              </div>
                              {isActive && !details.passed && (
                                <div className="px-4 pb-3 pt-1 border-t border-blue-200/50">
                                  <NumInput label={`ค่าใช้จ่ายต่อปี (บ.)`} value={eduFees[lv.key] || lv.feeDefault}
                                    onChange={val => setFF(prev => ({ ...prev, eduFees: { ...prev.eduFees, [lv.key]: val } }))} />
                                </div>
                              )}
                            </div>
                          );
                        })}
                        <div className="bg-blue-100 rounded-xl p-4 border border-blue-200 text-sm">
                          <div className="flex justify-between font-bold text-blue-800"><span>ทุนการศึกษารวม</span><span>{fmt(eduGoal)} บ.</span></div>
                        </div>
                        <div className="grid grid-cols-2 gap-3">
                          <NumInput label="เงินฝาก/ทรัพย์สินเตรียมแล้ว" value={educationReady} onChange={s('educationReady')} />
                          <NumInput label="เงินคืนจากกรมธรรม์" value={educationInsurance} onChange={s('educationInsurance')} />
                        </div>
                        <GapRow label="ทุนการศึกษา" goal={Math.round(eduGoal)} ready={educationReady+educationInsurance} />
                      </div>
                    )}
                  </SectionCard>
                )}

                {planRetire && (
                  <SectionCard icon={<Sunset size={20} className="text-orange-500"/>} title="Retirement Plan" color="border-orange-400" priority="3" badge="เงินออมเกษียณ">
                    <div className="grid grid-cols-3 gap-3 mb-4">
                      <NumInput label="อายุเกษียณ" value={retireAge} onChange={s('retireAge')} unit="ปี" />
                      <NumInput label="อายุขัย" value={lifeExpect} onChange={s('lifeExpect')} unit="ปี" />
                      <div className="flex flex-col gap-1"><label className="text-xs font-semibold text-gray-600">สะสม / ใช้</label><div className="p-2 bg-orange-50 border border-orange-200 rounded-lg text-sm font-bold text-orange-700 text-center">{retireAccumYears} / {retireYears} ปี</div></div>
                    </div>
                    <NumInput label="ค่าใช้จ่ายหลังเกษียณต่อเดือน" value={monthlyExpenseRetire} onChange={s('monthlyExpenseRetire')} className="mb-4" />
                    <div className="bg-orange-50 rounded-xl p-4 border border-orange-100 space-y-2 text-sm mb-4">
                      <div className="flex justify-between font-bold text-orange-700"><span>กองทุนเกษียณที่ต้องการ</span><span>{fmt(retireGoal)} บ.</span></div>
                    </div>
                    <div className="grid grid-cols-2 gap-3 mb-4">
                      <NumInput label="เงินฝาก/ทรัพย์สินเตรียมแล้ว" value={retirementReady} onChange={s('retirementReady')} />
                      <NumInput label="เงินคืนจากกรมธรรม์" value={retirementInsurance} onChange={s('retirementInsurance')} />
                    </div>
                    <GapRow label="กองทุนเกษียณ" goal={Math.round(retireGoal)} ready={retirementReady+retirementInsurance} />
                  </SectionCard>
                )}

                {planHealth && (
                  <SectionCard icon={<HeartPulse size={20} className="text-red-500"/>} title="Health Protection" color="border-red-400" priority="2" badge="คุ้มครองสุขภาพ">
                    <div className="space-y-5">
                      <div>
                        <label className="text-xs font-semibold text-gray-600 block mb-1">โรงพยาบาลที่ใช้ประจำ</label>
                        <input value={currentHospital} onChange={e=>sRaw('currentHospital')(e.target.value)} placeholder="เช่น รพ.พระรามเก้า, รพ.รัฐ..." className="w-full p-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:border-red-400" />
                      </div>
                      <div className="bg-red-50 rounded-xl p-4 border border-red-100 space-y-3">
                        <p className="text-xs font-bold text-red-700">🏥 ค่าห้องพักโรงพยาบาล</p>
                        <div className="grid grid-cols-2 gap-3">
                          <NumInput label="ค่าห้องที่ต้องการ (บ./คืน)" value={roomRateNeeded} onChange={s('roomRateNeeded')} unit="" />
                          <NumInput label="สวัสดิการที่มีอยู่ (บ./คืน)" value={roomRateHave} onChange={s('roomRateHave')} unit="" />
                        </div>
                        <GapRow label="ค่าห้องพัก" goal={roomRateNeeded} ready={roomRateHave} />
                      </div>
                      <div className="bg-blue-50 rounded-xl p-4 border border-blue-100 space-y-3">
                        <p className="text-xs font-bold text-blue-700">🛡️ วงเงินค่ารักษาพยาบาลต่อปี</p>
                        <div className="grid grid-cols-2 gap-3">
                          <NumInput label="งบที่เหมาะสม (บ./ปี)" value={annualHealthBudget} onChange={s('annualHealthBudget')} unit="" />
                          <NumInput label="สวัสดิการที่เตรียมแล้ว" value={annualHealthHave} onChange={s('annualHealthHave')} unit="" />
                        </div>
                        <GapRow label="วงเงินค่ารักษา" goal={annualHealthBudget} ready={annualHealthHave} />
                      </div>
                      <div className="bg-rose-50 rounded-xl p-4 border border-rose-200 space-y-3">
                        <p className="text-xs font-bold text-rose-700">🎗️ โรคร้ายแรง</p>
                        <div className="grid grid-cols-2 gap-3">
                          <NumInput label="วงเงินที่เหมาะสม (บาท)" value={ciCoverageNeeded} onChange={s('ciCoverageNeeded')} unit="" />
                          <NumInput label="สวัสดิการที่มีอยู่ (บาท)" value={ciCoverageHave} onChange={s('ciCoverageHave')} unit="" />
                        </div>
                        <GapRow label="ความคุ้มครองโรคร้ายแรง" goal={ciCoverageNeeded} ready={ciCoverageHave} />
                      </div>
                      <div className="bg-amber-50 rounded-xl p-4 border border-amber-200 space-y-3">
                        <p className="text-xs font-bold text-amber-700">⚡ อุบัติเหตุ</p>
                        <div className="grid grid-cols-2 gap-3">
                          <NumInput label="วงเงินที่ต้องการ (บาท)" value={accidentNeeded} onChange={s('accidentNeeded')} unit="" />
                          <NumInput label="สวัสดิการที่มีอยู่ (บาท)" value={accidentHave} onChange={s('accidentHave')} unit="" />
                        </div>
                        <GapRow label="ความคุ้มครองอุบัติเหตุ" goal={accidentNeeded} ready={accidentHave} />
                      </div>
                    </div>
                  </SectionCard>
                )}

                {planLife && (
                  <SectionCard icon={<Shield size={20} className="text-purple-500"/>} title="Human Value Protection" color="border-purple-400" priority="4" badge="คุ้มครองรายได้">
                    <div className="space-y-4">
                      <div className="grid grid-cols-2 gap-3">
                        <NumInput label="ค่าใช้จ่ายครอบครัวต่อเดือน" value={monthlyFamilyExpense} onChange={s('monthlyFamilyExpense')} />
                        <NumInput label="ต้องการคุ้มครองกี่ปี" value={protectionYears} onChange={s('protectionYears')} unit="ปี" />
                      </div>
                      <div className="grid grid-cols-3 gap-3">
                        <NumInput label="ค่าผ่อนบ้านคงค้าง" value={debtHome} onChange={s('debtHome')} unit="" />
                        <NumInput label="ค่าผ่อนรถคงค้าง" value={debtCar} onChange={s('debtCar')} unit="" />
                        <NumInput label="หนี้อื่นๆ" value={debtOther} onChange={s('debtOther')} unit="" />
                      </div>
                      <div className="bg-purple-50 rounded-xl p-4 border border-purple-100 text-sm mb-2">
                        <div className="flex justify-between font-bold text-purple-700"><span>จำนวนเงินที่ต้องการ</span><span>{fmt(lifeGoal)} บ.</span></div>
                      </div>
                      <div className="grid grid-cols-2 gap-3">
                        <NumInput label="เงินฝาก/ทรัพย์สินสภาพคล่อง" value={liquidAssets} onChange={s('liquidAssets')} />
                        <NumInput label="ทุนประกันที่มีอยู่" value={existingInsurance} onChange={s('existingInsurance')} />
                      </div>
                      <GapRow label="คุ้มครองรายได้ครอบครัว" goal={Math.round(lifeGoal)} ready={lifeReady} />
                    </div>
                  </SectionCard>
                )}
              </div>

              <div className="bg-gray-900 rounded-3xl p-6 md:p-8 text-white shadow-xl border border-gray-800">
                <h2 className="text-xl font-bold mb-6 flex items-center gap-2 border-b border-gray-700 pb-4">
                  <Target className="text-blue-400"/> บทสรุป {clientName && <span className="text-blue-300">({clientName})</span>}
                </h2>
                <div className="overflow-x-auto mb-6">
                  <table className="w-full text-sm whitespace-nowrap">
                    <thead><tr className="text-gray-400 border-b border-gray-700"><th className="text-left p-3">ลำดับ</th><th className="text-left p-3">รายการ</th><th className="text-right p-3">เป้าหมาย</th><th className="text-right p-3">เตรียมแล้ว</th><th className="text-right p-3">ขาดอยู่</th></tr></thead>
                    <tbody>
                      {priorities.map((row, i) => (
                        <tr key={i} className="border-b border-gray-800 hover:bg-gray-800/50">
                          <td className="p-3 font-bold text-teal-400">{row.no}</td>
                          <td className="p-3 font-medium">{row.name}{row.sub && <span className="text-xs text-gray-400 block">{row.sub}</span>}</td>
                          <td className="p-3 text-right text-gray-300">{fmt(row.goal)}</td>
                          <td className="p-3 text-right text-green-400">{fmt(row.ready)}</td>
                          <td className={`p-3 text-right font-bold ${row.gap>0?'text-red-400':'text-green-400'}`}>{row.gap>0?fmt(row.gap):'✓'}</td>
                        </tr>
                      ))}
                    </tbody>
                  </table>
                </div>
                <div className="bg-gray-800 rounded-2xl p-4 border border-gray-700">
                  <p className="text-gray-400 text-xs mb-2">งบประมาณจัดสรรต่อเดือน</p>
                  <div className="flex items-end gap-2">
                    <input type="text" value={budget===0?'':budget.toLocaleString('en-US')} onChange={e=>{const v=e.target.value.replace(/\D/g,'');sRaw('budget')(v?parseInt(v):0);}} className="w-full p-2 bg-gray-700 rounded-lg text-white font-bold text-lg text-center focus:outline-none focus:ring-1 focus:ring-blue-400" />
                    <span className="text-gray-400 text-sm mb-2">บาท/เดือน</span>
                  </div>
                </div>
              </div>
            </div>
          );
        };

        // ============================================================
        // MAIN APP
        // ============================================================
        function App() {
          const [mainTab, setMainTab] = useState('FACTFINDING');

          const [taxState, setTaxState] = useState({
            incomes: [
              { id: 1, type: '40_1', amount: 600000, withholding: 0 },
              { id: 2, type: '40_8', amount: 0, withholding: 0 },
            ],
            spouseDeduct: false, childCount: 0, childOld: 0, pregnancyCost: 0,
            parentDeduct: 0, disabledCare: 0, lifeInsurance: 0, healthInsurance: 0,
            annuityInsurance: 0, parentInsurance: 0, socialSecurity: 0, rmf: 0, pvd: 0,
            govPension: 0, natSaving: 0, ssf: 0, thaiesp: 0, mortgage: 0, politicalDonate: 0,
            eduDonate: 0, generalDonate: 0, shopDee: 0,
          });

          const [ffState, setFF] = useState({
            planEdu: true, planRetire: true, planHealth: true, planLife: true,
            clientName: '', gender: 'M', age: 35, retireAge: 60, lifeExpect: 80, inflation: 3, annualIncome: 600000,
            hasChild: true, childAge: 3, eduEnabled: { K:true, Pri:true, SecLow:true, SecHigh:true, Uni:true, Master:false },
            eduFees: { K:100000, Pri:80000, SecLow:100000, SecHigh:100000, Uni:400000, Master:300000 },
            educationReady: 0, educationInsurance: 0, monthlyExpenseRetire: 30000, retirementReady: 0, retirementInsurance: 0,
            currentHospital: '', roomRateNeeded: 3000, roomRateHave: 0, annualHealthBudget: 5000000, annualHealthHave: 0,
            ciCoverageNeeded: 3000000, ciCoverageHave: 0, accidentNeeded: 1000000, accidentHave: 0,
            monthlyFamilyExpense: 0, protectionYears: 0, debtHome: 0, debtCar: 0, debtOther: 0,
            liquidAssets: 0, existingInsurance: 0, budget: 0,
          });

          const tabs = [
            { id:'FACTFINDING', label:'Fact Finding', icon:<ClipboardList size={16}/>, color:'blue' },
            { id:'TAX', label:'วางแผนภาษี', icon:<Receipt size={16}/>, color:'emerald' }
          ];

          return (
            <div className="min-h-screen bg-gray-50 p-2 md:p-6 font-sans text-gray-800">
              <div className="max-w-7xl mx-auto">
                <div className="bg-gradient-to-r from-[#1e3a8a] via-blue-700 to-[#D31145] text-white rounded-3xl p-6 md:p-8 shadow-lg mb-6">
                  <div className="flex items-center gap-4 mb-2">
                    <ShieldCheck size={36}/>
                    <div>
                      <h1 className="text-2xl md:text-3xl font-bold tracking-wide">Financial Planning Suite</h1>
                      <p className="text-white/70 text-sm md:text-base">Fact Finding | วางแผนภาษี</p>
                    </div>
                  </div>
                </div>

                <div className="flex bg-gray-200 p-1 rounded-2xl mb-6 gap-1 flex-wrap">
                  {tabs.map(t=>(
                    <button key={t.id} onClick={()=>setMainTab(t.id)}
                      className={`flex items-center gap-2 px-4 py-3 rounded-xl font-bold text-sm transition-all flex-1 justify-center min-w-[120px] ${mainTab===t.id?`bg-white shadow-sm text-${t.id==='FACTFINDING'?'blue':'emerald'}-600`:'text-gray-500 hover:text-gray-700'}`}>
                      {t.icon} {t.label}
                    </button>
                  ))}
                </div>

                {mainTab==='FACTFINDING' && <FactFindingPage ffState={ffState} setFF={setFF} />}
                {mainTab==='TAX' && <TaxPlanningPage taxState={taxState} setTaxState={setTaxState} />}
              </div>

              <style dangerouslySetInnerHTML={{__html:`
                @keyframes fadeInUp{from{opacity:0;transform:translateY(15px)}to{opacity:1;transform:translateY(0)}}
                .animate-fade-in-up{animation:fadeInUp 0.4s cubic-bezier(0.16,1,0.3,1) forwards}
                .no-scrollbar::-webkit-scrollbar{display:none}
                .no-scrollbar{-ms-overflow-style:none;scrollbar-width:none}
              `}}/>
            </div>
          );
        }

        const root = createRoot(document.getElementById('root'));
        root.render(<App/>);
    </script>
</body>
</html>
