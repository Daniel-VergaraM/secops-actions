# secops-actions

SAST · SCA · DAST · IAST · Fuzzing como GitHub Actions, listos para soltar en
cualquier repo con la mínima config posible. Detectan solos el lenguaje (vía
la API de Linguist de GitHub) y, cuando aplica, cómo arranca la app
(`Dockerfile` / `docker-compose.yml`) — nadie tiene que decirle al pipeline en
qué está escrito el proyecto.

| Técnica | Archivo(s) | ¿Corre? | Vista | Motor |
|---|---|---|---|---|
| SAST | `sast.yml` + `codeql-sast.yml` | No | código propio, desde adentro | Semgrep (siempre) + CodeQL (solo en público, o si forzás) |
| SCA | `sca.yml` | No | librerías de terceros | Trivy + dependency-review |
| DAST | `dast.yml` | Sí | desde afuera, como atacante | OWASP ZAP |
| IAST | `iast.yml` | Sí | desde adentro, con agente (aproximado) | cobertura nativa + ZAP |
| Fuzzing | `fuzzing.yml` | Sí | entradas malformadas | Schemathesis si hay schema (OpenAPI/GraphQL), si no ffuf (caja negra) |

## Uso: esto no corre en este repo, corre en el tuyo

Este repo es un paquete de workflows para copiar, no un proyecto con código
que escanear. Para usarlo:

```bash
git clone https://github.com/<tu-usuario-u-org>/secops-actions.git
cp -r secops-actions/.github /ruta/a/tu/proyecto/
cd /ruta/a/tu/proyecto
git add .github
git commit -m "ci: agregar pipeline de seguridad (secops-actions)"
git push
```

Si ya tienes `.github/workflows` en el proyecto destino, copia solo lo que
falte: los 6 `.yml` de `.github/workflows/` y la carpeta completa
`.github/actions/detect-stack/`.

Alternativa sin clonar nada por consola: en GitHub, marca este repo como
**Template repository** (Settings → General → Template repository) y que cada
equipo use el botón **"Use this template"** para arrancar un repo nuevo ya con
`.github/` adentro — mismo resultado, sin pasos de terminal.

## Qué necesitas configurar en el repo DESTINO (donde copiaste `.github/`)

Dos cosas van en **Settings** del repo destino, antes de que nada de esto
sirva:

1. **Settings → Actions → General → Workflow permissions** → "Read and write
   permissions". Sin esto, el paso que sube resultados a la pestaña Security
   (`upload-sarif`) falla en silencio aunque el scan haya corrido bien.
2. **Settings → Code security → Dependency graph** → activado (gratis). Lo usa
   `dependency-review-action` dentro de `sca.yml`.

Con eso, **SAST (Semgrep) y SCA ya funcionan solos**, sin tocar nada más.
CodeQL (`codeql-sast.yml`) necesita que `sast.yml` ya exista en la rama
default antes de poder dispararse (se encadenan por `workflow_run`, ver más
abajo) — así que su primera corrida llega después del primer push, no en el
mismo.

### Configuración semi-automática: variables opcionales

Todo lo de acá abajo es **opcional** y vive en `Settings → Secrets and
variables → Actions → Variables` del repo destino (son variables, visibles
dentro del repo, no secretos — salvo el que dice "(secret)"). Ningún workflow
falla por no tenerlas: cada uno se salta solo la parte que no puede resolver y
deja un `::notice::` explicando qué le faltó.

| Variable | Usada por | Cuándo hace falta | Si no la pones |
|---|---|---|---|
| `DAST_TARGET_URL` | `dast.yml` | no tienes `Dockerfile`/`docker-compose.yml` en el repo, o preferís atacar tu staging real en vez de que el runner levante la app | intenta levantar la app con Docker |
| `DAST_APP_PORT` | `dast.yml`, `iast.yml`, `fuzzing.yml` | tu app no escucha en `8080` | `8080` |
| `IAST_START_CMD` | `iast.yml` | querés que este workflow corra (ej: `node dist/server.js`, `python -m app`, `go run .`) | se salta solo, sin fallar el pipeline |
| `OPENAPI_PATH` | `fuzzing.yml` | tu spec OpenAPI no está en una ruta común | prueba `/openapi.json`, `/swagger.json`, `/v3/api-docs`, etc. |
| `GRAPHQL_PATH` | `fuzzing.yml` | tu endpoint GraphQL no es `/graphql` | prueba `/graphql`, `/api/graphql`, `/v1/graphql`, etc. |
| `CODEQL_FORCE_RUN` | `codeql-sast.yml` | el repo es privado pero sí tenés licencia GHAS | en privado, sin esto, CodeQL se salta solo |
| `SEMGREP_APP_TOKEN` **(secret)** | `sast.yml` | querés comentarios inline en el PR vía Semgrep AppSec Platform | Semgrep corre igual, solo sin esa integración |

Si tu repo ya tiene `Dockerfile` o `docker-compose.yml` en la raíz y la app
escucha en `8080`, **no necesitas poner ninguna variable**: DAST y Fuzzing
levantan la app ellos solos y la atacan.

## Licenciamiento: lo único que hay que decidir, no configurar

`codeql-sast.yml` consulta la visibilidad real del repo (`repos/{owner}/{repo}`
vía la API de GitHub) antes de correr nada:

- **Repo público** → CodeQL es gratis sin límites, corre siempre.
- **Repo privado** → CodeQL avanzado requiere GitHub Advanced Security (de
  pago). El workflow se salta solo para no fallar por licenciamiento ni gastar
  minutos de CI en algo que no vas a poder usar. Si tu organización sí tiene
  GHAS, poné `CODEQL_FORCE_RUN=true` (tabla de arriba) para forzarlo.

Mientras tanto, Semgrep (`sast.yml`) cubre el SAST sin costo ni licencia,
corra o no CodeQL. Todo lo demás del pipeline (Trivy, dependency-review,
OWASP ZAP, Schemathesis, ffuf) también es gratis y open source, sin cuenta ni
token obligatorios.

`codeql-sast.yml` y `sast.yml` están en archivos separados a propósito:
GitHub Actions no deja usar `needs:` entre archivos de workflow distintos, así
que se encadenan con el trigger `workflow_run` — `codeql-sast.yml` se dispara
en cuanto `sast.yml` termina.

## Sobre IAST: por qué `iast.yml` es una aproximación, no léelo como "IAST real"

A la fecha no existe una herramienta de IAST real (con *taint-tracking* —
seguir un dato del atacante hasta ver si llega sin sanitizar a una query SQL)
que sea gratis/open source y cubra Go, TypeScript/Node y Python a la vez:

- **Contrast Community Edition**, la única IAST gratuita seria (Java/.NET
  Core), fue **descontinuada el 30 de junio de 2025**. Contrast Assess hoy es
  100% comercial.
- **OpenRASP** (Baidu, Apache 2.0) sigue libre, pero su último release es de
  enero de 2022 y solo cubre bien Java y PHP sobre servidores de aplicaciones
  tradicionales — no encaja con microservicios en contenedores.
- **Datadog Code Security (IAST)** es comercial (trial de 14 días); es la
  opción más razonable hoy si ya usás Datadog para APM.

`iast.yml` hace lo más honesto que se puede hacer gratis: levanta la app con
instrumentación de cobertura **nativa del lenguaje** (un agente real corriendo
adentro del proceso — `go tool cover`, `coverage.py`, `c8`, JaCoCo), la ataca
con ZAP mientras corre, y entrega qué código se ejecutó realmente durante el
ataque. Es *glass-box testing* real y útil, pero no taint-tracking. Si tu
equipo necesita IAST de verdad, la ruta honesta es pagar Datadog o Contrast —
no hay atajo gratuito hoy.

## Sobre Fuzzing: no toda API tiene spec

La mayoría de las APIs reales no exponen OpenAPI/Swagger. `fuzzing.yml` prueba
tres niveles en orden, sin que nadie elija nada:

1. **REST con spec** → Schemathesis lee el OpenAPI/Swagger y genera miles de
   casos límite solo.
2. **GraphQL** → se autodescribe: `fuzzing.yml` prueba una query de
   introspección (`{__typename}`) contra rutas típicas y, si responde,
   Schemathesis fuzzea desde ahí sin necesidad de spec publicado.
3. **Ninguna de las dos** (el caso más común en APIs internas sin documentar)
   → fallback de caja negra con **ffuf**: descubre endpoints por wordlist
   (SecLists) y golpea cada uno con payloads malformados en query string y
   body, buscando 5xx. Menos preciso que un fuzzer schema-aware, pero no
   depende de que documentes nada.

Tope de 20 endpoints procesados en el paso 3 para no disparar el tiempo de CI.
Para fuzzing de librerías/parsers puros (no HTTP) en Go/C/C++/Python/Rust/Java,
la referencia es [ClusterFuzzLite](https://google.github.io/clusterfuzzlite/)
de Google — no incluido como workflow activo porque siempre exige escribir un
"fuzz target" a mano.

## Checks obligatorios en PRs (opcional, recomendado)

Una vez que confirmes que SAST y SCA corren limpio en el repo destino,
actívalos como *required status checks* en **Settings → Branches → Branch
protection rules** para que no se pueda mergear con vulnerabilidades CRITICAL
sin que alguien lo vea explícitamente (el job `trivy` → step "gate" y el job
`codeql` ya fallan solos ante hallazgos; Semgrep queda en modo informativo por
defecto — quita el `continue-on-error` en `sast.yml` si también querés que
bloquee).

## Estructura de este repo

```
.github/
  actions/detect-stack/action.yml   # detección de lenguaje + cómo arranca la app
  workflows/
    sast.yml           # SAST — Semgrep
    codeql-sast.yml     # SAST — CodeQL (condicionado a visibilidad/GHAS)
    sca.yml             # SCA — Trivy + dependency-review
    dast.yml            # DAST — OWASP ZAP
    iast.yml             # IAST — aproximación (cobertura nativa + ZAP)
    fuzzing.yml          # Fuzzing — Schemathesis / ffuf
README.md
```
