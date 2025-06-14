Intro Dinámica: Crearemos una pantalla de bienvenida que aparecerá al cargar la página. Tendrá un efecto de "texto tecleado" y un botón para entrar a la agenda. Una vez se entra, la intro se desvanecerá para mostrar el contenido principal.
Fechas de Creación y Modificación:

Cada vez que se cree una nueva entrada, se guardará la fecha y hora de creación.

Cada vez que se actualice una entrada, se registrará la fecha y hora de la última modificación.

Estas fechas se mostrarán en la tabla, dándole un contexto temporal a cada registro.

Enviar por Correo (Mailto Link):
Añadiremos un botón "Enviar por Correo".
Al hacer clic, se generará un enlace mailto:. Este enlace abrirá la aplicación de correo electrónico predeterminada del usuario (Outlook, Gmail, Apple Mail, etc.) con el asunto y el cuerpo del correo ya rellenados con los datos de la agenda.
Importante: Este método depende de que el usuario tenga un cliente de correo configurado. No envía el correo directamente desde la página web (lo cual requeriría un servidor).
Código Completo (index.html)

Aquí está el código actualizado. Guarda todo en un solo archivo index.html y ábrelo en tu navegador.

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Agenda Dinámica v2</title>
    
    <!-- Librerías para Exportación (via CDN) -->
    <script src="https://unpkg.com/jspdf@2.5.1/dist/jspdf.umd.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.23/jspdf.plugin.autotable.min.js"></script>
    <script src="https://unpkg.com/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
    
    <style>
        /* --- Paleta de Colores Retro-Neón --- */
        :root {
            --bg-color: #1a0a24;
            --primary-glow: #d845f0;
            --secondary-glow: #3ad9f2;
            --text-color: #e0e0e0;
            --input-bg: rgba(0, 0, 0, 0.4);
            --border-color: rgba(216, 69, 240, 0.5);
            --header-bg: rgba(58, 217, 242, 0.1);
            --button-bg: transparent;
            --button-hover-bg: rgba(216, 69, 240, 0.2);
            --scrollbar-thumb: var(--primary-glow);
            --scrollbar-track: #333;
        }

        /* --- Animaciones --- */
        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
        @keyframes fadeOut { from { opacity: 1; } to { opacity: 0; } }
        @keyframes slideInDown { from { opacity: 0; transform: translateY(-30px); } to { opacity: 1; transform: translateY(0); } }
        @keyframes fadeOutUp { from { opacity: 1; } to { opacity: 0; transform: translateY(-20px); } }
        @keyframes neonGlow {
            from { text-shadow: 0 0 5px #fff, 0 0 10px #fff, 0 0 15px var(--secondary-glow); }
            to { text-shadow: 0 0 10px #fff, 0 0 20px #fff, 0 0 30px var(--secondary-glow), 0 0 40px var(--secondary-glow); }
        }
        @keyframes typing { from { width: 0; } to { width: 100%; } }
        @keyframes blink-caret { from, to { border-color: transparent; } 50% { border-color: var(--secondary-glow); } }

        body {
            font-family: 'Consolas', 'Menlo', 'monospace';
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 0; /* Sin padding en el body para la intro a pantalla completa */
            overflow: hidden; /* Evitar scroll durante la intro */
        }
        
        /* --- INTRO DINÁMICA --- */
        #intro-screen {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: var(--bg-color);
            display: flex;
            justify-content: center;
            align-items: center;
            z-index: 100;
            flex-direction: column;
            transition: opacity 1s ease-out;
        }
        #intro-screen.hidden {
            opacity: 0;
            pointer-events: none;
        }
        .intro-text {
            font-size: 2.5em;
            color: var(--secondary-glow);
            overflow: hidden;
            border-right: .15em solid var(--secondary-glow);
            white-space: nowrap;
            margin: 0 auto;
            letter-spacing: .10em;
            animation: typing 3.5s steps(40, end), blink-caret .75s step-end infinite;
        }
        #enter-btn {
            margin-top: 40px;
            padding: 15px 40px;
            font-size: 1.2em;
            background: transparent;
            color: var(--primary-glow);
            border: 2px solid var(--primary-glow);
            border-radius: 8px;
            cursor: pointer;
            opacity: 0;
            animation: fadeIn 1s ease-in 4s forwards; /* Aparece después de que el texto termine de escribirse */
            transition: all 0.3s ease;
        }
        #enter-btn:hover {
            background: var(--button-hover-bg);
            box-shadow: 0 0 20px var(--primary-glow);
            transform: scale(1.05);
        }

        /* --- Fondo Dinámico Canvas --- */
        #dynamic-bg {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            z-index: -1;
        }

        .main-content {
            padding: 20px;
            opacity: 0; /* Oculto inicialmente */
            transition: opacity 1s ease-in;
        }
        .main-content.visible {
            opacity: 1;
        }

        .container {
            position: relative;
            z-index: 1;
            max-width: 1600px; /* Aumentado para más columnas */
            margin: 0 auto;
            display: flex;
            flex-direction: column;
            gap: 25px;
        }

        .animated-section {
            background: rgba(10, 5, 15, 0.7);
            border: 1px solid var(--border-color);
            border-radius: 8px;
            padding: 25px;
            backdrop-filter: blur(5px);
            box-shadow: 0 0 15px rgba(216, 69, 240, 0.2);
        }

        h1, h2 { text-align: center; color: var(--secondary-glow); text-shadow: 0 0 8px var(--secondary-glow); margin-top: 0; }
        h1 { font-size: 2.5em; animation: neonGlow 1.5s ease-in-out infinite alternate; }

        /* --- Diseño del Formulario --- */
        #agendaForm { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; }
        .form-group { display: flex; flex-direction: column; }
        .form-group.full-width { grid-column: 1 / -1; }
        .form-group label { margin-bottom: 8px; color: var(--primary-glow); font-weight: bold; }
        .form-group input, .form-group textarea { background-color: var(--input-bg); color: var(--text-color); border: 1px solid var(--border-color); padding: 12px; font-family: inherit; border-radius: 5px; transition: all 0.3s ease; }
        .form-group input:focus, .form-group textarea:focus { outline: none; border-color: var(--secondary-glow); box-shadow: 0 0 10px var(--secondary-glow); }
        .form-group textarea { resize: vertical; min-height: 80px; }

        /* --- Diseño de Botones --- */
        .button-group, .export-section { display: flex; justify-content: center; flex-wrap: wrap; gap: 15px; margin-top: 20px; }
        .button-group button, .export-section button { background: var(--button-bg); color: var(--secondary-glow); border: 2px solid var(--secondary-glow); padding: 10px 25px; font-family: inherit; font-size: 1em; font-weight: bold; cursor: pointer; border-radius: 5px; transition: all 0.3s ease; text-transform: uppercase; }
        .button-group button:hover, .export-section button:hover { background: var(--button-hover-bg); color: #fff; border-color: var(--primary-glow); box-shadow: 0 0 15px var(--primary-glow); transform: translateY(-2px); }
        .button-group button:disabled { border-color: #555; color: #555; cursor: not-allowed; box-shadow: none; transform: none; }

        /* --- Diseño de la Tabla --- */
        .table-container { overflow: auto; max-height: 450px; }
        table { width: 100%; border-collapse: collapse; table-layout: fixed; }
        th, td { padding: 12px 15px; text-align: left; border-bottom: 1px solid var(--border-color); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
        th { background-color: var(--header-bg); color: var(--secondary-glow); font-weight: bold; position: sticky; top: 0; z-index: 2; }
        tbody tr { transition: background-color 0.3s ease, transform 0.2s ease; }
        tbody tr:hover { background-color: var(--button-hover-bg); transform: scale(1.01); cursor: pointer; }
        tbody tr.selected { background-color: rgba(58, 217, 242, 0.2); box-shadow: inset 0 0 10px var(--secondary-glow); }
        tr.new-entry { animation: slideInDown 0.5s ease-out forwards; }
        tr.deleting { animation: fadeOutUp 0.5s ease-out forwards; }

        /* --- Mensaje de Feedback --- */
        #feedback-message { position: fixed; bottom: -100px; left: 50%; transform: translateX(-50%); background-color: var(--secondary-glow); color: var(--bg-color); padding: 15px 25px; border-radius: 5px; box-shadow: 0 0 20px var(--secondary-glow); font-weight: bold; transition: bottom 0.5s ease-in-out; z-index: 1000; }
        #feedback-message.show { bottom: 20px; }
        
        /* Scrollbar */
        ::-webkit-scrollbar { width: 10px; }
        ::-webkit-scrollbar-track { background: var(--scrollbar-track); }
        ::-webkit-scrollbar-thumb { background: var(--scrollbar-thumb); border-radius: 5px; }
        ::-webkit-scrollbar-thumb:hover { background: var(--secondary-glow); }

    </style>
</head>
<body>

    <div id="intro-screen">
        <div class="intro-text">Cargando Agenda Dinámica...</div>
        <button id="enter-btn">Entrar</button>
    </div>

    <canvas id="dynamic-bg"></canvas>

    <div class="main-content">
        <div class="container">
            <h1>Agenda Dinámica</h1>
            <div class="animated-section">
                <form id="agendaForm">
                    <!-- ... campos del formulario ... -->
                    <div class="form-group"><label for="nombre">Nombre/Contenido</label><input type="text" id="nombre" required></div>
                    <div class="form-group"><label for="url">URL del Archivo</label><input type="text" id="url"></div>
                    <div class="form-group"><label for="password">Contraseña (¡Inseguro!)</label><input type="text" id="password"></div>
                    <div class="form-group"><label for="email">Email Asociado</label><input type="email" id="email"></div>
                    <div class="form-group"><label for="claveEspecial">Clave Especial</label><input type="text" id="claveEspecial"></div>
                    <div class="form-group"><label for="servicioAlmacenaje">Servicio Almacenaje</label><input type="text" id="servicioAlmacenaje"></div>
                    <div class="form-group full-width"><label for="notasExtra">Notas Extra</label><textarea id="notasExtra"></textarea></div>
                    <div class="button-group">
                        <button type="button" id="addBtn">✚ Añadir</button>
                        <button type="button" id="updateBtn" disabled>✔ Actualizar</button>
                        <button type="button" id="deleteBtn" disabled>✖ Eliminar</button>
                        <button type="button" id="clearBtn">⌫ Limpiar</button>
                    </div>
                </form>
            </div>
            <div class="animated-section">
                <h2>Mis Entradas</h2>
                <div class="table-container">
                    <table id="agendaTable">
                        <thead>
                            <tr>
                                <th>Nombre</th>
                                <th>URL</th>
                                <th>Contraseña</th>
                                <th>Email</th>
                                <th>Clave Especial</th>
                                <th>Servicio</th>
                                <th>Notas</th>
                                <th>Creado</th>
                                <th>Modificado</th>
                            </tr>
                        </thead>
                        <tbody></tbody>
                    </table>
                </div>
            </div>
            <div class="animated-section export-section">
                <button type="button" id="exportTxtBtn">Exportar a TXT</button>
                <button type="button" id="exportExcelBtn">Exportar a Excel</button>
                <button type="button" id="exportPdfBtn">Exportar a PDF</button>
                <button type="button" id="sendMailBtn">✉ Enviar por Correo</button>
            </div>
        </div>
    </div>

    <div id="feedback-message"></div>

    <script>
        // --- INICIO: LÓGICA DE LA INTRO ---
        const introScreen = document.getElementById('intro-screen');
        const enterBtn = document.getElementById('enter-btn');
        const mainContent = document.querySelector('.main-content');

        enterBtn.addEventListener('click', () => {
            introScreen.classList.add('hidden');
            mainContent.classList.add('visible');
            document.body.style.overflow = 'auto'; // Permitir scroll después de la intro
        });

        // --- INICIO: LÓGICA DEL FONDO DINÁMICO ---
        const canvas = document.getElementById('dynamic-bg');
        const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
        let particlesArray = [];
        const mouse = { x: null, y: null, radius: (canvas.height / 80) * (canvas.width / 80) };
        window.addEventListener('mousemove', e => { mouse.x = e.x; mouse.y = e.y; });
        class Particle {
            constructor(x, y, dX, dY, size, color) { this.x = x; this.y = y; this.directionX = dX; this.directionY = dY; this.size = size; this.color = color; }
            draw() { ctx.beginPath(); ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2); ctx.fillStyle = this.color; ctx.fill(); }
            update() {
                if (this.x > canvas.width || this.x < 0) this.directionX = -this.directionX;
                if (this.y > canvas.height || this.y < 0) this.directionY = -this.directionY;
                let dx = mouse.x - this.x; let dy = mouse.y - this.y; let distance = Math.sqrt(dx*dx + dy*dy);
                if (distance < mouse.radius + this.size) {
                    if (mouse.x < this.x && this.x < canvas.width - this.size * 10) this.x += 5;
                    if (mouse.x > this.x && this.x > this.size * 10) this.x -= 5;
                    if (mouse.y < this.y && this.y < canvas.height - this.size * 10) this.y += 5;
                    if (mouse.y > this.y && this.y > this.size * 10) this.y -= 5;
                }
                this.x += this.directionX; this.y += this.directionY; this.draw();
            }
        }
        function initParticles() {
            particlesArray = [];
            let num = (canvas.height * canvas.width) / 9000;
            for (let i = 0; i < num; i++) {
                let size = (Math.random() * 2) + 1; let x = (Math.random() * (innerWidth - size * 2)); let y = (Math.random() * (innerHeight - size * 2));
                let dX = (Math.random() * 0.4) - 0.2; let dY = (Math.random() * 0.4) - 0.2; let color = 'rgba(216, 69, 240, 0.5)';
                particlesArray.push(new Particle(x, y, dX, dY, size, color));
            }
        }
        function connectParticles() {
            for (let a = 0; a < particlesArray.length; a++) {
                for (let b = a; b < particlesArray.length; b++) {
                    let distance = ((particlesArray[a].x - particlesArray[b].x) ** 2) + ((particlesArray[a].y - particlesArray[b].y) ** 2);
                    if (distance < (canvas.width / 7) * (canvas.height / 7)) {
                        let opacity = 1 - (distance / 20000); ctx.strokeStyle = `rgba(58, 217, 242, ${opacity})`; ctx.lineWidth = 1;
                        ctx.beginPath(); ctx.moveTo(particlesArray[a].x, particlesArray[a].y); ctx.lineTo(particlesArray[b].x, particlesArray[b].y); ctx.stroke();
                    }
                }
            }
        }
        function animateParticles() { requestAnimationFrame(animateParticles); ctx.clearRect(0, 0, innerWidth, innerHeight); particlesArray.forEach(p => p.update()); connectParticles(); }
        window.addEventListener('resize', () => { canvas.width = innerWidth; canvas.height = innerHeight; mouse.radius = (canvas.height / 80) * (canvas.width / 80); initParticles(); });
        initParticles(); animateParticles();

        // --- INICIO: LÓGICA DE LA AGENDA ---
        const agendaForm = document.getElementById('agendaForm');
        const nombreInput = document.getElementById('nombre');
        const urlInput = document.getElementById('url');
        const passwordInput = document.getElementById('password');
        const notasExtraInput = document.getElementById('notasExtra');
        const emailInput = document.getElementById('email');
        const claveEspecialInput = document.getElementById('claveEspecial');
        const servicioAlmacenajeInput = document.getElementById('servicioAlmacenaje');
        
        const addBtn = document.getElementById('addBtn');
        const updateBtn = document.getElementById('updateBtn');
        const deleteBtn = document.getElementById('deleteBtn');
        const clearBtn = document.getElementById('clearBtn');
        
        const agendaTableBody = document.querySelector('#agendaTable tbody');
        const feedbackMessage = document.getElementById('feedback-message');
        
        let agendaEntries = [];
        let selectedEntryId = null;

        function showFeedback(message, duration = 3000) { feedbackMessage.textContent = message; feedbackMessage.classList.add('show'); setTimeout(() => feedbackMessage.classList.remove('show'), duration); }
        function loadEntries() { const stored = localStorage.getItem('agendaEntries'); agendaEntries = stored ? JSON.parse(stored) : []; renderTable(); }
        function saveEntries() { localStorage.setItem('agendaEntries', JSON.stringify(agendaEntries)); }

        const formatDate = (dateString) => dateString ? new Date(dateString).toLocaleString() : '---';

        function renderTable() {
            agendaTableBody.innerHTML = '';
            agendaEntries.forEach(entry => {
                const row = agendaTableBody.insertRow();
                row.dataset.id = entry.id;
                row.insertCell().textContent = entry.nombre;
                row.insertCell().textContent = entry.url;
                row.insertCell().textContent = entry.password;
                row.insertCell().textContent = entry.email;
                row.insertCell().textContent = entry.claveEspecial;
                row.insertCell().textContent = entry.servicioAlmacenaje;
                row.insertCell().textContent = entry.notas;
                row.insertCell().textContent = formatDate(entry.createdAt);
                row.insertCell().textContent = formatDate(entry.updatedAt);
                row.addEventListener('click', () => selectEntry(entry.id));
            });
            clearSelection();
        }

        function selectEntry(id) {
            if (selectedEntryId) { const prevRow = agendaTableBody.querySelector(`tr[data-id='${selectedEntryId}']`); if (prevRow) prevRow.classList.remove('selected'); }
            const entry = agendaEntries.find(e => e.id === id);
            if (entry) {
                selectedEntryId = id;
                nombreInput.value = entry.nombre;
                urlInput.value = entry.url;
                passwordInput.value = entry.password;
                notasExtraInput.value = entry.notas;
                emailInput.value = entry.email;
                claveEspecialInput.value = entry.claveEspecial;
                servicioAlmacenajeInput.value = entry.servicioAlmacenaje;
                const currentRow = agendaTableBody.querySelector(`tr[data-id='${id}']`); if (currentRow) currentRow.classList.add('selected');
                updateBtn.disabled = false; deleteBtn.disabled = false; addBtn.disabled = true;
            }
        }
        
        function clearFields() { agendaForm.reset(); }
        
        function clearSelection() {
            clearFields();
            if (selectedEntryId) { const prevRow = agendaTableBody.querySelector(`tr[data-id='${selectedEntryId}']`); if (prevRow) prevRow.classList.remove('selected'); }
            selectedEntryId = null; updateBtn.disabled = true; deleteBtn.disabled = true; addBtn.disabled = false;
        }

        function addEntry() {
            if (!nombreInput.value.trim()) { showFeedback('El campo "Nombre/Contenido" es obligatorio.'); return; }
            const now = new Date().toISOString();
            const newEntry = {
                id: Date.now(),
                nombre: nombreInput.value.trim(), url: urlInput.value.trim(), password: passwordInput.value.trim(),
                notas: notasExtraInput.value.trim(), email: emailInput.value.trim(), claveEspecial: claveEspecialInput.value.trim(),
                servicioAlmacenaje: servicioAlmacenajeInput.value.trim(),
                createdAt: now, updatedAt: now
            };
            agendaEntries.push(newEntry);
            saveEntries();
            renderTable();
            const newRow = agendaTableBody.querySelector(`tr[data-id='${newEntry.id}']`); if (newRow) { newRow.classList.add('new-entry'); setTimeout(() => newRow.classList.remove('new-entry'), 500); }
            showFeedback('Entrada añadida correctamente.');
        }

        function updateEntry() {
            if (selectedEntryId === null) return;
            const entryIndex = agendaEntries.findIndex(e => e.id === selectedEntryId); if (entryIndex === -1) return;
            const currentEntry = agendaEntries[entryIndex];
            agendaEntries[entryIndex] = {
                ...currentEntry,
                nombre: nombreInput.value.trim(), url: urlInput.value.trim(), password: passwordInput.value.trim(),
                notas: notasExtraInput.value.trim(), email: emailInput.value.trim(), claveEspecial: claveEspecialInput.value.trim(),
                servicioAlmacenaje: servicioAlmacenajeInput.value.trim(),
                updatedAt: new Date().toISOString()
            };
            saveEntries(); renderTable(); showFeedback('Entrada actualizada.');
        }

        function deleteEntry() {
            if (selectedEntryId === null || !confirm(`¿Eliminar la entrada "${nombreInput.value}"?`)) return;
            const rowToDelete = agendaTableBody.querySelector(`tr[data-id='${selectedEntryId}']`);
            if (rowToDelete) {
                rowToDelete.classList.add('deleting');
                rowToDelete.addEventListener('animationend', () => {
                    agendaEntries = agendaEntries.filter(e => e.id !== selectedEntryId); saveEntries(); renderTable(); showFeedback('Entrada eliminada.');
                }, { once: true });
            }
        }
        
        // --- Funciones de Exportación y Correo ---
        const getExportDataAsText = () => {
            let content = "--- DATOS DE LA AGENDA ---\n\n";
            agendaEntries.forEach((entry, index) => {
                content += `--- ENTRADA ${index + 1} ---\n`;
                content += `Nombre: ${entry.nombre}\n`;
                content += `URL: ${entry.url}\n`;
                content += `Contraseña: ${entry.password}\n`;
                content += `Email: ${entry.email}\n`;
                content += `Clave Especial: ${entry.claveEspecial}\n`;
                content += `Servicio: ${entry.servicioAlmacenaje}\n`;
                content += `Notas: ${entry.notas}\n`;
                content += `Creado: ${formatDate(entry.createdAt)}\n`;
                content += `Modificado: ${formatDate(entry.updatedAt)}\n\n`;
            });
            return content;
        };

        document.getElementById('exportTxtBtn').addEventListener('click', () => {
            const content = getExportDataAsText();
            const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a'); a.href = url; a.download = 'agenda.txt'; a.click();
            URL.revokeObjectURL(url); showFeedback('Exportado a TXT');
        });

        document.getElementById('exportExcelBtn').addEventListener('click', () => {
            const headers = ["Nombre", "URL", "Contraseña", "Email", "Clave", "Servicio", "Notas", "Creado", "Modificado"];
            const data = agendaEntries.map(e => [e.nombre, e.url, e.password, e.email, e.claveEspecial, e.servicioAlmacenaje, e.notas, formatDate(e.createdAt), formatDate(e.updatedAt)]);
            const ws = XLSX.utils.aoa_to_sheet([headers, ...data]);
            const wb = XLSX.utils.book_new();
            XLSX.utils.book_append_sheet(wb, ws, "Agenda");
            XLSX.writeFile(wb, "agenda.xlsx"); showFeedback('Exportado a Excel');
        });

        document.getElementById('exportPdfBtn').addEventListener('click', () => {
            const { jsPDF } = window.jspdf;
            const doc = new jsPDF('landscape');
            const headers = [["Nombre", "URL", "Password", "Email", "Clave", "Servicio", "Notas", "Creado", "Modificado"]];
            const data = agendaEntries.map(e => [e.nombre, e.url, e.password, e.email, e.claveEspecial, e.servicioAlmacenaje, e.notas, formatDate(e.createdAt), formatDate(e.updatedAt)]);
            doc.autoTable({ head: headers, body: data, theme: 'grid', styles: { fillColor: [26,10,36], textColor: [224,224,224] }, headStyles: { fillColor: [58,217,242,51], textColor: [58,217,242] } });
            doc.save('agenda.pdf'); showFeedback('Exportado a PDF');
        });
        
        document.getElementById('sendMailBtn').addEventListener('click', () => {
            if (agendaEntries.length === 0) {
                showFeedback("No hay datos para enviar por correo.");
                return;
            }
            const subject = "Datos de mi Agenda Dinámica";
            const body = getExportDataAsText();
            
            // Usamos encodeURIComponent para asegurar que el contenido se formatee correctamente en la URL
            const mailtoLink = `mailto:?subject=${encodeURIComponent(subject)}&body=${encodeURIComponent(body)}`;
            
            // Abrimos el cliente de correo
            window.location.href = mailtoLink;
        });

        // --- Event Listeners ---
        document.addEventListener('DOMContentLoaded', loadEntries);
        addBtn.addEventListener('click', addEntry);
        updateBtn.addEventListener('click', updateEntry);
        deleteBtn.addEventListener('click', deleteEntry);
        clearBtn.addEventListener('click', clearSelection);

    </script>

</body>
</html>

Resumen de las Mejoras y Cómo Funcionan

Intro Dinámica (#intro-screen):
CSS: Se ha creado un div a pantalla completa con un z-index alto para cubrir todo. El texto usa animaciones typing y blink-caret para simular ser escrito por una máquina. El botón de "Entrar" aparece con un fade-in después de que el texto termina.
JavaScript: Al hacer clic en el botón, se añade la clase .hidden a la intro, que la desvanece con una transición de opacidad. Al mismo tiempo, se hace visible el contenido principal (.main-content) y se permite el scroll en la página.

Fechas de Creación/Modificación:
Lógica de Añadir: Al crear una entrada, se captura la fecha actual con new Date().toISOString() y se guarda en los campos createdAt y updatedAt.
Lógica de Actualizar: Al actualizar, se mantiene el createdAt original y solo se actualiza el campo updatedAt con la nueva fecha.
Renderizado: Una función formatDate convierte la fecha en formato ISO (que es ideal para almacenar) a un formato legible (toLocaleString()) para mostrarlo en la tabla. Las columnas "Creado" y "Modificado" se han añadido a la tabla y a las exportaciones.
Enviar por Correo (#sendMailBtn):
Generación de Contenido: Se ha creado una función getExportDataAsText() que formatea todos los datos de la agenda en un texto claro y legible, perfecto para el cuerpo de un correo.

Creación del Link mailto:
Se construye un string mailto:?subject=...&body=....
encodeURIComponent() es crucial aquí. Asegura que caracteres especiales como espacios, saltos de línea (\n), &, ?, etc., se conviertan a un formato que pueda ser parte de una URL sin romperla.
Activación: window.location.href = mailtoLink; le dice al navegador que "navegue" a este enlace especial, lo que activa el cliente de correo predeterminado del sistema operativo.
