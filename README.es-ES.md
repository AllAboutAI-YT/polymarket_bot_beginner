

# polybot_beginner

Tu primer bot de trading en **Polymarket**. Opera en el mercado de 5 minutos "Bitcoin Arriba o Abajo" en Polygon e incluye un panel de control en tiempo real estilo terminal de Bloomberg para observar su funcionamiento.

La estrategia es **scalping de convergencia en la ventana final**: cuando el mercado ha elegido claramente un ganador en los últimos segundos de una ventana de 5 minutos, el bot toma la oferta del lado ganador para obtener una pequeña ventaja residual, y luego mantiene la posición hasta la resolución. El patrón fue desarrollado mediante ingeniería inversa a partir de un trader rentable en Polymarket: consulta [`research/bonereader_analysis.md`](research/bonereader_analysis.md).

Este repositorio está diseñado para aprendizaje. Las partes con dinero real se mantienen mínimas, cada acción riesgosa tiene un interruptor de emergencia y cada paso de configuración es un script de una sola línea.

---

## Orientación rápida: qué harás

```
1. install deps                       (5 min)
2. create a new account in MetaMask   (30 sec)
3. log into polymarket.com with it    (1 min)  -- se despliega la billetera de depósito
4. deposit USDC.e via Polymarket UI   (5 min)
5. export private key from MetaMask
   and import into the bot's .env     (30 sec)
6. derive API credentials             (10 sec)
7. verify setup                       (5 sec)
8. launch dashboard + bot             (10 sec)
```

De principio a fin: ~15 minutos si todo sale bien.

> **¿Quieres probarlo sin financiar una billetera?** Ejecuta
> `scripts/run_paper.sh` — el bot observa mercados en vivo y registra decisiones
> pero nunca coloca órdenes reales. Útil para aprender qué hace el bot
> antes de arriesgar dinero.

## Advertencias antes de comenzar

- **Polymarket tiene restricciones geográficas.** El CLOB no está disponible en muchas
  jurisdicciones — actualmente incluye EE. UU., Reino Unido, Francia, Alemania, Bélgica,
  Países Bajos, Portugal, Singapur, Polonia, además de regiones sancionadas por OFAC
  (la lista cambia; consulta
  [la lista de regiones de Polymarket](https://help.polymarket.com/en/articles/13364163-geographic-restrictions)).
  Si polymarket.com te bloquea, este bot tampoco funcionará.
  Usuarios de EE. UU.: existe un portal "Polymarket US" separado con KYC, pero opera en
  un exchange diferente que **este bot no apunta**.
  **No intentes hacerlo funcionar con una VPN** — Polymarket congela los fondos y no los
  devolverá si detecta el uso.
- **Esto es dinero real en una blockchain pública.** El trading arriesga lo que
  deposites en la billetera — comienza con $30 para aprender. El depósito mínimo es
  $3.
- **Solo billeteras completamente nuevas.** Esta guía asume que creas una cuenta
  nueva en MetaMask específicamente para el bot. Las cuentas de Polymarket anteriores a la V2
  (tipos Safe / Proxy) no obtienen una billetera de depósito y no funcionarán con
  este código.
- **La estrategia tal como se proporciona no es consistentemente rentable.** Lee la
  sección ["Expectativas"](#expectativas) antes de comprometer cualquier capital
  real. Este es un punto de partida que querrás ajustar.

---

## Prerrequisitos

| | versión | instalación |
|---|---|---|
| Python | 3.11+ | macOS: `brew install python@3.11` · Linux: `apt install python3.11 python3.11-venv` · Windows: [instalador de python.org](https://www.python.org/downloads/) |
| Node | 18+ | macOS: `brew install node` · Linux: [configuración de nodesource](https://github.com/nodesource/distributions) · Windows: [nodejs.org](https://nodejs.org/) |
| MetaMask | última | https://metamask.io/ (extensión de navegador) |

También necesitarás una forma de obtener **MATIC** y **USDC.e** en tu billetera en
la red Polygon. La ruta más fácil para principiantes:

1. Abre una cuenta en Coinbase, Kraken, Binance o cualquier exchange que
   soporte retiros a Polygon.
2. Compra MATIC y USDC. Mantén las cantidades bajas al inicio — ~$30 USDC, $1–2 MATIC.
3. Retira a la dirección de billetera que generarás en el paso 2 a continuación. **En el
   selector de red, elige Polygon (no Ethereum).**

Si ya tienes USDC.e en una billetera de otra cadena, puedes hacer el puente mediante
[app.polygon.technology](https://app.polygon.technology/).

> **USDC.e vs USDC.** Polymarket usa USDC.e (el USDC "puenteado" anterior).
> La mayoría de los exchanges envían el USDC "nativo" más reciente por defecto — asegúrate de que
> la red sea Polygon y que el símbolo del token sea **USDC.e** (o simplemente "USDC"
> cuando la red es Polygon — los exchanges a menudo lo etiquetan así).

---

## Configuración

### 1. Clonar e instalar

```bash
git clone https://github.com/<your-user>/polymarket_bot_beginner
cd polymarket_bot_beginner

# venv + dependencias de Python
python3.11 -m venv .venv
.venv/bin/pip install -r requirements.txt

# dependencias de UI
cd ui && npm install && cd ..
```

### 2. Crear una nueva cuenta en MetaMask

Harás la creación de la billetera, el registro en Polymarket **y** la
financiación todo dentro de MetaMask + polymarket.com — sin terminal por ahora.

¿No tienes MetaMask aún? Instala la extensión del navegador desde
**[metamask.io/download](https://metamask.io/download/)** (evita cualquier otra
fuente — las extensiones falsas de MetaMask son un estafa común). Configura una cuenta
nueva en MetaMask si aún no tienes una, y luego regresa aquí.

1. Abre MetaMask → menú de cuenta (círculo superior derecho) → **Añadir cuenta o
   billetera de hardware** → **+ Cuenta de Ethereum**.
2. Nómbrala con algo obvio como *polybot*.
3. Cambia a esa nueva cuenta en MetaMask.

> **No uses la pestaña integrada de Polymarket en MetaMask** si ves una — inicia
> sesión en polymarket.com en su lugar. El bot espera que la billetera de depósito sea
> aprovisionada a través del flujo del sitio web.

> **¿Por qué una cuenta dedicada?** Los bots de trading deberían ejecutarse con una billetera que
   no contenga nada más. Si el bot tiene un error o la clave se filtra después, solo
   el saldo pequeño del bot está en riesgo — no tu billetera principal.

### 3. Iniciar sesión en polymarket.com con la nueva cuenta

1. **Cierra sesión** de cualquier sesión existente en polymarket.com primero (avatar →
   Cerrar sesión) para que la cuenta incorrecta no se conecte automáticamente.
2. Abre polymarket.com → **Iniciar sesión** → elige **MetaMask**.
   Si aún no tienes una cuenta de Polymarket, regístrate a través de mi enlace de referido
   — ayuda a apoyar este proyecto sin costo para ti:
   [polymarket.com/?r=allaboutai](https://polymarket.com/?r=allaboutai).
3. **Verifica** que la ventana emergente de MetaMask muestre la cuenta *polybot* que acabas de
   crear, y luego firma el mensaje de autenticación.
4. Polymarket despliega automáticamente una **billetera de depósito** para ti (no requiere gas —
   ellos lo patrocina). Esta es la billetera de contrato inteligente desde la cual tu bot
   operará.

### 4. Depositar fondos a través de la interfaz de Polymarket

La ruta más fácil para principiantes — la interfaz de Polymarket maneja la plomería en cadena
(USDC.e → envoltura pUSD, enrutamiento de billetera de depósito) por ti.

1. En polymarket.com → avatar → **Depositar**.
2. Elige **Polygon · USDC.e** como activo de origen y sigue las
   instrucciones para enviar desde tu exchange. El mínimo es **$3**; comienza con
   **$30+** para tener suficiente margen para que la estrategia se active.
3. **En el formulario de retiro del exchange:**
   - **Dirección de destino:** la que Polymarket te mostró en la
     ventana modal de depósito.
   - **Red:** **Polygon** (no Ethereum, no BSC).
   - **Token:** USDC.e (a veces etiquetado simplemente como "USDC" en Polygon).
4. El depósito suele aparecer en Polymarket dentro de 1–3 minutos.
   Polymarket envuelve automáticamente tu USDC.e en pUSD y lo enruta a la
   billetera de depósito por ti — sin scripts necesarios.

> No necesitas **MATIC** en esta billetera. Polymarket patrocina el gas para las
   operaciones de la billetera de depósito. (Si alguna vez quieres *retirar* directamente
   en cadena necesitarás un poco, pero puedes conseguirlo después.)

### 5. Importar la billetera al bot

Ahora extrae la clave privada de MetaMask y entrégasela al bot. El bot
necesita la clave privada para firmar órdenes; MetaMask fue solo para la configuración.

1. En MetaMask, con la cuenta *polybot* seleccionada → menú **⋮** →
   **Detalles de la cuenta** → pestaña **Clave privada** → ingresa tu contraseña de MetaMask
   → mantén **presionado el botón "Mantener para mostrar la clave privada"**
   hasta que aparezca. Cópiala (comienza con `0x`, 64 caracteres hex).
2. Ejecuta:
   ```bash
   .venv/bin/python scripts/import_wallet.py
   ```
   El script te pedirá pegar la clave — la entrada está oculta, por lo que
   la clave no aparecerá en pantalla ni en el historial de tu shell. Presiona Enter para
   enviarla.

   Luego:
   - valida la clave, deriva la dirección,
   - **escanea el DepositWalletFactory en cadena** para encontrar la billetera de depósito
     que Polymarket desplegó para ti en el paso 3,
   - escribe `.env` con `SIGNATURE_TYPE=3` y `FUNDER_ADDRESS=<deposito>`.

> **Nota de seguridad:** una vez que `.env` tenga la clave, puedes eliminar
> la cuenta *polybot* de MetaMask si prefieres. El bot lee todo
> desde `.env`.

### 6. Derivar credenciales de API de L2

```bash
.venv/bin/python scripts/derive_api_creds.py
```

El bot usa esto para autenticar cada orden. Se almacena en `.env`.

### 7. Verificar toda la configuración

```bash
.venv/bin/python scripts/verify_setup.py
```

Recorre cada verificación (billetera, saldo, propiedad de la billetera de depósito,
autorizaciones, autenticación de API) e imprime `[ OK ]` / `[FAIL]` para cada una. Si algo
falla, el script te dice exactamente qué hacer.

---

### Ruta alternativa: solo terminal

Si prefieres no usar MetaMask en absoluto y quieres ejecutar todo desde
la terminal:

1. `scripts/generate_wallet.py` — crea una billetera aleatoria nueva en `.env`.
2. Financia el EOA con USDC.e **y** ~1 MATIC para gas.
3. Importa la misma clave en MetaMask **solo para registrarte en polymarket.com**
   (pasos 3 y 4 anteriores, pero usa *Importar cuenta* con la clave de `.env`).
4. `scripts/wrap_to_pusd.py` — envuelve tu USDC.e a pUSD.
5. `scripts/migrate_to_deposit_wallet.py 0xTU_DIRECCION_DEPOSITO` — mueve
   el pUSD a la billetera de depósito que obtuviste de polymarket.com.
6. Continúa desde el paso 6 anterior.

---

## Ejecución

### Dashboard

```bash
scripts/run_dashboard.sh
```

Abre FastAPI en el puerto 8787 y la UI de Vite en el puerto 5173. **Abre
http://127.0.0.1:5173 en tu navegador.**

| panel | muestra |
|---|---|
| Barra superior | estado del bot (RUNNING / STOPPED / LOCKED), salud de API, flash de última operación |
| Billetera / Patrimonio | efectivo en pUSD + valor de posición abierta + patrimonio total |
| G&P 24h | G&P realizado, conteo de ganancias/pérdidas, tasa de acierto |
| Estrategia | límites de entrada actuales + umbrales de riesgo |
| Mercado en Vivo | mercado activo de 5 min, cuenta regresiva, libro Arriba/Abajo (ask del ganador resaltado en ámbar cuando está en zona de compra) |
| Registro de Decisiones | cada decisión que toma el bot, transmisión en vivo |
| Posiciones Abiertas | tenencias actuales de CTF |
| Órdenes | resultados de órdenes recientes |

El panel funciona correctamente incluso cuando el bot no está ejecutándose — útil para
observar mercados antes de ir en vivo.

### Bot: simulado (paper) vs en vivo (live)

El bot tiene dos modos. **Comienza siempre en simulado (paper).** Cuando el panel muestre
decisiones verdes en mercados reales por un tiempo y entiendas lo que está
haciendo, cambia a en vivo.

```bash
# SIMULADO — sin órdenes reales, solo registra decisiones. Seguro.
scripts/run_paper.sh

# EN VIVO — coloca órdenes reales contra tu billetera de depósito financiada.
scripts/run_live.sh

# Detener cualquiera de los dos
scripts/stop_live.sh
```

Ambos modos usan el mismo script para detenerse. Solo un bot puede ejecutarse a la vez
(el lanzador se niega si `bot.pid` existe).

La barra superior del panel muestra el modo de forma prominente:

- `[ PAPER ]` en verde = seguro, sin órdenes reales
- `[ LIVE ]` en rojo = dinero real en juego
- `[ OFFLINE ]` en gris = bot no ejecutándose

El flash en pantalla de 5 segundos solo se activa en **rellenos reales**, nunca en simulado.

En el registro de decisiones y las tablas de órdenes, las entradas de simulado también están marcadas
internamente (`dry_run=1`) para que el G&P realizado solo cuente rellenos reales.

```bash
tail -f logs/bot_current.log    # observar el bot en tiempo real
```

### Detener todo

```bash
scripts/stop_live.sh
scripts/stop_dashboard.sh
```

---

## Ajustes

Todos los parámetros de estrategia están en `bot/config.py`:

```python
max_entry_price:      0.98     # solo activar cuando el ask del lado ganador <= esto
seconds_before_close: 35       # solo activar cuando t_remaining <= esto
min_t_remaining_sec:  8.0      # Y t_remaining >= esto (evitar carrera a la resolución)
order_size_shares:    5        # el mínimo es 5 por Polymarket
max_open_positions:   1
max_daily_loss_usd:   10000.0  # interruptor de pérdida diaria (10000 = efectivamente desactivado)
```

Y el umbral de convergencia en `bot/strategy.py`:

```python
LOSER_FLOOR = 0.85   # no activar a menos que el ask del lado ganador > esto
```

Conceptualmente: el bot solo entra cuando el mercado está convencido de un ganador
(`ask del ganador > LOSER_FLOOR`) pero aún no se ha convergido completamente
(`ask del ganador <= max_entry_price`). Más ajustado = menos operaciones con más margen;
más flojo = más operaciones con margen por operación más delgado.

Reinicia el bot después de los cambios:

```bash
scripts/stop_live.sh && scripts/run_live.sh
```

---

## Solución de problemas

**`maker address not allowed, please use the deposit wallet flow`**
Tu `SIGNATURE_TYPE` o `FUNDER_ADDRESS` no está configurado para la billetera de depósito V2. Vuelve a ejecutar `scripts/verify_setup.py` — te dirá qué paso
repetir.

**`balance: 0` desde `verify_setup.py` aunque la billetera de depósito
tenga pUSD en cadena**
La caché de API de Polymarket puede tener un retraso de un minuto. Espera 60s y vuelve a ejecutar. Si
persiste, tu `FUNDER_ADDRESS` en `.env` no coincide con la billetera de depósito
que conoce el CLOB — verifica en polymarket.com → Billetera para confirmar.

**`order couldn't be fully filled. FOK orders are fully filled or killed`**
No es un error — eso es el bot intentando tomar un ask que fue barrido por
alguien más primero. Normal en mercados rápidos. El bot simplemente pasa a la siguiente.

**Transacción atascada durante la configuración (`wrap_to_pusd.py` o
`migrate_to_deposit_wallet.py` se queda colgado)**
El gas de Polygon puede dispararse. Ejecuta `.venv/bin/python scripts/bump_stuck_tx.py` para
reemplazar la tx atascada con una versión de gas más alto.

**Errores `RPC 401 Unauthorized`**
El RPC público por defecto (`polygon-bor-rpc.publicnode.com`) a veces
aplica límites de tasa. Regístrate para una clave gratuita de Alchemy o QuickNode y reemplaza
`POLYGON_RPC_URL` en `.env`.

**El panel muestra `BOT LOCKED (LOSS_CAP)`**
La pérdida diaria excedió `max_daily_loss_usd` en `bot/config.py`. Espera
24 horas, aumenta el límite o reinicia con un `.env` nuevo para restablecer.

**Polymarket dice que mi país está bloqueado**
No puedes usar este bot. Polymarket aplica bloqueo geográfico a nivel de CLOB — no hay
solución alternativa que no viole los Términos de Servicio.

---

## Mapa de archivos

```
polybot_beginner/
├── .env                # secretos — ignorado por git, nunca hacer commit
├── .env.example        # referencia: qué script escribe cada campo
├── bot/                # motor de trading
│   ├── config.py       # todos los parámetros de estrategia
│   ├── markets.py      # descubrir el mercado BTC en vivo de 5 min
│   ├── book.py         # lector de libro de órdenes del CLOB
│   ├── strategy.py     # lógica de decisión
│   ├── orders.py       # envoltura SDK para colocación de órdenes FOK
│   ├── risk.py         # límites de pérdida diaria + posiciones abiertas
│   ├── store.py        # registrador SQLite (trades.db)
│   ├── resolver.py     # rellenar resoluciones para órdenes completadas
│   └── main.py         # bucle de eventos
├── server/dashboard.py # backend FastAPI (puerto 8787)
├── ui/                 # frontend Vite + React + TS (puerto 5173)
├── scripts/            # helpers de configuración + lanzamiento
│   ├── generate_wallet.py
│   ├── check_balance.py
│   ├── wrap_to_pusd.py
│   ├── bump_stuck_tx.py
│   ├── migrate_to_deposit_wallet.py
│   ├── derive_api_creds.py
│   ├── verify_setup.py
│   ├── run_live.sh / stop_live.sh
│   └── run_dashboard.sh / stop_dashboard.sh
├── research/           # análisis de estrategia + especificación de mercado (MDs + .py fetchers)
└── requirements.txt
```

---

## Expectativas

El mercado BTC de 5 minutos es competitivo. Los traders rentables aquí operan con
latencia subsegundo desde máquinas co-ubicadas con proveedores de RPC de pago e
infraestructura personalizada. Desde una laptop en un RPC público de Polygon, tus rellenos
serán más lentos y la ventaja en la ventana final es extremadamente delgada.

**La estrategia tal como se proporciona aquí perdió ~$10 en 54 rellenos durante una
ejecución nocturna durante el desarrollo.** La tasa de acierto fue del 89% pero el punto de equilibrio necesario
fue ~92% a nuestro precio de entrada promedio. La naturaleza del problema es
pagos asimétricos: muchas ganancias pequeñas (~$0.30) superadas por ocasionales
pérdidas de apuesta completa (~$4.50).

Cosas que puedes intentar para mejorarla (en dificultad aproximadamente creciente):

1. **Ajustar la zona de entrada.** Establece `LOSER_FLOOR` más alto (por ej. 0.93) para que
   el bot solo tome operaciones de muy alta confianza. Menos rellenos, mejor EV por operación.
2. **Añadir una puerta de precio spot de Binance.** Solo activar si BTC/USDT de Binance se mueve
   en la dirección favorable por más de ~5 bps dentro de la ventana. El
   esqueleto para esto vive en `bot/config.py` (`min_spot_offset_bps`).
3. **Mejor RPC.** Regístrate en Alchemy/QuickNode para ~50% menos latencia en
   lecturas.
4. **Reducir tamaño al entrar cerca de $0.98+.** Los pagos asimétricos significan apuesta menor
   en los niveles marginales.
5. **Ejecutar en modo sombra durante una semana.** Añade un modo `--dry-run` y recopila
   miles de decisiones de "habría entrado". Recalcula el G&P simulado
   con las tarifas actuales para ver si tus ajustes realmente ayudan antes de arriesgar
   capital.

Usa este código para aprender. No apuestes la casa.

---

## Soporte

Si este repositorio te ayudó, puedes registrarte en Polymarket a través de mi enlace de referido:
[polymarket.com/?r=allaboutai](https://polymarket.com/?r=allaboutai).
Sin obligación — simplemente me ayuda un poco. Gracias.
