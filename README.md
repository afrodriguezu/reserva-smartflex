# Reserva automática Smart Flex

Bot de Telegram que reserva tus clases del curso de inglés. Le escribes a
cualquier hora y reserva en ese momento. A las 9:20 p.m., si no has programado
la de mañana, te pregunta.

Corre en la nube, gratis, sin dejar el computador encendido.

---

## Cómo se usa

| Le escribes | Pasa |
|---|---|
| `reservar` | te pregunta, mostrando las horas que de verdad hay libres |
| `19:00` | reserva a esa hora para el próximo día hábil |
| `clase 12 8pm` | clase y hora específicas |
| `lunes 7pm` | para un día concreto |
| `clase 12` / `lunes` | sin hora: te muestra las horas libres de ese día y espera |
| `entra el domingo 00:15 19:00` | deja una cita: entra ese día y reserva a esa hora |
| `cancelar` | cancela tu reserva activa |
| `estado` | qué tienes reservado y qué citas puestas |
| `hola` | el menú |

---

## Por qué no reserva a medianoche

Parece lo obvio: los cupos abren a las 00:15, así que reservar ahí. No funciona.

El sistema **no deja programar una clase nueva mientras tengas otra sin ver**.
Si tus clases son de 7 a 9 p.m., a medianoche siempre tienes una pendiente, así
que el intento se rechaza siempre.

El momento útil es justo después de terminar la clase, cuando esa reserva pasa a
ser pasado y libera el cupo. De ahí el recordatorio de las 9:20 p.m.

Para los días en que no tienes clase (domingos, festivos), existen las **citas**:
le dices el sábado `entra el domingo 00:15 19:00` y entra solo a esa hora, cuando
abren los cupos del lunes.

La segunda hora es obligatoria: es la de la clase. A las 00:15 estás dormido, así
que el bot no elige por ti. Si no la escribes, no pone la cita.

---

## Los archivos

```
reserva-smartflex/
├── reservar_smartflex.py    la lógica completa
├── config.json              tus preferencias (sin datos personales)
├── estado.json              la memoria del bot, se actualiza sola
├── instalar.sh              automatiza el montaje del repositorio
├── RELOJ-EXTERNO.md         el reloj en cron-job.org — paso obligatorio
└── .github/workflows/
    ├── escuchar.yml         atiende el chat y reserva
    └── recordar.yml         el recordatorio de las 9:20 p.m.
```

Los workflows **no tienen `schedule:`**. Solo corren cuando el reloj externo los
llama. Es a propósito: ver `RELOJ-EXTERNO.md`.

---

## Montaje

### 1. El bot de Telegram

1. En Telegram busca **@BotFather** → `/newbot` → te da un **token**.
2. Abre tu bot y mándale `hola`. Sin ese primer mensaje no puede escribirte.
3. Entra a `https://api.telegram.org/bot<TOKEN>/getUpdates` y busca
   `"chat":{"id":123456789` — ese número es tu **chat id**.

### 2. GitHub CLI

**Windows** (PowerShell):

```powershell
winget install --id GitHub.cli
```

Cierra y vuelve a abrir PowerShell para que quede en el PATH. No hay que tocar
ningún perfil.

**macOS**, si no tienes Homebrew, descarga el binario:

```bash
ARCH=$([ "$(uname -m)" = "arm64" ] && echo arm64 || echo amd64)
curl -fsSL -o /tmp/gh.zip "https://github.com/cli/cli/releases/download/v2.98.0/gh_2.98.0_macOS_${ARCH}.zip"
unzip -oq /tmp/gh.zip -d ~/.local
echo 'export PATH="$HOME/.local/gh_2.98.0_macOS_'"$ARCH"'/bin:$PATH"' >> ~/.zshrc
```

En `gh auth login` responde **HTTPS** y autentica por navegador.

### 3. El repositorio

`instalar.sh` automatiza esto, pero necesita bash (macOS, Linux o Git Bash).
En PowerShell se hace a mano con los mismos comandos de `gh`.
Desde adentro de esta carpeta:

```bash
bash instalar.sh
```

Crea el repositorio público, sube los archivos, activa permisos de escritura y
te pide los secrets. Los escribes tú, con el texto oculto.

| Secret | Valor |
|---|---|
| `SMARTFLEX_DOC` | tu número de documento |
| `SMARTFLEX_EMAIL` | tu correo |
| `TELEGRAM_TOKEN` | el token de BotFather |
| `TELEGRAM_CHAT_ID` | tu chat id |
| `SMARTFLEX_DEVICE_ID` | lo genera solo |

### 4. Tus preferencias

En `config.json` ajusta `subnivel` y `clase`. El resto déjalo.

### 5. El reloj externo — obligatorio

Sin esto nada corre. Sigue `RELOJ-EXTERNO.md`.

### 6. Probar

```bash
gh workflow run escuchar.yml
```

Escríbele `hola` al bot antes de lanzarlo. Debe contestarte el menú.

---

## Detalles del sistema del curso

- **Una reserva activa a la vez.** No puedes programar si tienes otra sin ver.
- **Los cupos de un día abren la madrugada anterior**, sobre las 00:15.
- **El sistema reporta como activas las clases ya dictadas.** El código las
  filtra por fecha para que no te bloqueen.
- **Nunca canceles una clase que ya viste.** Podría quedar como no asistida. El
  bot se niega a hacerlo.
- **Domingos y festivos colombianos no hay clases.** El bot los calcula con la
  Ley Emiliani y te avisa cuál es el siguiente día hábil.

---

## Nunca reserva con una hora que no dijiste

No hay hora por defecto. Si el mensaje no trae hora, el bot consulta los cupos
reales de ese día, te los muestra como botones y espera. `hora` en `config.json`
solo se usa como sugerencia cuando el sistema del curso no responde, y
`horas_sugeridas` para armar los botones de una cita.

Cuando tocas una hora del menú, reserva la clase y el día que ese mismo menú
mostró, no lo que recalcule después. Esa propuesta vive 12 horas en `estado.json`.

---

## Cómo cuenta las clases

El contador se ancla en la última clase que **ya se dictó**, no en la última
que reservaste. Un booking solo cuenta cuando su hora de inicio más
`duracion_clase_horas` (2 por defecto) ya pasó.

Consecuencias:

- Cancelar una reserva no corre el contador: esa clase nunca se dictó.
- Una clase en curso tampoco cuenta todavía.
- Si cambias de subnivel, el contador se reinicia con la primera clase dictada
  del subnivel nuevo.

`estado.json` lo guarda en `ultima_clase_vista`. `ultima_clase_reservada` sigue
ahí solo como respaldo para estados de la versión anterior.

Si aun así se desalinea, `estado` te dice qué tiene registrado, y una reserva
explícita (`clase 11 19:00`) lo vuelve a alinear en cuanto esa clase se dicte.

---

## Privacidad

El repositorio es público para que Actions sea gratis. No hay nada personal en
los archivos: documento, correo y token viven en los secrets, invisibles incluso
para quien clone el repositorio.

Pero **los logs de Actions de un repositorio público los lee cualquiera**. Por
eso el script no escribe el contenido de tus mensajes. Si necesitas depurar,
agrega `LOG_DETALLADO: "1"` al workflow, míralo, y quítalo.

El bot ignora cualquier mensaje que no venga de tu `chat_id`.

---

## Cuando algo falle

```bash
gh run list --limit 10
gh run view <ID> --log | grep reservar_smartflex
```

Ese `grep` filtra solo lo que imprimió el script. Sin él ves cientos de líneas
de limpieza que no dicen nada.

Revisa también que el workflow le esté pasando los secrets al script: en el log,
bajo `env:`, deben aparecer los cinco. Si falta alguno, el `escuchar.yml` está
incompleto.
