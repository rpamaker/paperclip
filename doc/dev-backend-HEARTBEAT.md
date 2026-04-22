# HEARTBEAT.md — Dev Backend Heartbeat Checklist

Corré este checklist en cada heartbeat.

## 1. Identidad y contexto

- `GET /api/agents/me` — confirmá tu id, rol, budget, chainOfCommand.
- Revisá el wake context: `PAPERCLIP_TASK_ID`, `PAPERCLIP_WAKE_REASON`, `PAPERCLIP_WAKE_COMMENT_ID`.

## 2. Obtener asignaciones

- `GET /api/agents/me/inbox-lite` — inbox compacto.
- Priorizá: `in_progress` primero, luego `in_review` (si fuiste despertado por un comentario), luego `todo`. Saltá `blocked` si no podés desbloquearte.
- Si `PAPERCLIP_TASK_ID` está seteado y el task es tuyo, priorizalo.

## 3. Checkout

- `POST /api/issues/{issueId}/checkout` con `X-Paperclip-Run-Id` header.
- Si recibirás 409: el task es de otro agente. Pasá al siguiente. Nunca reintentes un 409.

## 4. Verificar label del issue GitHub

Antes de tocar cualquier código:

```bash
gh issue view {GITHUB_ISSUE_NUMBER} --repo $GITHUB_REPO --json labels
```

| Label | Acción |
|---|---|
| `spec-open` | Entrás en **Modo Revisión Técnica** |
| `spec-approved` | Entrás en **Modo Implementación** |
| `spec-review` | La pelota está en el PM. No actuás — salí sin tocar nada |
| cualquier otro / ninguno | Posteá "Esperando spec aprobada" en Paperclip y salí |

---

## Modo Revisión Técnica

**Trigger:** label `spec-open`

1. Actualizá el repo antes de explorar cualquier código:
   ```bash
   cd $POMETRIX_DIR
   git fetch origin
   git checkout main && git pull origin main
   ```
2. Leé el issue completo con `gh issue view`.
3. Leé los archivos relevantes del repo (usá Read, Glob, Grep).
4. Validá viabilidad técnica.
5. Identificá casos edge.
6. Estimá complejidad: Simple / Medio / Complejo.
7. Posteá comentario en GitHub con tus observaciones. El comentario **siempre** cierra con una de estas dos señales:
   - `✓ Ok técnico` — spec viable, sin observaciones bloqueantes, PM puede ir al Board
   - `Pendiente: [lista de puntos que el PM debe resolver antes de aprobar]`

   ```bash
   gh issue comment {NUMBER} --repo $GITHUB_REPO --body "$(cat <<'EOF'
   ## Revisión técnica

   **Viabilidad:** viable / no viable (con razón)
   **Complejidad:** Simple / Medio / Complejo

   ### Observaciones
   - ...

   ### Casos edge no contemplados
   - ...

   ### Archivos/módulos que probablemente cambian
   - `{path/al/archivo.py}` — {motivo}

   ✓ Ok técnico
   EOF
   )"
   ```

8. Cambiá el label a `spec-review`:
   ```bash
   gh issue edit {NUMBER} --repo $GITHUB_REPO --remove-label "spec-open" --add-label "spec-review"
   ```
9. Reasigná el issue al PM Agent en Paperclip:
   ```bash
   curl -X PATCH "$PAPERCLIP_API_URL/api/issues/$PAPERCLIP_TASK_ID" \
     -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
     -H "Content-Type: application/json" \
     -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
     -d '{"assigneeAgentId": "7e70ee79-e4e3-4c8c-912b-acb1906b4ad8"}'
   ```
10. Actualizá el issue en Paperclip con un resumen de lo que hiciste.
11. **No toques código.**

---

## Modo Implementación

**Trigger:** label `spec-approved`

`main` es el branch de staging. Todo el trabajo ocurre en un worktree aislado. Nunca mergeés el PR.

> El QA Agent escribe los tests de aceptación en paralelo desde la spec — no tenés que esperarlo para implementar.

1. Leé el issue completo — spec cerrada.
2. Actualizá `main` y creá el worktree para este issue:
   ```bash
   cd $POMETRIX_DIR
   git fetch origin
   git checkout main && git pull
   git worktree add ../pometrix-{issue-number} -b {issue-number}-{slug} main
   cd ../pometrix-{issue-number}/app
   pipenv install --dev
   cd ..
   ```
3. Leé el código relevante antes de escribir nada.
4. Implementá siguiendo la spec. No agregues nada que la spec no pida.
5. Escribí o actualizá los tests pytest correspondientes.
6. Corré los tests dentro del worktree:
   ```bash
   cd app
   pipenv run python -m pytest applications/<app>/tests/test_<modulo>.py -v
   pipenv run python -m pytest applications/<app>/ -v --tb=short -q
   ```
7. Si los tests pasan: commiteá y pusheá:
   ```bash
   git add {archivos relevantes}
   git commit -m "$(cat <<'EOF'
   fix/feat: descripción del cambio

   Refs #{issue-number}

   Co-Authored-By: Dev Backend <noreply@pometrix.com>
   EOF
   )"
   git push -u origin {issue-number}-{slug}
   ```
8. Verificá si ya existe un PR abierto para este branch. Si existe, no crees uno nuevo:
   ```bash
   EXISTING_PR=$(gh pr list --repo $GITHUB_REPO --head {issue-number}-{slug} --json number -q '.[0].number')

   if [ -n "$EXISTING_PR" ]; then
     echo "PR #$EXISTING_PR ya existe — push actualiza el PR automáticamente"
     PR_NUMBER=$EXISTING_PR
     PR_URL=$(gh pr view $PR_NUMBER --repo $GITHUB_REPO --json url -q '.url')
   else
     PR_URL=$(gh pr create --repo $GITHUB_REPO \
       --base main \
       --title "{título}" \
       --body "$(cat <<'EOF'
   Refs #{issue-number}

   ## Qué cambia
   - ...

   ## Tests
   - ...
   EOF
     )")
     PR_NUMBER=$(echo "$PR_URL" | grep -o '[0-9]*$')
   fi
   ```
9. Monitoreá el CI:
   ```bash
   gh pr checks {PR_NUMBER} --repo $GITHUB_REPO --watch
   ```
   Si falla: corregí en el worktree, commiteá y pusheá de nuevo.
10. Cuando el CI pasa: cambiá el label, comentá en el issue, y reasigná al QA Agent.
    ```bash
    gh issue edit {NUMBER} --repo $GITHUB_REPO --remove-label "spec-approved" --add-label "in-review"
    ```
    ```bash
    gh issue comment {NUMBER} --repo $GITHUB_REPO --body "PR listo para revisión de QA: {PR_URL}"
    ```
    ```bash
    curl -X PATCH "$PAPERCLIP_API_URL/api/issues/$PAPERCLIP_TASK_ID" \
      -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
      -H "Content-Type: application/json" \
      -H "X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID" \
      -d '{"assigneeAgentId": "30876036-0dfe-4dbe-806b-4e27a1e127f6"}'
    ```
11. Si el QA Agent bloquea: corregí en el worktree existente, pusheá, y re-notificá al QA.
12. Checklist de salida — NO salgas sin completar esto:
    - [ ] El PR está abierto y pusheado
    - [ ] Cambié el label del issue a `in-review`
    - [ ] Reasigné en Paperclip al QA Agent (`30876036-0dfe-4dbe-806b-4e27a1e127f6`)
    - [ ] Comenté en el **GitHub issue** con el link al PR (`gh issue comment`)

    Si alguno está incompleto, completalo antes de salir.
13. Cuando el QA Agent confirme que las pruebas pasan: limpiá el worktree.
    ```bash
    cd $POMETRIX_DIR
    git worktree remove ../pometrix-{issue-number}
    ```

---

## 5. Manejo de bloqueos

Si en cualquier punto estás bloqueado:

```bash
PATCH /api/issues/{issueId}
{ "status": "blocked", "comment": "Qué está bloqueado, por qué, y quién necesita desbloquearlo." }
```

Escalá al PM Agent con un comentario explicando el bloqueo.

## 6. Actualizar estado y salir

- Siempre comentá en el issue de Paperclip antes de salir.
- Si el PR está abierto y el CI pasa: cambiá label GitHub a `in-review`, marcá Paperclip como `in_review`, y notificá al QA Agent.
- Si QA bloquea y te despierta: corregí, pusheá, re-notificá al QA.
- Si QA aprueba: el Board mergea y cierra el issue.

---

## Reglas críticas

- `main` es staging. Todo PR apunta a `main`. Nunca pusheés directo a `main`.
- Siempre trabajá en un worktree — nunca en el repo base.
- Nunca mergeés PRs — eso lo hace el Board.
- Nunca modifiques `.github/workflows/` ni archivos de CI/CD.
- Con `spec-open`: Modo Revisión Técnica. Con `spec-approved`: Modo Implementación. Con cualquier otro label: salí sin tocar código.
- Siempre incluí `X-Paperclip-Run-Id` en requests que modifiquen issues.
- Cada commit debe incluir: `Co-Authored-By: Dev Backend <noreply@pometrix.com>`
- **NUNCA marques un task como `done` en Paperclip.** Eso lo hace el Board automáticamente cuando mergea y cierra el issue en GitHub.
- **Schema siempre actualizado:** si el PR agrega o modifica endpoints, verificá que el schema refleje los cambios antes de abrir el PR. El Dev Frontend lo consume así:
  ```bash
  curl -s -H "Authorization: Token $BACKEND_API_TOKEN" \
    https://backend.stg.doculyzer.ai/schema/agents/
  ```
  Si el schema no está actualizado (drf-spectacular lo genera automáticamente desde el código), revisá que los serializers y views tengan los decoradores correctos (`@extend_schema`, etc.) antes de pushear.
