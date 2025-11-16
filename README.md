<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Reservas Sala de Cine</title>
  <style>
    body{font-family:system-ui;margin:20px;background:#f5f5f9}
    h1{margin-bottom:10px}
    .panel{background:#fff;padding:14px;border-radius:10px;box-shadow:0 4px 14px rgba(0,0,0,0.1);margin-bottom:18px;position:relative}
    input,button{padding:8px;border-radius:6px}
    button{border:0;background:#2b6df6;color:#fff;cursor:pointer}
    .room-item{padding:10px;background:#eef2ff;margin:6px 0;border-radius:6px;cursor:pointer;position:relative}
    .back-btn{margin-bottom:12px}
    .grid{display:grid;gap:8px;margin-top:12px}
    .seat{padding:10px;text-align:center;border-radius:6px;background:#dfe8ff;cursor:pointer;user-select:none}
    .taken{background:#ffd1d1!important;color:#700;pointer-events:auto}
    .row-label{margin-top:10px;font-weight:600}
    #loginPanel{max-width:300px;margin:auto;margin-top:50px;padding:20px;background:#fff;border-radius:10px;box-shadow:0 4px 14px rgba(0,0,0,0.1);text-align:center}
    #loginPanel input{width:90%;margin-bottom:10px;}
  </style>
</head>
<body>
  <div id="loginPanel" class="panel">
    <h2>Iniciar sesión</h2>
    <input id="username" type="text" placeholder="Usuario" /><br>
    <input id="password" type="password" placeholder="Contraseña" /><br>
    <button id="loginBtn">Entrar</button>
  </div>

  <div id="home" class="panel" style="display:none">
    <h1>Salas de cine</h1>
    <strong>Crear nueva sala</strong><br><br>
    <label>Nombre de la sala / Película<br>
      <input id="roomName" type="text" />
    </label><br><br>
    <label>Número de filas<br>
      <input id="rowCount" type="number" min="1" max="20" value="5" />
    </label><br><br>
    <label>Asientos por fila<br>
      <input id="seatsPerRow" type="number" min="1" max="20" value="8" />
    </label><br><br>
    <label>Fecha y hora de la función<br>
      <input id="roomDateTime" type="datetime-local" />
    </label><br><br>
    <button id="createRoomBtn">Crear sala</button>

    <hr>
    <strong>Salas creadas</strong>
    <div id="roomList" style="margin-top:8px"></div>
  </div>

  <div id="roomView" class="panel" style="display:none"></div>

<script>
const KEY='cine_rooms_ticket';
let data = load();

function save(){ localStorage.setItem(KEY, JSON.stringify(data)); }
function load(){ try{ const s = localStorage.getItem(KEY); return s ? JSON.parse(s) : {rooms:[]} }catch(e){ return {rooms:[]} } }
function uid(len=10){ const c='0123456789abcdef'; let s=''; for(let i=0;i<len;i++) s += c[Math.floor(Math.random()*c.length)]; return s; }
function letter(n){ return String.fromCharCode(65 + n); }
function escapeHtml(str){ return str.replace(/[&<>\"']/g, function(char){ return {'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;','\'' :'&#39;'}[char]; }); }

const loginPanel = document.getElementById('loginPanel');
const loginBtn = document.getElementById('loginBtn');
const home = document.getElementById('home');
const roomList = document.getElementById('roomList');
const roomView = document.getElementById('roomView');
const createRoomBtn = document.getElementById('createRoomBtn');

loginBtn.addEventListener('click', ()=>{
  const user = document.getElementById('username').value;
  const pass = document.getElementById('password').value;
  if(user === 'Hugo' && pass === 'Hugo123'){
    loginPanel.style.display='none';
    home.style.display='block';
    renderList();
  } else {
    alert('Usuario o contraseña incorrectos');
  }
});

createRoomBtn.addEventListener('click', ()=>{
  const name = document.getElementById('roomName').value.trim();
  const rows = parseInt(document.getElementById('rowCount').value, 10);
  const per = parseInt(document.getElementById('seatsPerRow').value, 10);
  const dateTime = document.getElementById('roomDateTime').value;
  if(!name) return alert('Pon un nombre');
  if(!rows || rows < 1 || !per || per < 1) return alert('Datos inválidos');
  if(!dateTime) return alert('Selecciona fecha y hora');
  data.rooms.push({ id: uid(10), name, rows, seatsPerRow: per, seats: {}, datetime: dateTime });
  save();
  renderList();
  document.getElementById('roomName').value='';
  document.getElementById('roomDateTime').value='';
});

function removePastRooms(){
  const now = new Date();
  let changed = false;
  data.rooms = data.rooms.filter(r => {
    if(new Date(r.datetime) <= now){ changed = true; return false; }
    return true;
  });
  if(changed) save();
}

function openRoom(roomId){
  const room = data.rooms.find(r => r.id === roomId);
  if(!room) return alert('Sala no encontrada');
  home.style.display='none';
  roomView.style.display='block';
  roomView.innerHTML='';

  const backBtn = document.createElement('button');
  backBtn.className='back-btn';
  backBtn.textContent='Volver';
  backBtn.addEventListener('click', backToHome);
  roomView.appendChild(backBtn);

  const title = document.createElement('h2');
  title.textContent=room.name;
  roomView.appendChild(title);

  for(let i=0;i<room.rows;i++){
    const rowLabel=document.createElement('div');
    rowLabel.className='row-label';
    rowLabel.textContent='Fila '+(i+1);
    roomView.appendChild(rowLabel);
    const grid=document.createElement('div');
    grid.className='grid';
    grid.style.gridTemplateColumns=`repeat(${room.seatsPerRow},1fr)`;
    for(let j=0;j<room.seatsPerRow;j++){
      const seat=document.createElement('div');
      seat.className='seat';
      const seatKey=`${i}-${j}`;
      seat.textContent=letter(j);
      if(room.seats[seatKey] && room.seats[seatKey].code) seat.classList.add('taken');
      seat.onclick = function(){ reserveSeatSingle(room, seatKey, seat); };
      grid.appendChild(seat);
    }
    roomView.appendChild(grid);
  }
}

function showTicket(name, room, seatKey, ticketCode){
  const ticketWindow = window.open('', '_blank');
  const htmlContent = `<!DOCTYPE html><html><head><title>Ticket</title><style>body{font-family:system-ui;padding:20px}h2{text-align:center}p{font-size:18px}</style></head><body>` +
                      `<h2>Ticket de reserva</h2>` +
                      `<p>Nombre: ${escapeHtml(name)}</p>` +
                      `<p>Sala: ${escapeHtml(room.name)}</p>` +
                      `<p>Fecha y hora: ${room.datetime}</p>` +
                      `<p>Asiento: ${letter(parseInt(seatKey.split('-')[1]))} Fila ${parseInt(seatKey.split('-')[0])+1}</p>` +
                      `<p>Código: ${ticketCode}</p>` +
                      `<br><button onclick='window.print()'>Imprimir Ticket</button>` +
                      `</body></html>`;
  ticketWindow.document.write(htmlContent);
  ticketWindow.document.close();
}

function reserveSeatSingle(room,seatKey,seat){
  if(room.seats[seatKey] && room.seats[seatKey].code){
    showTicket(room.seats[seatKey].name, room, seatKey, room.seats[seatKey].code);
    return;
  }
  const name = prompt('Ingrese su nombre para la reserva');
  if(!name) return alert('Nombre requerido');
  const ticketCode = uid(6).toUpperCase();
  room.seats[seatKey]={name, code:ticketCode};
  seat.classList.add('taken');
  save();
  showTicket(name, room, seatKey, ticketCode);
}

function backToHome(){
  roomView.style.display='none';
  home.style.display='block';
  renderList();
}

function renderList(){
  removePastRooms();
  roomList.innerHTML='';
  data.rooms.forEach(r=>{
    const item=document.createElement('div');
    item.className='room-item';
    item.innerHTML=`<strong>${escapeHtml(r.name)}</strong><br><small>Reservas: ${Object.keys(r.seats).length}<br>Fecha: ${r.datetime}</small>`;
    item.addEventListener('click',()=>openRoom(r.id));
    roomList.appendChild(item);
  });
}
</script>
</body>
</html>
