elite-auto-sales-app/
├─ package.json
├─ server/
│  └─ index.js
├─ public/
│  └─ index.html
├─ src/
│  ├─ index.js
│  ├─ App.js
│  ├─ firebase.js
│  └─ components/
│     ├─ StaffPortal.js
│     └─ CustomerPortal.js
└─ functions/
   └─ weeklyInsuranceCheck.js
   {
  "name": "elite-auto-sales-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node server/index.js",
    "dev": "concurrently \"nodemon server/index.js\" \"parcel public/index.html --open\""
  },
  "dependencies": {
    "express": "^4.18.2",
    "stripe": "^12.0.0",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2"
  },
  "devDependencies": {
    "concurrently": "^7.6.0",
    "nodemon": "^2.0.22",
    "parcel": "^2.9.3"
  }
}
// server/index.js
const express = require('express');
const path = require('path');
const bodyParser = require('body-parser');
const cors = require('cors');

const STRIPE_SECRET = process.env.STRIPE_SECRET_KEY || 'sk_test_PLACEHOLDER';
const Stripe = require('stripe');
const stripe = Stripe(STRIPE_SECRET);

const app = express();
app.use(cors());
app.use(bodyParser.json());

// Stripe: create payment intent
app.post('/create-payment-intent', async (req, res) => {
  try {
    const { amount } = req.body;
    if (!amount) return res.status(400).json({ error: 'Amount required' });
    const paymentIntent = await stripe.paymentIntents.create({
      amount: parseInt(amount, 10),
      currency: 'usd',
      automatic_payment_methods: { enabled: true }
    });
    res.json({ clientSecret: paymentIntent.client_secret });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: err.message });
  }
});

// Serve static SPA
app.use(express.static(path.join(__dirname, '..', 'public')));
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, '..', 'public', 'index.html'));
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server listening on ${PORT}`));
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Elite Auto Sales</title>
  <style>
    body{font-family:Arial,Helvetica,sans-serif;background:#f5f6f8;margin:0;padding:12px}
    .container{max-width:1000px;margin:0 auto}
    .header{display:flex;justify-content:space-between;align-items:center}
    h1{margin:0;font-size:20px}
    .card{background:#fff;padding:12px;border-radius:8px;margin-top:12px;border:1px solid #e6e6e6}
    input,select,textarea{width:100%;padding:8px;margin-top:8px;border:1px solid #ddd;border-radius:6px}
    .row{display:flex;gap:8px}
    .half{flex:1}
    .btn{background:#007bff;color:#fff;padding:10px;border-radius:6px;border:0;margin-top:10px;cursor:pointer}
    img.vehicle{width:100%;height:150px;object-fit:cover;border-radius:6px;margin-top:8px}
    .alert{background:#fff4f4;border:1px solid #ffcfcf;padding:8px;border-radius:6px;margin-top:8px}
  </style>

  <!-- React (CDN) -->
  <script src="https://unpkg.com/react@18/umd/react.development.js"></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>

  <!-- Firebase compat for simplicity -->
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-firestore-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-auth-compat.js"></script>
</head>
<body>
  <div id="root" class="container"></div>

<script>
/* ====== PASTE YOUR FIREBASE CONFIG HERE ======
   (from Firebase Console -> Project Settings -> Add Web App)
*/
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};
/* ============================================= */

firebase.initializeApp(firebaseConfig);
const db = firebase.firestore();
const auth = firebase.auth();

const e = React.createElement;
const useState = React.useState;
const useEffect = React.useEffect;

/* ---------- App: main SPA ---------- */
function App(){
  const [view, setView] = useState('staff'); // staff or customer
  const [uid, setUid] = useState('');
  return e('div',null,
    e('div',{className:'header'}, e('h1',null,'Elite Auto Sales'), e('div',null,
      e('select',{value:view,onChange:ev=>setView(ev.target.value)}, e('option',{value:'staff'},'Staff Portal'), e('option',{value:'customer'},'Customer Portal')),
      e('input',{placeholder:'Customer ID (for demo)', value:uid,onChange:ev=>setUid(ev.target.value), style:{marginLeft:8,padding:8,borderRadius:6,border:'1px solid #ddd'}})
    )),
    view==='staff' ? e(StaffPortal,null) : e(CustomerPortal,{customerId:uid})
  );
}

/* ---------- StaffPortal component (frazer-like simplified) ---------- */
function StaffPortal(){
  const [vehicles,setVehicles] = useState([]);
  const [customers,setCustomers] = useState([]);
  const [alerts,setAlerts] = useState([]);
  const [form,setForm] = useState({make:'',model:'',price:'',downPayment:'',imageUrl:'',maintenance:''});
  const [manualPay,setManualPay] = useState({customerId:'',vehicleId:'',amount:'',method:'Cash',notes:''});
  const [stripePay,setStripePay] = useState({customerId:'',vehicleId:'',amount:''});
  const [insEdit,setInsEdit] = useState({});

  useEffect(()=>{
    const u1 = db.collection('vehicles').onSnapshot(s=>{
      const arr=[]; s.forEach(d=>arr.push({id:d.id,...d.data()})); setVehicles(arr);
    });
    const u2 = db.collection('customers').onSnapshot(s=>{
      const arr=[]; s.forEach(d=>arr.push({id:d.id,...d.data()})); setCustomers(arr);
    });
    const u3 = db.collection('staffAlerts').orderBy('date','desc').onSnapshot(s=>{
      const arr=[]; s.forEach(d=>arr.push({id:d.id,...d.data()})); setAlerts(arr);
    });
    return ()=>{u1();u2();u3();}
  },[]);

  const addVehicle = async ()=>{
    if(!form.make||!form.model||!form.price) return alert('fill required fields');
    await db.collection('vehicles').add({
      make: form.make, model: form.model, price: parseFloat(form.price||0),
      downPayment: parseFloat(form.downPayment||0),
      maintenance: form.maintenance||'', imageUrl: form.imageUrl||'', sold:false, createdAt: new Date()
    });
    setForm({make:'',model:'',price:'',downPayment:'',imageUrl:'',maintenance:''});
    alert('Vehicle added');
  };

  const addCustomer = async ()=>{
    const name = prompt('Customer name?'); if(!name) return;
    const ref = await db.collection('customers').add({ name, email:'', phone:'', vehicleId:null, balanceRemaining:0, paymentPlan:'semi-monthly', nextPaymentDue:null, pickupNote:null, insurance:null, createdAt:new Date()});
    alert('Customer created: '+ref.id);
  };

  const manualPayment = async ()=>{
    if(!manualPay.customerId || !manualPay.amount) return alert('fill customer and amount');
    await db.collection('payments').add({ customerId:manualPay.customerId, vehicleId:manualPay.vehicleId||null, amount:parseFloat(manualPay.amount), method:manualPay.method, notes:manualPay.notes||'', status:'paid', timestamp:new Date()});
    const custRef = db.collection('customers').doc(manualPay.customerId);
    const snap = await custRef.get();
    if(snap.exists){ await custRef.update({ balanceRemaining: (snap.data().balanceRemaining||0) - parseFloat(manualPay.amount) }); }
    alert('Manual payment recorded');
    setManualPay({customerId:'',vehicleId:'',amount:'',method:'Cash',notes:''});
  };

  const stripeProcess = async ()=>{
    if(!stripePay.customerId || !stripePay.amount) return alert('enter details');
    try{
      const resp = await fetch('/create-payment-intent',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({amount:Math.round(parseFloat(stripePay.amount)*100)})});
      const j = await resp.json(); if(j.error) throw new Error(j.error||'stripe error');
      await db.collection('payments').add({ customerId:stripePay.customerId, vehicleId:stripePay.vehicleId||null, amount:parseFloat(stripePay.amount), method:'Stripe', status:'paid', timestamp:new Date()});
      const custRef = db.collection('customers').doc(stripePay.customerId); const snap = await custRef.get();
      if(snap.exists) await custRef.update({ balanceRemaining: (snap.data().balanceRemaining||0) - parseFloat(stripePay.amount) });
      alert('Stripe Payment (demo) recorded; PaymentIntent created on server (client confirm omitted in demo).');
      setStripePay({customerId:'',vehicleId:'',amount:''});
    }catch(e){ alert('Stripe error: '+e.message); }
  };

  const runInsuranceCheckNow = async ()=>{
    const snap = await db.collection('customers').get(); let created=0;
    for(const d of snap.docs){
      const data = d.data(); const ins = data.insurance||{}; const issues=[];
      if(ins.coverage !== 'full') issues.push('Coverage not full');
      if(!ins.deductible || ins.deductible < 500 || ins.deductible > 1000) issues.push('Deductible out of range');
      if(ins.lienholder !== 'Elite Auto Sales') issues.push('Lienholder not correct');
      if(issues.length>0){ await db.collection('staffAlerts').add({ type:'Insurance Issue', customerId:d.id, vehicleId:data.vehicleId||null, date:new Date(), details:issues.join('; '), seen:false }); created++; }
    }
    alert('Insurance check run. Alerts created: '+created);
  };

  const markHandled = async (id)=>{ await db.collection('staffAlerts').doc(id).update({ seen:true }); };

  return e('div',null,
    e('div',{className:'card'}, e('h2',null,'Add Vehicle'),
      e('div',{className:'row'}, e('input',{className:'half',placeholder:'Make',value:form.make,onChange:ev=>setForm({...form,make:ev.target.value})}), e('input',{className:'half',placeholder:'Model',value:form.model,onChange:ev=>setForm({...form,model:ev.target.value})})),
      e('div',{className:'row'}, e('input',{className:'half',placeholder:'Price',value:form.price,onChange:ev=>setForm({...form,price:ev.target.value})}), e('input',{className:'half',placeholder:'Down Payment',value:form.downPayment,onChange:ev=>setForm({...form,downPayment:ev.target.value})})),
      e('input',{placeholder:'Image URL',value:form.imageUrl,onChange:ev=>setForm({...form,imageUrl:ev.target.value})}),
      e('textarea',{placeholder:'Maintenance notes',value:form.maintenance,onChange:ev=>setForm({...form,maintenance:ev.target.value})}),
      e('button',{className:'btn',onClick:addVehicle},'Add Vehicle')
    ),
    e('div',{className:'card'}, e('h2',null,'Vehicles'), vehicles.length===0?e('div',null,'No vehicles yet'):vehicles.map(v=> e('div',{key:v.id, style:{padding:8,border:'1px solid #eee',borderRadius:6,marginTop:8}},
      e('strong',null,v.make+' '+v.model),
      e('div',null,'Price: $'+(v.price||0)),
      e('div',null,'Down Payment: $'+(v.downPayment||0)),
      v.imageUrl? e('img',{src:v.imageUrl,className:'vehicle'}):null
    ))),
    e('div',{className:'card'}, e('h2',null,'Customers'), e('button',{className:'btn',onClick:addCustomer},'Add Customer'),
      customers.length===0?e('div',null,'No customers yet'):customers.map(c=> e('div',{key:c.id, style:{padding:8,border:'1px solid #eee',borderRadius:6,marginTop:8}},
        e('div',{style:{fontWeight:'bold'}}, c.name||('ID:'+c.id)),
        e('div',null,'Balance: $'+(c.balanceRemaining||0)),
        e('div',null,'Pickup Note: '+(c.pickupNote?('$'+c.pickupNote.amount+' due '+ new Date(c.pickupNote.dueDate).toLocaleDateString()):'None')),
        e('div',null,'Insurance: '+(c.insurance? (c.insurance.company+' ('+ (c.insurance.coverage||'') +')') : 'None')),
        e('div',null,' '),
        e('div',null,
          e('input',{placeholder:'Company', value:insEdit[c.id]?.company ?? (c.insurance?.company||''), onChange:ev=>setInsEdit({...insEdit,[c.id]:{...(insEdit[c.id]||{}),company:ev.target.value}})}),
          e('input',{placeholder:'Policy #', value:insEdit[c.id]?.policyNumber ?? (c.insurance?.policyNumber||''), onChange:ev=>setInsEdit({...insEdit,[c.id]:{...(insEdit[c.id]||{}),policyNumber:ev.target.value}})}),
          e('input',{placeholder:'Coverage', value:insEdit[c.id]?.coverage ?? (c.insurance?.coverage||''), onChange:ev=>setInsEdit({...insEdit,[c.id]:{...(insEdit[c.id]||{}),coverage:ev.target.value}})}),
          e('input',{placeholder:'Deductible', value:insEdit[c.id]?.deductible ?? (c.insurance?.deductible||''), onChange:ev=>setInsEdit({...insEdit,[c.id]:{...(insEdit[c.id]||{}),deductible:ev.target.value}})}),
          e('input',{placeholder:'Lienholder', value:insEdit[c.id]?.lienholder ?? (c.insurance?.lienholder||'Elite Auto Sales'), onChange:ev=>setInsEdit({...insEdit,[c.id]:{...(insEdit[c.id]||{}),lienholder:ev.target.value}})}),
          e('input',{placeholder:'Expiration', value:insEdit[c.id]?.expiration ?? (c.insurance?.expiration||''), onChange:ev=>setInsEdit({...insEdit,[c.id]:{...(insEdit[c.id]||{}),expiration:ev.target.value}})}),
          e('button',{className:'btn', onClick:()=>{ db.collection('customers').doc(c.id).update({ insurance: insEdit[c.id]||c.insurance }); alert('Insurance saved'); }}, 'Save Insurance')
        )
      ))),
    e('div',{className:'card'}, e('h2',null,'Payments / Manual'),
      e('input',{placeholder:'Customer ID', value:manualPay.customerId, onChange:ev=>setManualPay({...manualPay,customerId:ev.target.value})}),
      e('input',{placeholder:'Vehicle ID (optional)', value:manualPay.vehicleId, onChange:ev=>setManualPay({...manualPay,vehicleId:ev.target.value})}),
      e('input',{placeholder:'Amount', value:manualPay.amount, onChange:ev=>setManualPay({...manualPay,amount:ev.target.value})}),
      e('select',{value:manualPay.method,onChange:ev=>setManualPay({...manualPay,method:ev.target.value})}, e('option',{value:'Cash'},'Cash'), e('option',{value:'Card'},'Card'), e('option',{value:'Phone'},'Phone')),
      e('textarea',{placeholder:'Notes', value:manualPay.notes, onChange:ev=>setManualPay({...manualPay,notes:ev.target.value})}),
      e('button',{className:'btn', onClick:manualPayment}, 'Record Manual Payment')
    ),
    e('div',{className:'card'}, e('h2',null,'Stripe (demo)'),
      e('input',{placeholder:'Customer ID', value:stripePay.customerId, onChange:ev=>setStripePay({...stripePay,customerId:ev.target.value})}),
      e('input',{placeholder:'Vehicle ID', value:stripePay.vehicleId, onChange:ev=>setStripePay({...stripePay,vehicleId:ev.target.value})}),
      e('input',{placeholder:'Amount (USD)', value:stripePay.amount, onChange:ev=>setStripePay({...stripePay,amount:ev.target.value})}),
      e('div',{className:'small'},'Creates a PaymentIntent server-side; client confirm step omitted (demo).'),
      e('button',{className:'btn', onClick:stripeProcess}, 'Process Stripe Payment')
    ),
    e('div',{className:'card'}, e('h2',null,'Insurance Compliance (manual)'),
      e('button',{className:'btn', onClick:runInsuranceCheckNow}, 'Run Insurance Check Now')
    ),
    e('div',{className:'card'}, e('h2',null,'Alerts'), alerts.length===0?e('div',null,'No alerts') : alerts.map(a => e('div',{key:a.id,className:'alert'},
      e('div',null, e('strong',null,a.type)),
      e('div',null,'Customer: '+(a.customerId||'N/A')),
      e('div',null,'Details: '+(a.details||'')),
      e('div',null, !a.seen ? e('button',{className:'btn', onClick:()=>markHandled(a.id)}, 'Mark Handled') : e('div',null,'Handled'))
    )))
  );
}

/* ---------- CustomerPortal component ---------- */
function CustomerPortal({customerId}){
  const [cust,setCust]=useState(null);
  useEffect(()=>{
    if(!customerId) { setCust(null); return; }
    const unsub = db.collection('customers').doc(customerId).onSnapshot(s=> setCust(s.exists? {id:s.id,...s.data()} : null));
    return ()=>unsub();
  },[customerId]);

  return e('div',null,
    e('div',{className:'card'}, e('h2',null,'Customer Portal'),
      customerId ? (cust ? e('div',null,
        e('div',null, e('strong',null,'Name: '), cust.name),
        e('div',null, e('strong',null,'Balance: $'), (cust.balanceRemaining||0)),
        e('div',null, e('strong',null,'Next Due: '), cust.nextPaymentDue? new Date(cust.nextPaymentDue).toLocaleDateString(): 'N/A'),
        e('div',null, e('strong',null,'Pickup Note: '), cust.pickupNote ? ('$'+cust.pickupNote.amount + ' due ' + new Date(cust.pickupNote.dueDate).toLocaleDateString()) : 'None'),
        e('div',null, e('strong',null,'Insurance: '), cust.insurance ? (cust.insurance.company+' ('+cust.insurance.coverage+')') : 'None')
      ) : e('div',null,'Loading...')) : e('div',null,'Enter your Customer ID in the top-right to view your account')
    ),
    e('div',{className:'card'}, e('h3',null,'Customer Expectations'),
      e('div',null,'• Must maintain full coverage insurance'),
      e('div',null,'• Deductible must be $500–$1000'),
      e('div',null,'• Elite Auto Sales must be listed as lienholder'),
      e('div',null,'• Customer must notify dealership of insurance changes'),
      e('div',null,'• Vehicle will not be altered without permission'),
      e('div',null,'• Tires/rims will not be sold without consent')
    )
  );
}

/* Render */
ReactDOM.createRoot(document.getElementById('root')).render(e(App));
</script>
</body>
</html>
// functions/weeklyInsuranceCheck.js
const functions = require("firebase-functions");
const admin = require("firebase-admin");
admin.initializeApp();
const db = admin.firestore();

exports.weeklyInsuranceCheck = functions.pubsub
  .schedule("0 10 * * 1") // Monday 10:00 AM America/Chicago
  .timeZone("America/Chicago")
  .onRun(async () => {
    const customersSnap = await db.collection("customers").get();
    for(const docSnap of customersSnap.docs){
      const customer = docSnap.data() || {};
      const insurance = customer.insurance || {};
      const issues = [];
      if(insurance.coverage !== "full") issues.push("Coverage not full");
      if(!insurance.deductible || insurance.deductible < 500 || insurance.deductible > 1000) issues.push("Deductible out of range");
      if(insurance.lienholder !== "Elite Auto Sales") issues.push("Lienholder not correct");
      if(issues.length>0){
        await db.collection('staffAlerts').add({
          type: "Insurance Issue",
          customerId: docSnap.id,
          vehicleId: customer.vehicleId || null,
          date: new Date(),
          details: issues.join('; '),
          seen: false
        });
      }
    }
    return null;
  });
