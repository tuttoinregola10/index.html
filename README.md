
<!DOCTYPE html>
<html lang="it">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Tutto in Regola</title>
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<style>
*{box-sizing:border-box}
body{margin:0;background:#f4f6fa;color:#17243a;font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}
header{background:#172b4d;color:white;padding:28px 22px 32px;border-radius:0 0 25px 25px}
header small{color:#c5d2e5}
h1{font-size:27px;margin:8px 0}
main{padding:20px;max-width:650px;margin:auto}
.card{background:white;padding:20px;border-radius:17px;margin-bottom:16px;box-shadow:0 3px 15px #172b4d0d}
h2{font-size:19px;margin:0 0 15px}
input,select,button{width:100%;padding:14px;border-radius:10px;border:1px solid #dce2eb;font-size:16px;margin:6px 0}
button{background:#2457a6;color:white;border:0;font-weight:600}
button.secondary{background:#e9eef7;color:#24436e}
.item{padding:13px 0;border-bottom:1px solid #edf0f5}
.item:last-child{border:0}
.muted{color:#758197;font-size:13px}
.hidden{display:none}
#message{font-size:14px;color:#b23b3b;margin-top:8px}
</style>
</head>
<body>
<header>
<small>DOCUMENTI · SCADENZE · PAGAMENTI</small>
<h1>Tutto in Regola</h1>
<div>La tua vita, sempre sotto controllo.</div>
</header>

<main>
<section id="auth" class="card">
<h2>Benvenuto!</h2>
<p class="muted">Accedi o crea il tuo account personale.</p>
<input id="email" type="email" placeholder="La tua email">
<input id="password" type="password" placeholder="Password">
<button onclick="signIn()">Accedi</button>
<button class="secondary" onclick="signUp()">Crea un account</button>
<p id="message"></p>
</section>

<section id="app" class="hidden">
<div class="card">
<h2>Le tue scadenze</h2>
<p class="muted">Tieni tutto sotto controllo, in un unico posto.</p>
<div id="reminders">Caricamento...</div>
</div>

<div class="card">
<h2>+ Nuova scadenza</h2>
<input id="title" placeholder="Es. Assicurazione auto">
<select id="category">
<option>Auto</option>
<option>Casa</option>
<option>Documenti</option>
<option>Abbonamenti</option>
<option>Altro</option>
</select>
<label class="muted">Data di scadenza</label>
<input id="date" type="date">
<input id="amount" type="number" step="0.01" placeholder="Importo in euro (facoltativo)">
<button onclick="addReminder()">Salva scadenza</button>
<p id="saveMessage" class="muted"></p>
</div>
<button class="secondary" onclick="signOut()">Esci dal tuo account</button>
</section>
</main>

<script>
const SUPABASE_URL = "https://unisezzrzqvhcagoqwqw.supabase.co";
const SUPABASE_KEY = "sb_publishable_hsvNCbDm51ifV40rxS5alA_YCPoHjVq";
const db = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

const authBox=document.getElementById("auth");
const appBox=document.getElementById("app");

async function signUp(){
 const {error}=await db.auth.signUp({
  email:email.value.trim(),
  password:password.value
 });
 document.getElementById("message").textContent=
  error ? error.message : "Registrazione inviata! Controlla la tua email per confermare l'account.";
}

async function signIn(){
 const {error}=await db.auth.signInWithPassword({
  email:email.value.trim(),
  password:password.value
 });
 if(error){document.getElementById("message").textContent=error.message;return;}
 await refresh();
}

async function signOut(){
 await db.auth.signOut();
 showAuth();
}

function showAuth(){
 authBox.classList.remove("hidden");
 appBox.classList.add("hidden");
}

async function refresh(){
 const {data:{user}}=await db.auth.getUser();
 if(!user){showAuth();return;}
 authBox.classList.add("hidden");
 appBox.classList.remove("hidden");
 await loadReminders();
}

async function loadReminders(){
 const list=document.getElementById("reminders");
 const {data,error}=await db.from("reminders")
  .select("*").order("due_date",{ascending:true});
 if(error){list.textContent="Errore: "+error.message;return;}
 if(!data.length){list.innerHTML='<p class="muted">Ancora nessuna scadenza. Aggiungine una qui sotto!</p>';return;}
 list.innerHTML="";
 data.forEach(r=>{
  const div=document.createElement("div");
  div.className="item";
  const name=document.createElement("strong");
  name.textContent=r.title;
  const detail=document.createElement("div");
  detail.className="muted";
  detail.textContent=r.category+" · "+r.due_date+(r.amount?" · € "+r.amount:"");
  const del=document.createElement("button");
  del.className="secondary";
  del.textContent="Elimina";
  del.style.marginTop="8px";
  del.onclick=()=>deleteReminder(r.id);
  div.append(name,detail,del);
  list.appendChild(div);
 });
}

async function addReminder(){
 const {data:{user}}=await db.auth.getUser();
 if(!user)return;
 const titleValue=document.getElementById("title").value.trim();
 const dateValue=document.getElementById("date").value;
 const msg=document.getElementById("saveMessage");
 if(!titleValue||!dateValue){msg.textContent="Inserisci titolo e data.";return;}
 const amountValue=document.getElementById("amount").value;
 const {error}=await db.from("reminders").insert({
  user_id:user.id,
  title:titleValue,
  category:document.getElementById("category").value,
  due_date:dateValue,
  amount:amountValue?Number(amountValue):null
 });
 if(error){msg.textContent="Errore: "+error.message;return;}
 document.getElementById("title").value="";
 document.getElementById("date").value="";
 document.getElementById("amount").value="";
 msg.textContent="Scadenza salvata!";
 await loadReminders();
}

async function deleteReminder(id){
 const {error}=await db.from("reminders").delete().eq("id",id);
 if(error){alert(error.message);return;}
 await loadReminders();
}

db.auth.onAuthStateChange((event,session)=>{
 if(session) refresh();
 else showAuth();
});
refresh();
</script>
</body>
</html>
