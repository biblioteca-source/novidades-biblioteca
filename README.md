# novidades-biblioteca
Novidades Biblioteca CEIP Belesar
# Novidades Biblioteca · CEIP Belesar

Aplicación web que mostra as últimas adquisicións da biblioteca do colexio nunha televisión da entrada e como galería na web do centro. As portadas, sinopses e datos enriquécense automaticamente desde RBGalicia (Rede de Bibliotecas de Galicia) e outras fontes.

**URL pública:** https://[teu-usuario].github.io/novidades-biblioteca/

- **Modo TV** (carrusel automático): URL principal.
- **Modo web** (galería estilo revista): URL principal + `?modo=web`.

---

## Como se construíu este proxecto

Este sistema desenvolveuse en colaboración con **Claude (de Anthropic)** a través da aplicación de chat en **claude.ai**. Se queres revisar, modificar ou ampliar a aplicación no futuro, podes:

1. Acceder a [claude.ai](https://claude.ai) coa mesma conta `@ceipbelesar.org`.
2. Buscar a conversación orixinal nas túas conversacións (busca por "biblioteca" ou "novidades").
3. Ou comezar unha conversación nova, pegándolle a Claude unha copia deste README e dos ficheiros do repositorio para darlle todo o contexto.

A versión de Claude empregada foi **Claude Sonnet 4.5** e posteriores (Opus 4.7).

---

## Arquitectura do sistema

```
KOHA (catálogo do cole)
        │ feed Atom de últimas adquisicións
        ▼
Apps Script (Google)
   ├─ Le o feed de KOHA
   ├─ Scrapea fichas de detalle
   ├─ Enriquece desde RBGalicia (portada + sinopse + autor)
   ├─ Fallback: Google Books, Open Library, Gemini
   └─ Garda todo nunha Google Sheet
        │ JSON público
        ▼
GitHub Pages (esta web estática)
        ▼
TV LG na entrada do cole + web do centro
```

### Compoñentes

| Compoñente | Onde está | Función |
|---|---|---|
| **KOHA** | `https://ceipbelesar.edubib.xunta.gal/` | Catálogo orixinal da biblioteca |
| **Apps Script** | `script.google.com` (proxecto "Novidades Biblioteca API") | Backend que enriquece os datos |
| **Google Sheet** | Drive de `@ceipbelesar.org` | Curadoría manual + caché de datos |
| **GitHub Pages** | Este repositorio | Frontend público (HTML estático) |
| **RBGalicia** | `catalogo-rbgalicia.xunta.gal` | Fonte de portadas e sinopses |
| **Gemini API** | Google AI | Fallback para sinopses cando RBGalicia non as ten |

---

## Ficheiros do repositorio

- **`index.html`** — A aplicación web (todo en un só ficheiro).
- **`Logo_Belesar2024.png`** — Logo do cole para a cabeceira.
- **`README.md`** — Este ficheiro.
- **`apps-script.gs`** — Código do backend (NON está despregado dende aquí, está copiado en Apps Script de Google; este é só unha copia de referencia).

---

## Como funciona o fluxo automático

Unha vez catalogado un libro novo no KOHA:

1. KOHA inclúeo no feed de novidades (~minutos despois).
2. Apps Script detéctao a próxima vez que se chama (cada 15 min máximo).
3. Scrapea a ficha de KOHA, consulta RBGalicia, e garda os datos enriquecidos na pestana "Caché" da Sheet.
4. A web da TV recarga cada 30 minutos automaticamente. O libro aparece.

**Tempo total estimado dende catalogación ata aparecer na TV:** entre 30 min e 1 h.

---

## Tarefas habituais

### 1. Que faga aparecer un libro novo xa mesmo (sen agardar 30 min)

1. Ir a Apps Script (`script.google.com`, proxecto "Novidades Biblioteca API").
2. Seleccionar `limparCache` no despregable de funcións.
3. Pulsar ▶ Executar.
4. Recargar a páxina da TV. Aparecerá no seguinte refresco.

### 2. Destacar un libro

1. Abrir a Google Sheet.
2. Pestana **Curaduría**.
3. Buscar o libro pola columna `Título`.
4. Na columna `Destacado`, escribir `X` ou `SÍ`.
5. Esperar ata 45 min ou executar `limparCache` para velo decontado.

Os libros destacados aparecen primeiro no carrusel e teñen un texto especial (`Recomendación destacada`) en lugar de `Nova adquisición`.

### 3. Ocultar un libro do carrusel

1. Google Sheet → Pestana Curaduría.
2. Buscar o libro.
3. Na columna `Amosar`, escribir `NON`.
4. Limpar caché para que se note inmediatamente.

### 4. Cambiar a sinopse dun libro pola túa propia

1. Google Sheet → Pestana Curaduría.
2. Na columna `Sinopse`, escribir o teu texto.
3. Esa sinopse manual ten prioridade sobre a automática.

### 5. Engadir libros novos á pestana Curaduría

A pestana Curaduría só se actualiza manualmente. Para engadir os libros novos do feed:

1. Apps Script → seleccionar `actualizarSheet`.
2. ▶ Executar.

Os novos engadiranse ao final da pestana Curaduría con `Amosar = SÍ` por defecto.

**Recomendación:** configura un activador semanal en Apps Script para que `actualizarSheet` se execute automaticamente cada luns. Vai ao icono ⏰ Activadores → + Engadir activador → función `actualizarSheet`, semanal, luns 8:00.

---

## Funcións de Apps Script (referencia)

| Función | Que fai |
|---|---|
| `doGet` | Endpoint HTTP que serve o JSON ao frontend (non se executa manualmente) |
| `proba` | Procesa libros e mostra os primeiros 3 nos rexistros, para depurar |
| `limparCache` | Borra a caché en memoria (15 min). Forza recarga sen perder datos |
| `limparTodaACache` | Borra **toda** a pestana Caché. Usar só se hai datos corruptos |
| `actualizarSheet` | Engade á pestana Curaduría os libros novos do feed |
| `enriquecerConRBGalicia` | Volve consultar RBGalicia para libros que aínda non teñen sinopse boa |

---

## Configuración

### Variables clave en `apps-script.gs`

```javascript
const KOHA_FEED_URL = '...&count=100&format=atom';  // 100 libros máximos
const SHEET_ID = '1-FFrr6jgehiKdxJ9pvXqF3PRc2LD5xer9Wb_9YN0d20';
const CACHE_MINUTES = 15;       // Tempo de caché en memoria
const MAX_BOOKS = 100;          // Máximo de libros mostrados
const MAX_FETCH_PER_RUN = 10;   // Libros novos procesados por execución
```

### Variables clave en `index.html`

```javascript
const API_URL = 'https://script.google.com/macros/s/AKfycby.../exec';
const TV_DURATION_MS = 9000;      // Duración de cada slide na TV (9s)
const REFRESH_MINUTES = 30;       // Refresco do JSON desde a API
```

### Propiedades do script (Apps Script → Configuración do proxecto)

- `GEMINI_API_KEY`: clave da API de Gemini para sinopses como fallback.

---

## Cobertura de datos esperada

Para libros do catálogo do cole (publicados en España, principalmente literatura infantil/xuvenil 2023+):

- **Portadas:** ~80-95% (RBGalicia cobre case todo).
- **Sinopses de RBGalicia:** ~60-70% (algúns libros teñen sinopse profesional na ficha de RBGalicia).
- **Sinopses de Gemini (fallback):** completa case o resto, sempre pedindo "DESCONOCIDO" se non está seguro.
- **Libros sen ningún tipo de portada/sinopse:** mostranse cunha portada tipográfica xerada (cor + título + autor).

---

## Solución de problemas

### A web non carga libros

1. Comproba que a URL de Apps Script está activa: ábrea nun navegador en modo incógnito. Debe devolver JSON.
2. Se devolve erro, vai a Apps Script → Implantar → Xestionar implantacións → editar → Versión: Nova versión → Implantar.

### Os libros aparecen pero sen portada nin sinopse

A caché está a procesarse. Os primeiros 10 libros novos enchense por execución. Espera unhas execucións ou activa un trigger temporal cada 5 min para `proba` ata cubrir todos.

### Cambiei algo no Apps Script e non se ve na TV

A URL pública executa a versión despregada, non a última gardada. Tras calquera cambio no código:
1. Implantar → Xestionar implantacións → editar → Versión: Nova versión → Implantar.
2. Apps Script → executar `limparCache`.
3. Recargar a TV.

### Quero cambiar o número de libros mostrados (100 → outro)

1. En `apps-script.gs`, cambiar `MAX_BOOKS` e o parámetro `count=` da `KOHA_FEED_URL`.
2. Volver despregar (Implantar → ...).
3. Limpar caché.

---

## Despregamento desde cero (se algún día hai que reconstruír)

Se polo que sexa hai que reconstruír o sistema:

1. **Crear Apps Script** novo en `script.google.com` coa cabeceira de `apps-script.gs`. Pegar o código.
2. **Crear Google Sheet** nova. Copiar o ID e pegalo na constante `SHEET_ID`.
3. **Engadir clave de Gemini** nas Propiedades do script (chave `GEMINI_API_KEY`).
4. **Despregar como Web App** con acceso "Calquera".
5. **Copiar a URL** e pegala na constante `API_URL` do `index.html`.
6. **Subir `index.html` + logo** a este repositorio de GitHub.
7. **Activar GitHub Pages** en Settings → Pages → Branch: main, root.

---

## Configuración da TV LG webOS

A TV LG da entrada (modelo webOS 3.x, de 2016) ten dúas opcións documentadas:

**Opción A:** se o navegador de TV permite definir URL de inicio, configurar:
```
https://[teu-usuario].github.io/novidades-biblioteca/
```

**Opción B (recomendada):** Raspberry Pi Zero 2 W conectada por HDMI á TV, executando un kiosco Chromium en pantalla completa. É a solución máis fiable para evitar problemas con timeouts ou pantalla en negro do navegador da TV.

---

## Mantemento esperado

- **Día a día:** ningún. O sistema funciona só.
- **Cada poucos meses:** revisar que `MAX_BOOKS` e `count=` seguen sendo coherentes co que queres mostrar.
- **Cada ano escolar:** opcional, executar `limparTodaACache` ao comezo do curso para forzar reprocesado completo cos cambios que poida haber en RBGalicia.

---

## Contacto técnico

Sistema deseñado para o **CEIP de Belesar** (Galicia).
Desenvolvemento orixinal: **Pablo + Claude (Anthropic) vía claude.ai**, primavera 2026.

Para futuras modificacións, recoméndase abrir unha conversación nova en claude.ai pegando este README como contexto inicial.
