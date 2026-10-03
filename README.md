# QR-SPLITTER-APP
export function generateQRCodes(totalBill, upiIds) {
  if (!upiIds || upiIds.length === 0) throw new Error("Kam se kam 1 UPID chahiye");
  const MAX = 1999; const MIN_QR_AMOUNT_SPLIT = 100;
  if (totalBill <= MAX) {
    return [{ upi: upiIds[0], amount: totalBill, upiLink: `upi://pay?pa=${upiIds[0]}&am=${totalBill}&cu=INR` }];
  }
  let k = Math.ceil(totalBill / MAX);
  while (k * MAX - (k * (k - 1)) / 2 < totalBill) { k++; }
  function generateRandomUniqueAmounts(total, count) {
    for (let attempt = 0; attempt < 200; attempt++) {
      let used = new Set(); let amounts = []; let remaining = total; let success = true;
      for (let i = 0; i < count - 1; i++) {
        let partsLeft = count - i - 1;
        let low = Math.max(MIN_QR_AMOUNT_SPLIT, remaining - partsLeft * MAX);
        let high = Math.min(MAX, remaining - partsLeft * MIN_QR_AMOUNT_SPLIT);
        if (low > high) { success = false; break; }
        let val, tries = 0;
        do { val = Math.floor(Math.random() * (high - low + 1)) + low; tries++; } while (used.has(val) && tries < 100);
        if (used.has(val)) { success = false; break; }
        used.add(val); amounts.push(val); remaining -= val;
      }
      if (success && remaining >= MIN_QR_AMOUNT_SPLIT && remaining <= MAX &&!used.has(remaining)) { amounts.push(remaining); return amounts; }
    }
    return null;
  }
  let amounts = generateRandomUniqueAmounts(totalBill, k);
  if (!amounts) throw new Error("Split failed");
  let qrList = [];
  for (let i = 0; i < amounts.length; i++) {
    let upi = upiIds[i % upiIds.length];
    qrList.push({ upi: upi, amount: amounts[i], upiLink: `upi://pay?pa=${upi}&am=${amounts[i]}&cu=INR` });
  }
  return qrList;
}
import { useState, useEffect } from 'react';
import { generateQRCodes } from './utils/qrGenerator';
import QRCode from 'qrcode';

// JPG Share Function
async function shareAsJPG(qrList: any[], shopName: string) {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d')!;
  const QR_SIZE = 400; const PADDING = 50;
  const ITEM_H = 550;
  canvas.width = QR_SIZE + PADDING * 2;
  canvas.height = qrList.length * ITEM_H;
  ctx.fillStyle = 'white'; ctx.fillRect(0,0,canvas.width, canvas.height);
  for(let i=0; i<qrList.length; i++){
    const y = i * ITEM_H;
    const qrDataUrl = await QRCode.toDataURL(qrList[i].upiLink, {width: QR_SIZE});
    const img = new Image();
    await new Promise(r=>{img.onload=r; img.src=qrDataUrl});
    ctx.fillStyle='black'; ctx.font='bold 22px Arial'; ctx.textAlign='center';
    ctx.fillText(`QR No. ${i+1} - ${shopName}`, canvas.width/2, y+40);
    ctx.drawImage(img, PADDING, y+70, QR_SIZE, QR_SIZE);
    ctx.font='bold 32px Arial';
    ctx.fillText(`Rs. ${qrList[i].amount}`, canvas.width/2, y+520);
  }
  canvas.toBlob(async (blob)=>{
    const file = new File([blob!], 'QR.jpg', {type: 'image/jpeg'});
    if(navigator.canShare && navigator.canShare({files:[file]})){
      await navigator.share({files:[file], title: shopName});
    } else {
      const url = URL.createObjectURL(blob!);
      const a = document.createElement('a'); a.href=url; a.download='QR.jpg'; a.click();
    }
  }, 'image/jpeg');
}

export default function App(){
  const [shopName, setShopName] = useState('The Baby Shop');
  const [total, setTotal] = useState(5997);
  const [upiInput, setUpiInput] = useState('masterjain@apl, 9422760741@okicici, jayantilalshiran@okhdfcbank');
  const [qrList, setQrList] = useState<any[]>([]);
  const [history, setHistory] = useState<any[]>([]);

  useEffect(()=>{ setHistory(JSON.parse(localStorage.getItem('qr_history')||'[]')) },[]);

  const handleGenerate = () => {
    const upiIds = upiInput.split(',').map(s=>s.trim()).filter(Boolean);
    const list = generateQRCodes(total, upiIds);
    setQrList(list);
    const newHistory = [{id:Date.now(), total, shopName, count:list.length, list, date:new Date().toLocaleString()},...history];
    setHistory(newHistory);
    localStorage.setItem('qr_history', JSON.stringify(newHistory));
  }

  return (
    <div style={{padding:20, fontFamily:'sans-serif'}}>
      <h2>QR Splitter - {shopName}</h2>
      <input value={shopName} onChange={e=>setShopName(e.target.value)} placeholder="Shop Name" style={{width:'100%', padding:10, marginBottom:10}}/>
      <input type="number" value={total} onChange={e=>setTotal(Number(e.target.value))} placeholder="Total Amount" style={{width:'100%', padding:10, marginBottom:10}}/>
      <textarea value={upiInput} onChange={e=>setUpiInput(e.target.value)} placeholder="UPI IDs comma se" style={{width:'100%', padding:10}}/>
      <button onClick={handleGenerate} style={{width:'100%', padding:15, background:'black', color:'white', marginTop:10}}>Generate QR</button>

      {qrList.length>0 && <button onClick={()=>shareAsJPG(qrList, shopName)} style={{width:'100%', padding:15, background:'green', color:'white', marginTop:10}}>Share as JPG Image</button>}

      <h3 style={{marginTop:30}}>History</h3>
      {history.map((h:any)=><div key={h.id} onClick={()=>setQrList(h.list)} style={{border:'1px solid #ccc', padding:10, marginBottom:5}}>Rs.{h.total} - {h.count} QR - {h.date}</div>)}
    </div>
  )
}
