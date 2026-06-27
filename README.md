# kzsc-khata-app
Shop management and ledger app
import { useState, useMemo, useCallback, useRef, useEffect } from "react";

// ── Design Tokens ─────────────────────────────────────────────────────
const C = {
  soil:"#1B1207", soilMid:"#2D1F0E", soilLight:"#4A3520",
  leaf:"#2E7D32", leafMid:"#388E3C", leafLight:"#4CAF50", leafPale:"#E8F5E9", leafFaint:"#F1F8F1",
  gold:"#F59E0B", goldLight:"#FFFBEB", goldBorder:"#FDE68A",
  sky:"#0369A1", skyLight:"#E0F2FE",
  danger:"#B91C1C", dangerMid:"#DC2626", dangerLight:"#FEF2F2", dangerBorder:"#FECACA",
  amber:"#D97706",
  gray50:"#F9FAFB", gray100:"#F3F4F6", gray200:"#E5E7EB", gray300:"#D1D5DB",
  gray400:"#9CA3AF", gray500:"#6B7280", gray700:"#374151", gray900:"#111827",
  white:"#FFFFFF",
};

// ── localStorage helpers ──────────────────────────────────────────────
function lsGet(key, fallback) {
  try {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : fallback;
  } catch { return fallback; }
}
function lsSet(key, value) {
  try { localStorage.setItem(key, JSON.stringify(value)); } catch {}
}

// ── Seed Data (used only on very first launch) ────────────────────────
const SEED_PRODUCTS = [
  {id:"p1",name:"Dursban 40EC",category:"Pesticide",unit:"Litre",stock:45,purchasePrice:850,sellingPrice:1100,lowStockAlert:10},
  {id:"p2",name:"Urea Fertilizer",category:"Fertilizer",unit:"Bag (50kg)",stock:8,purchasePrice:2200,sellingPrice:2600,lowStockAlert:15},
  {id:"p3",name:"Cotton Seeds BT",category:"Seeds",unit:"Packet",stock:120,purchasePrice:1400,sellingPrice:1800,lowStockAlert:20},
  {id:"p4",name:"Confidor 200SL",category:"Pesticide",unit:"100ml Bottle",stock:6,purchasePrice:420,sellingPrice:580,lowStockAlert:10},
  {id:"p5",name:"DAP Fertilizer",category:"Fertilizer",unit:"Bag (50kg)",stock:22,purchasePrice:3200,sellingPrice:3700,lowStockAlert:10},
  {id:"p6",name:"Wheat Seeds",category:"Seeds",unit:"Kg",stock:200,purchasePrice:90,sellingPrice:130,lowStockAlert:50},
];
const SEED_CUSTOMERS = [
  {id:"c1",name:"Muhammad Akram",phone:"0300-1234567",village:"Chak 45",notes:"Reliable farmer"},
  {id:"c2",name:"Allah Ditta",phone:"0333-9876543",village:"Chak 12",notes:""},
  {id:"c3",name:"Ghulam Hussain",phone:"0312-5551234",village:"Chak 78",notes:"Cotton grower"},
];
const SEED_TX = [
  {id:"t1",type:"sale",customerId:"c1",customerName:"Muhammad Akram",date:"2025-06-20",items:[{productId:"p1",name:"Dursban 40EC",qty:2,price:1100}],total:2200,paymentMode:"credit",paid:0,note:""},
  {id:"t2",type:"sale",customerId:"c2",customerName:"Allah Ditta",date:"2025-06-21",items:[{productId:"p3",name:"Cotton Seeds BT",qty:5,price:1800}],total:9000,paymentMode:"cash",paid:9000,note:""},
  {id:"t3",type:"payment",customerId:"c1",customerName:"Muhammad Akram",date:"2025-06-22",items:[],total:1000,paymentMode:"cash",paid:1000,note:"Partial payment"},
  {id:"t4",type:"sale",customerId:"c3",customerName:"Ghulam Hussain",date:"2025-06-23",items:[{productId:"p5",name:"DAP Fertilizer",qty:3,price:3700}],total:11100,paymentMode:"credit",paid:5000,note:""},
  {id:"t5",type:"sale",customerId:"c1",customerName:"Muhammad Akram",date:"2025-06-24",items:[{productId:"p2",name:"Urea Fertilizer",qty:2,price:2600}],total:5200,paymentMode:"credit",paid:0,note:""},
];

// ── Utilities ─────────────────────────────────────────────────────────
const fmt = (n) => `Rs. ${Number(n||0).toLocaleString("en-PK")}`;
const todayStr = () => new Date().toISOString().split("T")[0];
const uid = () => Math.random().toString(36).slice(2, 9);
const fmtDate = (d) => { if(!d) return "—"; const dt = new Date(d+"T00:00:00"); return dt.toLocaleDateString("en-PK",{day:"numeric",month:"short",year:"numeric"}); };
const monthKey = (d) => d ? d.slice(0,7) : "";

// ── Math ──────────────────────────────────────────────────────────────
function calcBalances(customers, transactions) {
  const b = {};
  customers.forEach(c => { b[c.id] = 0; });
  transactions.forEach(t => {
    if (!Object.prototype.hasOwnProperty.call(b, t.customerId)) return;
    if (t.type === "sale") b[t.customerId] += (t.total - t.paid);
    else if (t.type === "payment") b[t.customerId] -= t.paid;
  });
  Object.keys(b).forEach(k => { if (b[k] < 0) b[k] = 0; });
  return b;
}

function buildLedger(txList) {
  const sorted = [...txList].sort((a,b) => {
    const dc = a.date.localeCompare(b.date);
    return dc !== 0 ? dc : a.id.localeCompare(b.id);
  });
  let running = 0;
  return sorted.map(t => {
    if (t.type === "sale") {
      const credit = t.total - t.paid;
      running += credit;
      return { ...t, credit, runningBalance: running };
    } else {
      running = Math.max(0, running - t.paid);
      return { ...t, credit: 0, runningBalance: running };
    }
  }).reverse();
}

function getReceiptData(tx, allTxForCustomer) {
  const sorted = [...allTxForCustomer].sort((a,b) => {
    const dc = a.date.localeCompare(b.date);
    return dc !== 0 ? dc : a.id.localeCompare(b.id);
  });
  let running = 0;
  let prevBalance = 0;
  for (const t of sorted) {
    if (t.id === tx.id) prevBalance = running;
    if (t.type === "sale") running += (t.total - t.paid);
    else running = Math.max(0, running - t.paid);
  }
  let afterThisTx = prevBalance;
  if (tx.type === "sale") afterThisTx = prevBalance + (tx.total - tx.paid);
  else afterThisTx = Math.max(0, prevBalance - tx.paid);
  return { prevBalance, afterThisTx };
}

// ── Primitives ────────────────────────────────────────────────────────
function Badge({ children, color="leaf", size="sm" }) {
  const map = {
    leaf:{bg:C.leafPale,text:C.leaf,border:C.leafLight},
    gold:{bg:C.goldLight,text:C.amber,border:C.goldBorder},
    sky:{bg:C.skyLight,text:C.sky,border:"#BAE6FD"},
    danger:{bg:C.dangerLight,text:C.danger,border:C.dangerBorder},
  };
  const s = map[color]||map.leaf;
  return <span style={{background:s.bg,color:s.text,border:`1px solid ${s.border}`,padding:size==="lg"?"4px 14px":"2px 9px",borderRadius:20,fontSize:size==="lg"?13:11,fontWeight:700,letterSpacing:.3,whiteSpace:"nowrap"}}>{children}</span>;
}

function Card({ children, style={}, onClick }) {
  return <div onClick={onClick} style={{background:C.white,borderRadius:16,padding:"20px 24px",boxShadow:"0 1px 4px rgba(0,0,0,.06),0 4px 16px rgba(0,0,0,.06)",border:`1px solid ${C.gray200}`,cursor:onClick?"pointer":"default",...style}}>{children}</div>;
}

function Btn({ children, onClick, variant="primary", size="md", style={}, disabled=false }) {
  const base={border:"none",borderRadius:10,cursor:disabled?"not-allowed":"pointer",fontWeight:700,display:"inline-flex",alignItems:"center",gap:6,transition:"opacity .15s",opacity:disabled?.5:1,fontFamily:"inherit"};
  const v={
    primary:{background:`linear-gradient(135deg,${C.leafMid},${C.leaf})`,color:"#fff",boxShadow:"0 2px 8px rgba(46,125,50,.35)"},
    danger:{background:`linear-gradient(135deg,${C.dangerMid},${C.danger})`,color:"#fff"},
    ghost:{background:C.leafFaint,color:C.leaf,border:`1px solid ${C.leafPale}`},
    sky:{background:`linear-gradient(135deg,#0284C7,${C.sky})`,color:"#fff"},
    outline:{background:C.white,color:C.leaf,border:`1.5px solid ${C.leaf}`},
    ghost2:{background:C.gray100,color:C.gray700,border:`1px solid ${C.gray200}`},
    receipt:{background:"linear-gradient(135deg,#7C3AED,#6D28D9)",color:"#fff"},
    whatsapp:{background:"linear-gradient(135deg,#25D366,#128C7E)",color:"#fff"},
  };
  const sz={sm:{padding:"5px 12px",fontSize:12},md:{padding:"9px 18px",fontSize:14},lg:{padding:"12px 26px",fontSize:15}};
  return <button onClick={onClick} disabled={disabled} style={{...base,...v[variant],...sz[size],...style}}>{children}</button>;
}

function Input({ label, ...props }) {
  return (
    <div style={{marginBottom:14}}>
      {label&&<label style={{display:"block",fontSize:12,fontWeight:700,color:C.gray700,marginBottom:5,textTransform:"uppercase",letterSpacing:.5}}>{label}</label>}
      <input {...props} style={{width:"100%",padding:"10px 13px",border:`1.5px solid ${C.gray200}`,borderRadius:10,fontSize:14,color:C.gray900,background:C.white,boxSizing:"border-box",outline:"none",fontFamily:"inherit",...props.style}}/>
    </div>
  );
}

function Sel({ label, children, ...props }) {
  return (
    <div style={{marginBottom:14}}>
      {label&&<label style={{display:"block",fontSize:12,fontWeight:700,color:C.gray700,marginBottom:5,textTransform:"uppercase",letterSpacing:.5}}>{label}</label>}
      <select {...props} style={{width:"100%",padding:"10px 13px",border:`1.5px solid ${C.gray200}`,borderRadius:10,fontSize:14,color:C.gray900,background:C.white,boxSizing:"border-box",outline:"none",fontFamily:"inherit"}}>{children}</select>
    </div>
  );
}

function Modal({ title, onClose, children, width=520 }) {
  return (
    <div style={{position:"fixed",inset:0,background:"rgba(0,0,0,.55)",zIndex:1000,display:"flex",alignItems:"center",justifyContent:"center",padding:16}}>
      <div style={{background:C.white,borderRadius:20,padding:"28px 28px 24px",width:"100%",maxWidth:width,maxHeight:"92vh",overflowY:"auto",boxShadow:"0 24px 80px rgba(0,0,0,.22)"}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:22}}>
          <h2 style={{margin:0,color:C.soilMid,fontSize:18,fontWeight:900}}>{title}</h2>
          <button onClick={onClose} style={{background:C.gray100,border:"none",borderRadius:8,width:32,height:32,cursor:"pointer",fontSize:16,color:C.gray500,fontFamily:"inherit"}}>✕</button>
        </div>
        {children}
      </div>
    </div>
  );
}

function ConfirmDialog({ message, sub="", onConfirm, onCancel }) {
  return (
    <div style={{position:"fixed",inset:0,background:"rgba(0,0,0,.55)",zIndex:2000,display:"flex",alignItems:"center",justifyContent:"center",padding:16}}>
      <div style={{background:C.white,borderRadius:18,padding:32,maxWidth:360,width:"100%",textAlign:"center"}}>
        <div style={{width:56,height:56,background:C.dangerLight,borderRadius:"50%",display:"flex",alignItems:"center",justifyContent:"center",margin:"0 auto 16px",fontSize:26}}>🗑️</div>
        <div style={{fontSize:17,fontWeight:800,color:C.gray900,marginBottom:6}}>Are you sure?</div>
        <div style={{fontSize:14,color:C.gray500,marginBottom:4}}>{message}</div>
        {sub&&<div style={{fontSize:12,color:C.danger}}>{sub}</div>}
        <div style={{display:"flex",gap:10,justifyContent:"center",marginTop:20}}>
          <Btn onClick={onCancel} variant="ghost2">Cancel</Btn>
          <Btn onClick={onConfirm} variant="danger">Yes, Delete</Btn>
        </div>
      </div>
    </div>
  );
}

// ── Digital Stamp ─────────────────────────────────────────────────────
function DigitalStamp() {
  const size=150, r=75, outerR=71, innerR=59;
  return (
    <div style={{position:"absolute",bottom:110,right:14,width:size,height:size,transform:"rotate(-18deg)",opacity:.82,zIndex:10,pointerEvents:"none",filter:"drop-shadow(0 2px 6px rgba(46,125,50,.35))"}}>
      <svg width={size} height={size} viewBox={`0 0 ${size} ${size}`}>
        <defs>
          <path id="stampTop" d={`M ${r-outerR+6},${r} A ${outerR-6},${outerR-6} 0 0,1 ${r+outerR-6},${r}`}/>
          <path id="stampBot" d={`M ${r-innerR+4},${r+4} A ${innerR-4},${innerR-4} 0 0,0 ${r+innerR-4},${r+4}`}/>
        </defs>
        <circle cx={r} cy={r} r={outerR} fill="none" stroke="#1B5E20" strokeWidth={4}/>
        <circle cx={r} cy={r} r={outerR-6} fill="none" stroke="#2E7D32" strokeWidth={1.8} strokeDasharray="6 4"/>
        <circle cx={r} cy={r} r={innerR} fill="rgba(232,245,233,.55)" stroke="#1B5E20" strokeWidth={2.5}/>
        <text fontSize="9.5" fontWeight="900" letterSpacing="1.8" fill="#1B5E20" fontFamily="'Segoe UI',system-ui,sans-serif">
          <textPath href="#stampTop" startOffset="50%" textAnchor="middle">KHALID ZARAI SPRAY CENTER</textPath>
        </text>
        <text fontSize="8.5" fontWeight="800" letterSpacing="1.4" fill="#1B5E20" fontFamily="'Segoe UI',system-ui,sans-serif">
          <textPath href="#stampBot" startOffset="50%" textAnchor="middle">✦ VERIFIED · KZSC ✦</textPath>
        </text>
        <text x={r} y={r-8} textAnchor="middle" fontSize="28" style={{userSelect:"none"}}>🌿</text>
        <text x={r} y={r+10} textAnchor="middle" fontSize="8.5" fontWeight="900" fill="#1B5E20" letterSpacing="1.5" fontFamily="'Segoe UI',system-ui,sans-serif">OFFICIAL</text>
        <text x={r} y={r+22} textAnchor="middle" fontSize="7.5" fontWeight="700" fill="#2E7D32" letterSpacing="1" fontFamily="'Segoe UI',system-ui,sans-serif">MOHAR / مہر</text>
        {[0,90,180,270].map(deg => {
          const rad=(deg*Math.PI)/180, tx=r+(outerR-2)*Math.cos(rad), ty=r+(outerR-2)*Math.sin(rad);
          return <text key={deg} x={tx} y={ty+3} textAnchor="middle" fontSize="7" fill="#1B5E20" style={{userSelect:"none"}}>✦</text>;
        })}
      </svg>
    </div>
  );
}

// ── Receipt Modal ─────────────────────────────────────────────────────
function ReceiptModal({ tx, customer, allTxForCustomer, onClose }) {
  const isSale = tx.type === "sale";
  const { prevBalance, afterThisTx } = getReceiptData(tx, allTxForCustomer);
  const totalBill  = tx.total;
  const amountPaid = tx.paid;
  const creditGiven = totalBill - amountPaid;
  const newBalance  = afterThisTx;
  const isNaqad   = isSale && amountPaid >= totalBill;
  const isPartial = isSale && amountPaid > 0 && amountPaid < totalBill;
  const isFullUdhaar = isSale && amountPaid === 0;
  const receiptNo = `KZSC-${tx.id.toUpperCase().slice(0,8)}`;
  const now = new Date();

  function buildShareText() {
    const lines = [
      `🌿 *Khalid Zarai Spray Center*`,`خالد زراعی اسپرے سینٹر`,`─────────────────`,
      `📋 Bill No: ${receiptNo}`,`📅 Date: ${fmtDate(tx.date)}`,`👨‍🌾 Farmer: ${tx.customerName}`,
      `📍 Village: ${customer?.village||"—"}`,`─────────────────`,
    ];
    if (isSale && tx.items.length > 0) {
      lines.push(`🛒 *Items Purchased:*`);
      tx.items.forEach(it => lines.push(`  • ${it.name} x${it.qty} @ ${fmt(it.price)} = ${fmt(it.qty*it.price)}`));
      lines.push(`─────────────────`);
      lines.push(`💰 *Total Bill: ${fmt(totalBill)}*`);
      lines.push(`✅ Amount Paid: ${fmt(amountPaid)}`);
      if (creditGiven > 0) lines.push(`📒 Udhaar (This Sale): ${fmt(creditGiven)}`);
    } else {
      lines.push(`💵 *Payment Received: ${fmt(amountPaid)}*`);
    }
    lines.push(`─────────────────`);
    lines.push(`📌 Previous Balance: ${fmt(prevBalance)}`);
    lines.push(`⚖️ *New Remaining Udhaar: ${fmt(newBalance)}*`);
    lines.push(`─────────────────`);
    lines.push(newBalance > 0 ? `⚠️ Baqi Udhaar: ${fmt(newBalance)}` : `✅ Account Cleared!`);
    lines.push(`\nشکریہ · Thank you · KZSC`);
    return lines.join("\n");
  }

  async function handleShare() {
    const text = buildShareText();
    if (navigator.share) {
      try { await navigator.share({ title:`KZSC Receipt – ${tx.customerName}`, text }); return; }
      catch(e) { if (e.name==="AbortError") return; }
    }
    try {
      const encoded = encodeURIComponent(text);
      const win = window.open(`https://wa.me/?text=${encoded}`, "_blank", "noopener,noreferrer");
      if (win) return;
    } catch {}
    try {
      await navigator.clipboard.writeText(text);
      alert("✅ Receipt copied to clipboard!\nPaste it in WhatsApp or any messaging app.");
    } catch {
      window.prompt("Copy this receipt text:", text);
    }
  }

  return (
    <div style={{position:"fixed",inset:0,background:"rgba(0,0,0,.68)",zIndex:3000,display:"flex",alignItems:"center",justifyContent:"center",padding:10,overflowY:"auto"}}>
      <div style={{background:"#fff",borderRadius:22,width:"100%",maxWidth:400,boxShadow:"0 32px 100px rgba(0,0,0,.4)",fontFamily:"'Segoe UI',system-ui,sans-serif",position:"relative",margin:"auto"}}>
        <button onClick={onClose} style={{position:"absolute",top:13,right:13,zIndex:10,background:"rgba(0,0,0,.22)",border:"none",borderRadius:8,width:30,height:30,cursor:"pointer",color:"#fff",fontSize:14,display:"flex",alignItems:"center",justifyContent:"center",fontFamily:"inherit"}}>✕</button>
        {/* Header */}
        <div style={{background:`linear-gradient(150deg,${C.soil} 0%,${C.soilLight} 100%)`,borderRadius:"22px 22px 0 0",padding:"22px 22px 20px",textAlign:"center"}}>
          <div style={{width:50,height:50,background:"linear-gradient(135deg,#4CAF50,#2E7D32)",borderRadius:14,display:"flex",alignItems:"center",justifyContent:"center",fontSize:24,margin:"0 auto 9px",boxShadow:"0 4px 14px rgba(76,175,80,.45)"}}>🌿</div>
          <div style={{color:"#fff",fontWeight:900,fontSize:16,letterSpacing:.4}}>Khalid Zarai Spray Center</div>
          <div style={{color:"rgba(255,255,255,.45)",fontSize:9,marginTop:2,letterSpacing:1,textTransform:"uppercase"}}>خالد زراعی اسپرے سینٹر · Official Bill</div>
          <div style={{marginTop:13,display:"inline-flex",alignItems:"center",gap:13,background:"rgba(255,255,255,.09)",border:"1px solid rgba(255,255,255,.14)",borderRadius:14,padding:"10px 18px"}}>
            <div style={{textAlign:"left"}}>
              <div style={{color:"rgba(255,255,255,.5)",fontSize:9,fontWeight:700,textTransform:"uppercase",letterSpacing:.8}}>{isSale?"Total Bill":"Amount Received"}</div>
              <div style={{color:"#FCD34D",fontSize:22,fontWeight:900,lineHeight:1.1,marginTop:2}}>{fmt(totalBill)}</div>
            </div>
            <div style={{width:1,height:34,background:"rgba(255,255,255,.18)"}}/>
            <div>
              {isSale ? (
                <div style={{background:isNaqad?"rgba(134,239,172,.2)":isPartial?"rgba(251,191,36,.2)":"rgba(252,100,100,.2)",border:`1px solid ${isNaqad?"rgba(134,239,172,.45)":isPartial?"rgba(251,191,36,.45)":"rgba(252,100,100,.45)"}`,borderRadius:18,padding:"5px 12px",textAlign:"center"}}>
                  <div style={{color:isNaqad?"#86EFAC":isPartial?"#FCD34D":"#FCA5A5",fontSize:12,fontWeight:900}}>{isNaqad?"✅ نقد":isPartial?"🔶 جزوی":"📒 ادھار"}</div>
                  <div style={{color:isNaqad?"rgba(134,239,172,.65)":isPartial?"rgba(252,211,77,.65)":"rgba(252,165,165,.65)",fontSize:8,fontWeight:700,letterSpacing:.8,marginTop:1}}>{isNaqad?"NAQAD":isPartial?"PARTIAL":"UDHAAR"}</div>
                </div>
              ):(
                <div style={{background:"rgba(134,239,172,.2)",border:"1px solid rgba(134,239,172,.45)",borderRadius:18,padding:"5px 12px",textAlign:"center"}}>
                  <div style={{color:"#86EFAC",fontSize:12,fontWeight:900}}>💵 ادائیگی</div>
                  <div style={{color:"rgba(134,239,172,.65)",fontSize:8,fontWeight:700,letterSpacing:.8,marginTop:1}}>PAYMENT</div>
                </div>
              )}
            </div>
          </div>
        </div>
        {/* Tear */}
        <div style={{position:"relative",height:12,background:`linear-gradient(135deg,${C.soil},${C.soilLight})`}}>
          <div style={{position:"absolute",bottom:-1,left:0,right:0,height:13,background:"#fff",borderRadius:"50% 50% 0 0 / 100% 100% 0 0"}}/>
        </div>
        <div style={{borderTop:"2px dashed #E5E7EB",margin:"0 18px"}}/>
        {/* Body */}
        <div style={{padding:"14px 20px 20px"}}>
          {/* Meta */}
          <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:"3px 10px",marginBottom:14}}>
            {[{l:"Bill No",v:receiptNo},{l:"Date",v:fmtDate(tx.date)},{l:"Farmer",v:tx.customerName},{l:"Village",v:customer?.village||"—"},{l:"Contact",v:customer?.phone||"—"},{l:"Time",v:now.toLocaleTimeString("en-PK",{hour:"2-digit",minute:"2-digit"})}].map(r=>(
              <div key={r.l} style={{padding:"5px 0",borderBottom:`1px solid ${C.gray100}`}}>
                <div style={{fontSize:9,color:C.gray400,fontWeight:700,textTransform:"uppercase",letterSpacing:.4}}>{r.l}</div>
                <div style={{fontSize:12,color:C.gray900,fontWeight:700,marginTop:1,wordBreak:"break-all"}}>{r.v}</div>
              </div>
            ))}
          </div>
          {/* Items */}
          {isSale ? (
            <div style={{marginBottom:14}}>
 
