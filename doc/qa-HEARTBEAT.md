# HEARTBEAT.md — QA Agent Heartbeat Checklist

Corré este checklist en cada heartbeat.

## 1. Identidad y contexto

- `GET /api/agents/me` — confirmá tu id, rol, budget, chainOfCommand.
- Revisá el wake context: `PAPERCLIP_TASK_ID`, `PAPERCLIP_WAKE_REASON`, `PAPERCLIP_WAKE_COMMENT_ID`.

## 2. Obtener asignaciones

- `GET /api/agents/me/inbox-lite` — inbox compacto.
- Priorizá: `in_progress` primero, luego `in_review` (si fuiste despertado por un comentario), luego `todo`.
- Si `PAPERCLIP_TASK_ID` está seteado y el task es tuyo, priorizalo.

## 3. Checkout

- `POST /api/issues/{issueId}/checkout` con `X-Paperclip-Run-Id` header.
- 409: el task es de otro agente. Pasá al siguiente. Nunca reintentes un 409.

## 4. Verificar label del issue GitHub

```bash
gh issue view {ISSUE_NUMBER} --repo $GITHUB_REPO --json labels
```

| Label | Acción |
|---|---|
| `spec-approved` | Entrás en **Modo Escritura de Tests** |
| `in-review` | Entrás en **Modo Validación** (el Dev ya abrió el PR) |
| cualquier otro | Salí sin hacer nada |

---

## Modo Escritura de Tests

**Trigger:** label `spec-approved`

1. Leé el issue completo con la spec aprobada:
   ```bash
   gh issue view {ISSUE_NUMBER} --repo $GITHUB_REPO
   ```
2. Extraé los criterios de aceptación de la spec.
3. Escribí los tests en `applications/<app>/tests/test_<modulo>.py` basándote en la spec — sin necesitar el PR todavía.
   - Creá `applications/<app>/tests/__init__.py` vacío si no existe
   - Usá `@pytest.mark.django_db` en toda clase que acceda a DB
   - Nombres descriptivos: `test_retorna_error_si_org_no_existe`
4. Commiteá los tests al branch `main` del worktree base:
   ```bash
   cd $POMETRIX_DIR
   git add applications/<app>/tests/
   git commit -m "$(cat <<'EOF'
   test: acceptance tests for #{issue-number}

   Co-Authored-By: QA Backend <noreply@pometrix.com>
   EOF
   )"
   git push origin main
   ```
5. Actualizá Paperclip:
   ```bash
   scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status in_progress <<'MD'
   Tests de aceptación escritos desde spec — listos para correr contra el PR

   - Archivo: applications/<app>/tests/test_<modulo>.py
   - Criterios cubiertos: {lista}
   - Esperando PR del Dev Agent para correr los tests
   MD
   ```

---

## Modo Validación

### 0. Verificar precondiciones

```bash
gh pr view {PR_NUMBER} --repo $GITHUB_REPO --json state
```

| Condición | Acción |
|---|---|
| PR existe y está abierto | Arrancá la validación |
| No hay PR en el issue | Posteá "Esperando PR del Dev Agent" en Paperclip y salí |

---

### 1. Leer la spec

```bash
gh issue view {ISSUE_NUMBER} --repo $GITHUB_REPO
```

Extraé los **criterios de aceptación** — los 4 puntos SDD:
- Comportamiento actual / problema
- Comportamiento esperado / feature
- Criterios de aceptación de negocio
- Fuera de scope

### 2. Leer el diff del PR

```bash
gh pr diff {PR_NUMBER} --repo $GITHUB_REPO
```

Entendé qué archivos cambió el Dev y qué lógica implementó.

### 3. Crear worktree del branch del PR

```bash
cd $POMETRIX_DIR
git fetch origin
PR_BRANCH=$(gh pr view {PR_NUMBER} --repo $GITHUB_REPO --json headRefName -q .headRefName)
git worktree add ../pometrix-qa-{issue-number} $PR_BRANCH
cd ../pometrix-qa-{issue-number}
cd app
pipenv install --dev
cd ..
```

### 4. Escribir tests pytest de validación

Los tests van dentro de la app de Django afectada, en `applications/<app>/tests/`:

```bash
# Estructura requerida
applications/<app>/tests/__init__.py      # vacío, crearlo si no existe
applications/<app>/tests/test_<modulo>.py # un archivo por módulo testeado
```

Ejemplo:

```python
# applications/bank_movements/tests/test_extraction.py

import pytest

@pytest.mark.django_db
class TestZureoExtraction:

    def test_factura_con_cabecera_extrae_nro_correctamente(self, ...):
        ...

    def test_factura_sin_cabecera_retorna_error_controlado(self, ...):
        ...

    def test_extraccion_no_rompe_flujo_existente_neuralsofts(self, ...):
        ...
```

Convenciones:
- Un archivo por módulo: `test_<modulo>.py`
- Agrupar por clase: `class Test<NombreFuncion>:`
- Nombres descriptivos: `test_retorna_error_si_org_no_existe`
- `@pytest.mark.django_db` en toda clase o función que acceda a DB
- Fixtures en orden de dependencia: `account → org`
- Docstrings solo para casos complejos — el nombre del test se autodocumenta

### 5. Correr los tests

```bash
cd $POMETRIX_DIR/../pometrix-qa-{issue-number}/app

# Solo los tests del issue
pipenv run python -m pytest applications/<app>/tests/test_<modulo>.py -v

# Suite completo para detectar regresiones
pipenv run python -m pytest applications/<app>/ -v --tb=short -q
```

### 6a. Si todos los tests pasan

Comentá en el PR con el detalle técnico:

```bash
gh pr comment {PR_NUMBER} --repo $GITHUB_REPO --body "$(cat <<'EOF'
## ✓ QA aprobado

**Tests de aceptación:** todos pasan
**Regresiones:** ninguna detectada

### Criterios validados
- [CA1] descripción — ✓
- [CA2] descripción — ✓
- [Regresión] flujo existente — ✓

Listo para merge.
EOF
)"
```

Cerrá el loop en el **issue**:

```bash
gh issue comment {ISSUE_NUMBER} --repo $GITHUB_REPO --body "✓ QA aprobado — PR listo para merge. Ver detalles: {PR_URL}"
```

Reasigná al PM Agent y actualizá Paperclip:

```bash
curl -X PATCH "$PAPERCLIP_API_URL/api/issues/$PAPERCLIP_TASK_ID" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
  -d '{"assigneeAgentId": "7e70ee79-e4e3-4c8c-912b-acb1906b4ad8", "status": "in_review"}'
```

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status in_review <<'MD'
QA completado — PR listo para merge

- Tests de aceptación: pasan
- Regresiones: ninguna
- PR: {link}

Board puede mergear.
MD
```

### 6b. Si algún test falla

Comentá en el PR con el detalle técnico exacto:

```bash
gh pr comment {PR_NUMBER} --repo $GITHUB_REPO --body "$(cat <<'EOF'
## ✗ QA bloqueado

**Tests fallidos:**

### `test_factura_sin_cabecera_retorna_error_controlado`
- **Esperado:** HTTP 422
- **Recibido:** HTTP 500
- **Línea:** `api/views.py:87`

El Dev Agent debe corregir antes de re-validar.
EOF
)"
```

Cerrá el loop en el **issue**:

```bash
gh issue comment {ISSUE_NUMBER} --repo $GITHUB_REPO --body "✗ QA bloqueado — el PR requiere correcciones antes de poder mergear. Ver detalles: {PR_URL}"
```

Cambiá el label de GitHub de vuelta a `in-progress`:

```bash
gh issue edit {ISSUE_NUMBER} --repo $GITHUB_REPO --remove-label "in-review" --add-label "in-progress"
```

Actualizá Paperclip a `in_progress` y notificá al Dev Agent:

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status in_progress <<'MD'
QA bloqueado — tests fallidos

- PR: {link}
- Fallo: descripción exacta
- Label GitHub: vuelto a `in-progress`
- Dev Agent debe corregir y notificar cuando el PR esté actualizado
MD
```

### 7. Checklist de salida — NO salgas sin completar esto

Antes de limpiar el worktree y salir, verificá que hiciste **todo**:

**Si tests pasan:**
- [ ] Comenté en el PR con el detalle técnico
- [ ] Comenté en el issue con el resumen (✓ QA aprobado)
- [ ] Reasigné en Paperclip al PM Agent (`7e70ee79-e4e3-4c8c-912b-acb1906b4ad8`) con status `in_review`

**Si tests fallan:**
- [ ] Comenté en el PR con el detalle técnico
- [ ] Comenté en el issue con el resumen (✗ QA bloqueado)
- [ ] Cambié el label del issue a `in-progress`
- [ ] Reasigné en Paperclip al Dev Agent con status `in_progress`

Si alguno de estos pasos está incompleto, completalo antes de salir.

### 8. Limpiar worktree

```bash
cd $POMETRIX_DIR
git worktree remove ../pometrix-qa-{issue-number}
```

---

## 5. Manejo de bloqueos

Si estás bloqueado (no podés acceder al repo, el entorno falla, etc.):

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status blocked <<'MD'
Bloqueado: descripción del problema y quién necesita actuar
MD
```

---

## Reglas críticas

- `main` es staging. Nunca pusheés a `main`.
- Nunca mergeés PRs — eso lo hace el Board.
- Siempre validás en un worktree del branch del PR, nunca en `main`.
- No das ok sin haber corrido los tests. No es un rubber stamp.
- Siempre incluí `X-Paperclip-Run-Id` en requests que modifiquen issues.
- Si hay que commitear tests al branch: `Co-Authored-By: QA Backend <noreply@pometrix.com>`
- **NUNCA marques un task como `done` en Paperclip.** Eso lo hace el Board automáticamente cuando mergea y cierra el issue en GitHub.
