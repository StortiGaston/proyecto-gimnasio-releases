Release- Descarga de .exe
Proyecto-Gimnasio
App de escritorio liviana para gestionar gimnasios en Argentina. Funciona sin conexión y sincroniza con la nube cuando hay internet.

En recepción se escanea el DNI del socio (lector 2D del código PDF417) o se tipea el número. Al instante se ve su ficha con el estado de la cuota y queda registrado el ingreso con fecha y hora, haya o no internet.

Es multi-gimnasio: una sola app y una sola base en la nube atienden a muchos gimnasios, y cada uno ve únicamente sus datos.

Funcionalidades
Módulo	Quién
Recepción: DNI → ficha con semáforo (al día / por vencer / vencida / sin cuota paga), ingreso con hora, un ingreso por socio por día, control de sucursal, lista de ingresos del día	Admin y profesor
Alta de socio: autocompletado con el DNI escaneado, teléfono obligatorio, primera cuota obligatoria, sucursal	Admin y profesor
Cobrar cuota: búsqueda por DNI o apellido, vencimiento el mismo día del mes siguiente, "mis cobros de hoy" con totales por método	Admin y profesor (con restricciones)
Socios: búsqueda y filtros, edición, cambio de vencimiento y de sucursal, baja y reactivación, baja automática de quien no paga hace N meses, eliminar (solo sin cobros), historial de cambios y pagos, anular pagos	Admin (el profesor ve una lista de solo lectura)
Planes: alta, edición, precios, duración en meses o días, plan de una sucursal o pase para varias	Admin
Caja: cierre diario por método y por usuario, gastos y otros ingresos, reporte por período, por sucursal, CSV para Excel	Admin
Avisos: cuotas y aptos médicos por vencer o vencidos, marcar leídos, cobrar desde el aviso	Admin
Usuarios y configuración: miembros del gimnasio con su rol y sucursal, días de aviso y de gracia, bloqueo de ingreso, baja automática, sucursales	Admin
Panel de gimnasios: alta con su primer admin, suspensión, usuarios de cada gimnasio, eliminar gimnasios vacíos	Superadmin
Actualizaciones remotas: las PCs instaladas ofrecen "Actualizar ahora" cuando sale una versión nueva	Todos
Sucursales
Un gimnasio puede tener varias sedes con un solo padrón. Cada socio pertenece a una sucursal; un plan común entra solo en la suya y un pase entra en varias. Cada PC elige su sucursal, y los cobros, ingresos y la caja quedan registrados por sucursal.

Roles
Admin del gimnasio: acceso completo a los datos de su gimnasio.
Profesor: registra ingresos, da de alta socios y cobra el plan actual del socio a precio de lista. No puede cambiar planes, aplicar descuentos, editar socios ni ver la caja.
Superadmin (dueño del software): crea y suspende gimnasios. No ve socios, cobros ni asistencias de ningún gimnasio.
Stack
Capa	Tecnología
Escritorio	Tauri 2 (usa el WebView del sistema: instalador de pocos MB)
Interfaz	React 18 + TypeScript (estricto) + Vite
Estilos	Tailwind CSS 4, componentes propios
Base local	SQLite vía @tauri-apps/plugin-sql
Nube	Supabase: PostgreSQL, Auth, Storage, pg_cron
Validación	Zod
Tests	Vitest (con node:sqlite para probar la base local)
Presupuesto de peso (versión 1.0, medido en Windows 10, i7-6700, 8 GB):

Límite	Medido
Instalador	< 15 MB	2,6 MB
RAM en uso	< 150 MB	< 100 MB
Arranque hasta poder escanear	< 2 s	< 2 s
JS de recepción (gzip)	< 300 KB	156 KB (el resto se carga al abrir cada pantalla)
Arquitectura
Offline-first
Toda acción de recepción (ingresos, altas, cobros) se guarda primero en SQLite en la PC y se considera hecha. La sincronización con Supabase corre en segundo plano: al iniciar, al volver la conexión y cada 60 segundos.

┌──────────────── PC de recepción ────────────────┐            ┌──────── Supabase ────────┐
│                                                 │            │                          │
│  Pantallas ──► SQLite local                     │   subida   │  Funciones PostgreSQL    │
│               ├─ cola_altas    ─────────────────┼──────────► │  ├─ alta_socio           │
│               ├─ cola_cobros   ─────────────────┼──────────► │  ├─ registrar_pago_cuota │
│               ├─ cola_ingresos ─────────────────┼──────────► │  └─ sincronizar_ingresos │
│               │                                 │            │                          │
│               ├─ padron  ◄──────────────────────┼─────────── │  padron_recepcion        │
│               ├─ planes  ◄──────────────────────┼─────────── │  datos_gimnasio          │
│               └─ gimnasio (configuración)       │   bajada   │                          │
└─────────────────────────────────────────────────┘            └──────────────────────────┘
Orden de subida obligatorio: altas → cobros → ingresos (en lotes de 500), para que un socio dado de alta sin conexión exista antes de que lleguen sus cobros e ingresos.
Idempotencia: cada operación lleva un UUID generado en la PC. Si se corta internet a mitad de un envío, reenviarlo no duplica nada.
Rechazos: lo que el servidor rechaza por una regla de negocio queda marcado "para revisión" en la PC; nunca se pierde.
Padrón incremental y paginado: solo baja lo que cambió desde la última sincronización.
Sesión sin internet: la app abre y opera aunque el token de Supabase haya vencido; se renueva sola al volver la conexión.
Indicador en la barra: 🟢 en línea · 🟡 sincronizando · 🔴 sin conexión (con pendientes).
Seguridad
La seguridad vive en la base de datos, no en la interfaz:

Row Level Security en todas las tablas, filtrando por gimnasio_id según la membresía del usuario.
El profesor no tiene acceso directo a ninguna tabla: opera solo por funciones (RPC) que validan su rol y sus restricciones del lado del servidor.
Claves foráneas compuestas (gimnasio_id, id) impiden mezclar datos de dos gimnasios.
La app usa solo la publishable key de Supabase. La secret key nunca se distribuye.
Todo cambio en socios, planes, cuotas, pagos y miembros queda en la tabla auditoria (quién, cuándo, antes y después), visible para el admin en el historial del socio.
Reglas replicadas
Las reglas que la app necesita evaluar sin conexión están escritas igual en SQL y en TypeScript, con tests que cubren los mismos casos:

Semáforo de cuota (v_estado_socios ↔ src/lib/estado.ts).
Vencimiento: el mismo día del mes siguiente, encadenado si la cuota sigue vigente, con el ajuste de fin de mes de PostgreSQL (registrar_pago_cuota ↔ calcularRenovacion).
Normalización del teléfono (normalizar_telefono ↔ src/lib/telefono.ts).
Puesta en marcha
Requisitos
Node.js 22.5 o superior (los tests usan node:sqlite).
Rust (para compilar la app de escritorio).
Una cuenta en Supabase.
1. Supabase
Crear un proyecto (región São Paulo, la más cercana a Argentina).
Authentication → Sign In / Providers: dejar Email activo y desactivar Allow new users to sign up (las cuentas las crea el administrador).
Authentication → Users → Add user (con Auto Confirm User): tu cuenta y la del admin de cada gimnasio.
SQL Editor: ejecutar en orden el contenido de cada archivo de supabase/migrations/.
Database → Extensions: activar pg_cron y ejecutar el paso A del final de la migración inicial (todos los días: avisos y bajas automáticas). Para verificarlo: select jobname, schedule from cron.job;
SQL Editor: paso B (hacerte superadmin) del final de la migración inicial. Los gimnasios se crean después desde la app (pantalla Gimnasios).
Edge Functions → Deploy a new function → Via Editor: crear crear-usuario con el contenido de supabase/functions/crear-usuario/index.ts (alta de usuarios y gimnasios).
2. Variables de entorno
cp .env.example .env
Completar con los datos de Project Settings → API Keys (o el botón Connect):

VITE_SUPABASE_URL=https://<tu-proyecto>.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=sb_publishable_...
⚠️ Nunca uses la secret key ni la service_role en este archivo: el .env se empaqueta dentro de la app. .env está en .gitignore.

3. Instalar y ejecutar
npm install
npm run tauri dev
Instalar en una PC de recepción (Windows)
Descargar el archivo …_x64-setup.exe de la última versión en proyecto-gimnasio-releases.
Ejecutarlo. Windows muestra "Windows protegió su PC" porque el instalador no tiene firma de código: Más información → Ejecutar de todas formas (solo la primera vez).
Iniciar sesión con la cuenta que creó el admin del gimnasio. Si el gimnasio tiene varias sucursales, elegir la de esa PC.
Requiere Windows 10 u 11 con WebView2 (si falta, el instalador lo descarga).

Publicar una versión nueva
npm version 0.9.1          # cambia la versión, hace commit y crea la etiqueta v0.9.1
git push --follow-tags     # GitHub Actions compila, firma y publica el instalador
El workflow .github/workflows/publicar.yml corre los tests, compila el instalador de Windows, lo firma para el actualizador y lo publica con su latest.json en el repo público de releases (el código queda en este repo privado). Las PCs instaladas lo detectan al abrir la app o cada 6 horas y muestran "Actualizar ahora"; nada de lo pendiente se pierde.

Secrets del repo: TAURI_SIGNING_PRIVATE_KEY, RELEASES_TOKEN, VITE_SUPABASE_URL, VITE_SUPABASE_PUBLISHABLE_KEY.
La clave privada del actualizador (~/.tauri/proyecto-gimnasio.key) no se sube al repo y hay que tener una copia de respaldo: sin ella, las PCs instaladas no pueden actualizarse.
Los cambios de esquema tienen que seguir sirviendo a la versión anterior de la app (puede haber PCs que tarden en actualizar): agregar columnas y funciones, no borrar ni renombrar.
Backups
El plan Free de Supabase no guarda copias. El workflow .github/workflows/backup.yml vuelca la base todos los domingos a las 03:00 (roles, esquema y datos, con la CLI de Supabase), la cifra con AES256 y la guarda 90 días como artifact de este repo privado. También se lanza a mano: Actions → Backup semanal → Run workflow.

Secrets: SUPABASE_DB_URL (Connect → Session pooler → URI, con la contraseña de la base) y BACKUP_PASSPHRASE (guardala aparte: sin ella los backups no se pueden abrir).

Bajar y descifrar el último (pide la contraseña; necesita gpg: brew install gnupg):

scripts/bajar-backup.sh      # deja backup-restaurado/backup/{roles,schema,data}.sql
Restaurar en un proyecto de Supabase nuevo (necesita psql: brew install libpq):

psql --single-transaction --variable ON_ERROR_STOP=1 \
  --file backup-restaurado/backup/roles.sql \
  --file backup-restaurado/backup/schema.sql \
  --command 'SET session_replication_role = replica' \
  --file backup-restaurado/backup/data.sql \
  --dbname "postgresql://postgres.<ref>:<contraseña>@<host-del-pooler>:5432/postgres"
Después: volver a publicar las Edge Functions, programar pg_cron (paso A) y cambiar la URL y la publishable key en los secrets del repo y en el .env. backup-restaurado/ tiene datos personales: no subirlo nunca al repo (está en .gitignore) y borrarlo al terminar.

Comandos
npm run tauri dev     # app de escritorio en desarrollo
npm run dev           # solo la interfaz en el navegador (sin base local: recepción no funciona)
npm test              # tests (Vitest)
npm run typecheck     # chequeo de tipos de la app, la configuración y los tests
npm run tauri build   # instalador
Después de un cambio de esquema, regenerar los tipos de la base:

npx supabase login
npx supabase gen types typescript --project-id <ID> > src/lib/database.types.ts
Estructura
├── .github/workflows/       # publicación de versiones (instalador de Windows)
├── supabase/
│   ├── migrations/          # esquema, RLS, funciones (siempre con migraciones nuevas)
│   └── functions/           # Edge Functions (crear-usuario)
├── src-tauri/               # app de escritorio (Rust): plugins SQL, actualizador
└── src/
    ├── auth/                # sesión, gimnasio activo, roles, login, selector
    ├── sync/                # motor de sincronización, conexión, indicador
    ├── lib/                 # base local, reglas replicadas, DNI, teléfono, formatos, errores
    ├── features/
    │   ├── recepcion/       # ingreso por DNI
    │   ├── socios/          # alta (todos) y gestión con historial (admin)
    │   ├── cobros/          # cobro de cuotas y "mis cobros de hoy"
    │   └── …                # planes, caja, avisos, usuarios, superadmin
    └── components/          # componentes propios simples
CLAUDE.md contiene la guía completa del proyecto: reglas de negocio, permisos, convenciones y el plan de trabajo por etapas.

Tests
142 tests en 17 archivos. Entre ellos:

Motor de sincronización con SQLite en memoria y un servidor simulado: orden de subida, corte de internet a mitad de 1200 ingresos sin pérdidas ni duplicados, rechazos para revisión, aislamiento entre gimnasios, padrón paginado.
Semáforo y vencimientos con los mismos casos que el SQL (días de gracia, fin de mes, años bisiestos, hora argentina).
Lectura del DNI: tarjeta actual, formato anterior y lectores con otra distribución de teclado (datos inventados).
Reglas de cobro por rol, recepción (sucursales y pases), planes, configuración e historial de cambios.
Las migraciones SQL se prueban aplicándolas en orden sobre PostgreSQL (PGlite) con dos gimnasios: aislamiento, roles, idempotencia y las reglas de cada función.

Datos personales
La app maneja datos personales y de salud (Ley 25.326). Pide solo lo necesario, no los escribe en logs ni los envía a servicios externos. Cada gimnasio es responsable de los datos de sus socios y el superadmin no accede a ellos.

Licencia
Proyecto privado. Todos los derechos reservados.
