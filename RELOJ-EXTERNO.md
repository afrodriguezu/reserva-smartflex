# El reloj externo — paso obligatorio

Los workflows de este proyecto **no tienen `schedule:`**. Solo corren cuando algo
externo los llama. No es un descuido.

## Por qué

El disparador `schedule` de GitHub Actions vive en una cola que se drena en modo
"mejor esfuerzo". Bajo carga se atrasa o se descarta. En pruebas reales:

- Una corrida programada a las 00:15 se ejecutó a las **11:29**.
- Un cron de `*/10` no disparó ni una sola vez en dos días.

`workflow_dispatch`, en cambio, se atiende casi al instante. Así que el reloj
real vive afuera y solo llama a esa API.

Se quitaron los `schedule:` por completo para que no haya corridas sorpresa a
horas impredecibles, que con las citas programadas sí pueden hacer daño.

---

## 1. El token de GitHub

Deja lanzar workflows en un solo repositorio, nada más.

1. **github.com → Settings → Developer settings → Personal access tokens →
   Fine-grained tokens → Generate new token**
2. Configúralo:
   - **Token name**: `reloj-smartflex`
   - **Expiration**: la máxima disponible
   - **Repository access**: *Only select repositories* → `reserva-smartflex`
   - **Permissions → Repository permissions → Actions**: `Read and write`
3. **Copia el token.** Solo se muestra una vez.

Si se filtrara, lo único que permite es lanzar estos workflows. No da acceso al
código ni a los secrets.

---

## 2. La cuenta del reloj

Regístrate en **cron-job.org** (gratis, sin tarjeta). Necesitas que permita
método POST, encabezados personalizados y cuerpo de la petición.

**Pon la zona horaria America/Bogota en tu perfil** antes de crear nada.

---

## 3. Los dos trabajos

Misma dirección, cambiando el nombre del archivo:

```
https://api.github.com/repos/TU_USUARIO/reserva-smartflex/actions/workflows/ARCHIVO/dispatches
```

Configuración idéntica en los dos:

- **Método**: `POST`
- **Encabezados**:

| Nombre | Valor |
|---|---|
| `Accept` | `application/vnd.github+json` |
| `Authorization` | `Bearer TU_TOKEN` |
| `X-GitHub-Api-Version` | `2022-11-28` |
| `Content-Type` | `application/json` |

- **Cuerpo**: `{"ref":"main"}`

| Trabajo | ARCHIVO | Horario (hora Bogotá) |
|---|---|---|
| Escuchar y reservar | `escuchar.yml` | cada 5 minutos |
| Recordar | `recordar.yml` | 21:20 todos los días |

Sobre la frecuencia del primero: cada minuto funciona, pero son 1.440 llamadas
diarias al backend del curso, que es un Apps Script modesto. Cada 5 minutos es
imperceptible en el uso real y mucho más considerado. Súbelo solo si te molesta
la espera.

**Una respuesta 204 es éxito.** GitHub no devuelve contenido en este endpoint.

- **404** — el token no tiene el permiso de Actions, o está mal copiado
- **401** — falta la palabra `Bearer` y el espacio antes del token
- **422** — el nombre del archivo no coincide con el del repositorio

---

## 4. Comprobar

Después de un par de minutos:

```bash
gh run list --limit 5
```

Deben aparecer corridas nuevas con `EVENT: workflow_dispatch` que tú no lanzaste.
Esas son las del reloj. Si no aparece ninguna, el trabajo está mal configurado.
