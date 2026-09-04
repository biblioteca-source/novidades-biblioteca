# Novidades Biblioteca · CEIP de Belesar

Panel de novas adquisicións da biblioteca escolar. Amosa os últimos libros
catalogados nunha televisión da entrada do centro e, en móbiles e tablets,
como galería navegable.

**https://novidadesbiblio.ceipbelesar.org**

| Enderezo | Que amosa |
|---|---|
| `novidadesbiblio.ceipbelesar.org` | Automático: pantalla ≥1280 px → presentación; menor → galería |
| `novidadesbiblio.ceipbelesar.org/?modo=tv` | Presentación (carrusel) sempre |
| `novidadesbiblio.ceipbelesar.org/?modo=web` | Galería sempre |

> Atención ao limiar de 1280 px: calquera **ordenador** entra en modo
> presentación. Para repartir un enlace que a xente poida navegar, usa a
> versión `?modo=web`.

Cada libro amósase con portada, autor, sinopse profesional en galego e
signatura. Os datos son do catálogo KOHA do centro; as portadas e sinopses
veñen da Rede de Bibliotecas de Galicia.

---

## Por que o sistema é como é

Ata setembro de 2026 isto era sinxelo: un script de Google lía o catálogo
KOHA directamente. Deixou de funcionar.

**O servidor de KOHA do centro descarta as conexións TCP que veñen de IPs de
centros de datos.** Comprobado desde varias redes: carga sen problema desde
unha casa, desde datos móbiles e desde o propio colexio, pero dá timeout
desde Google, desde un VPS e desde calquera provedor de nube, tanto no porto
443 como no 80. O servidor responde a `ping`, así que non está caído: hai un
firewall que deixa pasar ICMP e tira o resto.

Iso ten dúas consecuencias que conviña deixar escritas, porque son o motivo
de case todas as decisións de deseño:

1. **Non hai truco de código que o resolva.** O bloqueo actúa antes do TLS,
   así que o servidor nin chega a ver que dominio se lle pide. Cambiar
   cabeceiras ou o User-Agent non muda nada.
2. **Ningún proxy en nube serve.** Un VPS, un Cloudflare Worker, unha función
   serverless: todos son IPs de centro de datos e caen no mesmo filtro.

Por iso quen le KOHA agora é **unha Raspberry Pi que está no colexio**, nunha
conexión normal. É a única máquina do sistema que está no lado bo do filtro.

---

## Arquitectura

```
KOHA  (catálogo do centro; só accesible desde conexións non-datacenter)
  │
  │  feed Atom + fichas de detalle
  ▼
RASPBERRY PI  (no colexio, ~/novidades/)
  │  Le o feed, entra nas fichas dos libros novos e saca
  │  biblionumber, título, autor, ISBN e signatura.
  │
  │  POST JSON con token
  ▼
APPS SCRIPT  (Google)
  │  ├─ doPost:      garda o recibido na pestana "Feed"
  │  ├─ enriquece:   portada e sinopse desde RBGalicia
  │  │               (fallback: Google Books → Open Library → Gemini)
  │  ├─ curaduría:   aplica a pestana "Curaduría"
  │  └─ caché:       garda o enriquecido na pestana "Caché"
  │
  │  JSON público (doGet)
  ▼
GITHUB PAGES  (index.html estático)
  │
  ▼
Raspberry Pi en modo kiosco  →  TV da entrada
Móbiles e tablets            →  galería
```

O reparto non é caprichoso: **a Pi só fai o que ningunha outra máquina pode
facer**. Todo o traballo intelixente —e a clave da API de Gemini— queda en
Google. Se a Pi morre, o sistema segue amosando o último que recibiu.

### Compoñentes

| Compoñente | Onde está | Función |
|---|---|---|
| KOHA | `ceipbelesar.edubib.xunta.gal` | Catálogo orixinal |
| Raspberry Pi | Entrada do colexio | Le KOHA e envíao a Google. Move tamén a TV |
| Apps Script | `script.google.com`, proxecto "Novidades Biblioteca API" | Enriquece e serve o JSON |
| Google Sheet | Drive de `@ceipbelesar.org` | Pestanas Feed, Curaduría e Caché |
| GitHub Pages | Este repositorio | Frontend estático |
| RBGalicia | `catalogo-rbgalicia.xunta.gal` | Portadas e sinopses |
| Gemini | Google AI | Último recurso para sinopses |

---

## As tres pestanas da Google Sheet

Entender isto explica case todo o comportamento do sistema.

**`Feed`** — O que envía a Raspberry. Substitúese enteira en cada envío.
Datos crus de KOHA: biblionumber, título, autor, ISBN, signatura, e a orde.
Non a edites a man: o seguinte envío pisaría os cambios.

**`Curaduría`** — **A túa.** É onde mandas ti:
- `Amosar` = `NON` oculta un libro do panel.
- `Sinopse` escrita a man ten prioridade sobre calquera automática.
- `Destacado` = `X` põe o libro primeiro no carrusel, con outro rótulo.

**`Caché`** — Memoria do sistema. Cada libro xa enriquecido queda aquí con
portada e sinopse, para non repetir consultas. Tamén é a rede de seguridade:
se algún día non hai datos frescos, o panel serve isto.

---

## Modo degradado

O sistema ten tres fontes de datos, por orde de preferencia:

1. Pestana **Feed** (o normal, o que envía a Raspberry).
2. Feed Atom **directo de KOHA** (só funcionaría se levantasen o bloqueo).
3. Pestana **Caché** — o *modo degradado*.

Se as dúas primeiras fallan, o panel serve o que hai na Caché en lugar de
amosar un erro. Non entran libros novos, pero **a televisión da entrada nunca
queda en branco**. Foi unha decisión deliberada: un panel na entrada dun
colexio non pode quedar baleiro porque un servidor tardase dez segundos de
máis en contestar.

Pola mesma razón hai dúas barreiras contra o accidente inverso: nin o script
da Pi envía unha lista baleira, nin o `doPost` a acepta. Un fallo de lectura
non pode baleirar o panel.

### Circuit breaker

Cando o feed falla, Apps Script anota a hora e non volve intentalo ata
pasados 20 minutos. Sen iso, cada visita á páxina agardaba uns 50 segundos ao
timeout antes de amosar nada. `limparCache()` borra esa marca e forza un
reintento inmediato.

---

## Tarefas habituais

### Destacar un libro
Sheet → pestana `Curaduría` → columna `Destacado` → escribe `X`.
Aparece primeiro no carrusel, co rótulo *Recomendación destacada*.

### Ocultar un libro
Sheet → `Curaduría` → columna `Amosar` → escribe `NON`.

### Escribir a túa propia sinopse
Sheet → `Curaduría` → columna `Sinopse`. Ten prioridade sobre a automática e
non leva a marca ✨ de xerada automaticamente.

### Que un cambio se vexa xa
Apps Script → executar `limparCache` → recargar a páxina.
Sen iso, o cambio tarda ata 15 minutos.

### Que un libro recén catalogado apareza xa
Por SSH na Raspberry:
```bash
cd ~/novidades && ./novidades-feed.py
```
Despois, `limparCache` no Apps Script. Se non tes présa, a Pi faino soa ás
09:00 e ás 12:00.

### Engadir libros novos á pestana Curaduría
Apps Script → executar `actualizarSheet`. Engade os que aínda non estean, con
`Amosar = SÍ`.

---

## Funcións do Apps Script

| Función | Que fai |
|---|---|
| `doGet` | Serve o JSON ao frontend. Non se executa a man |
| `doPost` | Recibe os datos da Raspberry. Non se executa a man |
| **`diagnostico`** | **Informe de estado. A primeira que hai que executar cando algo falla** |
| `limparCache` | Borra a caché de memoria e desarma o circuit breaker |
| `xerarTokenDoFeed` | Xera un token novo. Invalida o anterior: hai que cambialo tamén na Pi |
| `probarDoPost` | Simula un envío da Raspberry, para probar sen ela |
| `actualizarSheet` | Engade á Curaduría os libros que faltan |
| `enriquecerConRBGalicia` | Volve buscar sinopses en RBGalicia |
| `reintentarPortadasFaltantes` | Volve buscar as portadas que faltan |
| `deduplicarCache` | Limpa filas repetidas da Caché, quedando coa mellor |
| `limparTodaACache` | Borra a pestana Caché enteira. Só se hai datos corruptos |
| `proba` | Amosa os tres primeiros libros nos rexistros |

`diagnostico` di, dunha vez: se KOHA e RBGalicia responden e canto tardan,
cantos libros hai na Feed e na Caché, se o token está configurado, e cantos
libros devolve o sistema agora mesmo.

---

## Configuración

### `apps-script.gs`
```javascript
const SHEET_ID = '1-FFrr6jgehiKdxJ9pvXqF3PRc2LD5xer9Wb_9YN0d20';
const CACHE_MINUTES = 15;           // caché en memoria, modo normal
const CACHE_MINUTES_DEGRADADO = 5;  // caché en memoria, modo degradado
const FEED_RETRY_MINUTES = 20;      // circuit breaker
const MAX_BOOKS = 100;              // libros amosados
const MAX_FETCH_PER_RUN = 10;       // libros novos enriquecidos por execución
```

Propiedades do script (Configuración do proxecto):
- `GEMINI_API_KEY` — clave de Gemini.
- `FEED_TOKEN` — token que valida os envíos da Raspberry.

### `index.html`
```javascript
const API_URL = 'https://script.google.com/macros/s/AKfycby.../exec';
const TV_DURATION_MS = 20000;   // duración de cada libro no carrusel
const REFRESH_MINUTES = 30;     // cada canto se recarga o JSON
const TV_MIN_WIDTH = 1280;      // limiar presentación / galería
```

### Raspberry: `~/novidades/config.json`
```json
{
  "api_url": "...",
  "token": "...",
  "koha_base": "https://ceipbelesar.edubib.xunta.gal/cgi-bin/koha",
  "sucursal": "PEC027",
  "cantos_libros": 100,
  "pausa_segundos": 1
}
```
O temporizador está en `/etc/systemd/system/novidades-feed.timer` (09:00 e
12:00, dentro do horario no que a Pi está acesa).

---

## Solución de problemas

### O panel non amosa libros
Apps Script → `diagnostico`. Mira a liña `getBooks() devolve N libros`.
Se N é maior que cero, o problema está no navegador ou na rede da TV, non nos
datos.

### Non entran libros novos
Na Raspberry: `tail -30 ~/novidades/novidades.log`.
Se di que non pode ler o feed, é a rede da Pi. Se di "Token inválido", o
token do `config.json` non coincide co das Propiedades do script.

### Cambiei o código e non se ve
A URL pública executa a **versión despregada**, non a última gardada. Tras
calquera cambio: **Implantar → Xestionar implantacións → editar → Versión:
Nova versión → Implantar**. Despois, `limparCache`.

### Aparecen dous libros de proba
Executaches `probarDoPost` e non borraches as filas que deixa na pestana
`Feed`. Bórraas e executa `limparCache`.

### Depurar o envío con curl dá 405
Non é un fallo. Apps Script responde 302 e redirixe; `curl -L` reenvía o POST
ao destino, que o rexeita. O script da Pi usa `urllib` de Python, que
converte a redirección en GET, e funciona. Non "arranxes" nada por isto.

### O dominio deixou de funcionar
Comproba que segue existindo o ficheiro `CNAME` na raíz deste repositorio,
cunha soa liña: `novidadesbiblio.ceipbelesar.org`. GitHub créao ao configurar
o dominio persoalizado e, se desaparece, a configuración desfaise soa.

---

## Ficheiros

- `index.html` — a aplicación enteira (HTML, CSS e JS nun só ficheiro).
- `Logo_Belesar2024.png` — logo do centro.
- `CNAME` — dominio persoalizado. **Non borrar.**
- `apps-script.gs` — copia de referencia do backend. **Non se desprega desde
  aquí**: o que corre está pegado no editor de Apps Script de Google.

> Regra aprendida a base de sustos: **antes de xerar unha versión nova de
> calquera dos dous ficheiros grandes, confirma que a copia local coincide co
> que está realmente en produción.** Máis dunha vez os axustes fixéronse
> directamente no editor de Google ou en GitHub e non volveron á copia de
> traballo.

---

## Pendente

- Escribir ao soporte de edubib polo bloqueo de IPs de centros de datos. Se o
  levantan, o sistema volvería á arquitectura simple sen tocar nada: Apps
  Script segue sabendo ler KOHA directamente se a pestana Feed está baleira.
- Se algún día se retira este sitio de GitHub Pages, **borrar tamén o
  rexistro CNAME do DNS** o mesmo día. Un subdominio apuntando a un oco de
  GitHub pode ser reclamado por outra persoa.

---

## Créditos

Sistema deseñado para a biblioteca do **CEIP de Belesar** (Baiona, Galicia).
Desenvolvemento: Pablo Nimo Liboreiro con Claude (Anthropic), 2026.
