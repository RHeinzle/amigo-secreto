---
name: "amigo-secreto"
description: "Sorteia um amigo secreto a partir de uma lista de nomes e publica um Artifact com um link privado e cifrado por pessoa. Use quando o usuário enviar nomes para um amigo secreto."
---

# Amigo secreto com links privados

Resultado: uma única página (Artifact) em que cada pessoa abre o próprio link e vê só o seu amigo secreto. O sorteio é feito em código, fica cifrado dentro da página e é decifrado no navegador com o código que vai no final do link (`#nome-código`). A página tem tema natalino (luzes, neve, estrela, pinheiros; sem Papai Noel).

## Antes de sortear

- É preciso poder rodar código (Node). Se não for possível, avise e pare. Nunca sorteie "de cabeça".
- Com menos de 2 nomes ou nomes repetidos, avise o usuário antes de sortear. O script também recusa esses casos.
- Restrições informadas pelo usuário (quem não pode tirar quem) entram como `--nao-tira="Nome>Nome"`, uma por restrição.

## Passo a passo

1. Numa pasta de trabalho, grave os quatro arquivos da seção "Arquivos" exatamente como estão, com os nomes `lib.js`, `template.src.html`, `montar-neve.mjs` e `gerar.mjs`. Não altere o código sem pedido do usuário.
2. Na pasta, rode `node montar-neve.mjs`. Ele gera `template.html` a partir de `template.src.html`, com a neve embutida.
3. Rode `node gerar.mjs --ano=<ano atual> [--nao-tira="A>B" ...] "Nome 1" "Nome 2" "Nome 3" ...`. O script sorteia, cifra, monta `amigo-secreto.html`, verifica tudo e imprime um link por pessoa (`Nome<tab>#nome-código`). Se imprimir `FALHOU` ou `ERRO`, não publique: corrija e rode de novo.
4. Publique `amigo-secreto.html` como Artifact (ícone `gift`; descrição: "Sorteio de amigo secreto: cada pessoa abre o próprio link e vê só quem tirou."). O título da página já vem pronto: "Amigo Secreto <ano>".
5. Responda só com a lista `Nome: <URL do Artifact>#nome-código` e uma linha lembrando que o Artifact precisa ser compartilhado antes de enviar os links: com Free, Pro ou Max, publicar com link público permite abrir sem login; com Team ou Enterprise só quem é da organização abre. Sugira testar um link numa aba anônima.

## Regras

- Nunca mostre o sorteio nem grave em texto aberto em arquivo, log ou saída de comando. Tudo isso aparece para o usuário. O script só imprime os links.
- Não mostre o código nem o conteúdo da página, a menos que o usuário peça.
- Se o usuário pedir para ver quem tirou quem, explique que cada resultado só aparece no link de cada pessoa.
- Código de 8 caracteres (A-Z, a-z, 0-9) que diferencia maiúsculas de minúsculas. Só o nome antes do último hífen vai em minúsculas.
- Página: o texto, o visual e o comportamento do template (saudação, setas, tela vazia, nome grande no fim; aviso para link sem código ou inválido; tema natalino) só mudam se o usuário pedir. Nesse caso, edite `template.src.html`, rode `montar-neve.mjs` e `gerar.mjs` de novo e republique. Mantenha os ids dos elementos e o script, porque o teste do `gerar.mjs` depende deles.

## Arquivos

### lib.js

```js
function b64ParaBytes(s) {
  var bin = atob(s);
  var out = new Uint8Array(bin.length);
  for (var i = 0; i < bin.length; i++) out[i] = bin.charCodeAt(i);
  return out;
}

function separarLink(hash) {
  var h = String(hash || '').replace(/^#/, '');
  var i = h.lastIndexOf('-');
  if (i < 1 || i === h.length - 1) return null;
  return { slug: h.slice(0, i).toLowerCase(), token: h.slice(i + 1) };
}

async function abrirSorteio(dados, hash) {
  var p = separarLink(hash);
  if (!p) return null;
  if (!Object.prototype.hasOwnProperty.call(dados.pessoas, p.slug)) return null;
  var reg = dados.pessoas[p.slug];
  try {
    var base = await crypto.subtle.importKey('raw', new TextEncoder().encode(p.token), 'PBKDF2', false, ['deriveKey']);
    var chave = await crypto.subtle.deriveKey(
      { name: 'PBKDF2', salt: b64ParaBytes(dados.sal), iterations: dados.iter, hash: 'SHA-256' },
      base,
      { name: 'AES-GCM', length: 256 },
      false,
      ['decrypt']
    );
    var bruto = await crypto.subtle.decrypt({ name: 'AES-GCM', iv: b64ParaBytes(reg.iv) }, chave, b64ParaBytes(reg.ct));
    return JSON.parse(new TextDecoder().decode(bruto));
  } catch (e) {
    return null;
  }
}
```

### template.src.html

```html
<title>Amigo Secreto /*ANO*/</title>
<meta name="claude:viewport-fit-sides" content="cover">
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,400..800&display=swap">
<style>
/* Layout: coluna alinhada à esquerda sob um fio de luzes de Natal; neve cai ao fundo; saudação, trilha de setas e, uma tela abaixo, o nome com estrela e pinheiros. */
:root {
  --papel: #E2EDE6;
  --tinta: #0E3A2D;
  --suave: #43655A;
  --fita: #B3202A;
  --ouro: #C99A1E;
  --pinho: #1B6B4F;
  --pinho-claro: #2E8A68;
  --listra: #FFFFFF;
  --cabo: #34584B;
  --luz-1: #D93A3A;
  --luz-2: #F2B632;
  --luz-3: #3DAA6D;
  --luz-4: #5BA8E0;
  --fonte-display: "Bricolage Grotesque", "Avenir Next", "Segoe UI", system-ui, sans-serif;
  --fonte-texto: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
}
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --papel: #0A261E;
    --tinta: #EEF4EF;
    --suave: #A3C2B6;
    --fita: #F2706B;
    --ouro: #F2CB6B;
    --pinho: #1F7A58;
    --pinho-claro: #34966F;
    --cabo: #4C7366;
    color-scheme: dark;
  }
}
:root[data-theme="dark"] {
  --papel: #0A261E;
  --tinta: #EEF4EF;
  --suave: #A3C2B6;
  --fita: #F2706B;
  --ouro: #F2CB6B;
  --pinho: #1F7A58;
  --pinho-claro: #34966F;
  --cabo: #4C7366;
  color-scheme: dark;
}

body {
  margin: 0;
  background: var(--papel);
  color: var(--tinta);
  font: 16px/1.5 var(--fonte-texto);
  padding-left: max(16px, env(safe-area-inset-left, 0px));
  padding-right: max(16px, env(safe-area-inset-right, 0px));
  -webkit-text-size-adjust: 100%;
}
main {
  position: relative;
  z-index: 1;
  max-width: 40rem;
  margin-inline: auto;
  padding-inline: 8px;
}

/* Neve: duas camadas de flocos que descem em velocidades diferentes. */
.neve {
  position: fixed;
  inset: 0;
  z-index: 0;
  pointer-events: none;
  background-image: url("/*NEVE-PERTO*/"), url("/*NEVE-LONGE*/");
  background-size: 500px 500px, 300px 300px;
  background-position: 0 0, 0 0;
  animation: nevar 28s linear infinite;
}
@keyframes nevar {
  from { background-position: 0 0, 0 0; }
  to { background-position: 0 500px, 0 300px; }
}

/* Fio de luzes: lâmpadas coloridas que alternam o brilho. */
.luzes {
  position: fixed;
  top: env(safe-area-inset-top, 0px);
  left: 0;
  right: 0;
  height: 28px;
  z-index: 2;
  pointer-events: none;
  background: linear-gradient(var(--cabo), var(--cabo)) 0 6px / 100% 2px no-repeat;
}
.luzes::before,
.luzes::after {
  content: "";
  position: absolute;
  inset: 0;
  animation: piscar 2.6s ease-in-out infinite;
}
.luzes::before {
  background:
    radial-gradient(circle at 20px 17px, var(--luz-1) 0 5px, transparent 5.5px) 0 0 / 160px 100%,
    radial-gradient(circle at 100px 17px, var(--luz-3) 0 5px, transparent 5.5px) 0 0 / 160px 100%;
}
.luzes::after {
  background:
    radial-gradient(circle at 60px 17px, var(--luz-2) 0 5px, transparent 5.5px) 0 0 / 160px 100%,
    radial-gradient(circle at 140px 17px, var(--luz-4) 0 5px, transparent 5.5px) 0 0 / 160px 100%;
  animation-delay: 1.3s;
}
@keyframes piscar {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
}

.abertura,
.revelacao,
.aviso {
  min-height: 34rem;
  min-height: max(34rem, 100vh);
  min-height: max(34rem, 100svh);
  display: flex;
  flex-direction: column;
  justify-content: center;
  box-sizing: border-box;
}
.abertura { gap: clamp(28px, 6vh, 56px); }
.revelacao { gap: 22px; }
.aviso { gap: 16px; }

.saudacao,
.aviso h1 {
  margin: 0;
  font-family: var(--fonte-display);
  font-size: clamp(2rem, 9vw, 3.5rem);
  line-height: 1.08;
  letter-spacing: -0.025em;
  text-wrap: balance;
}
.saudacao { font-weight: 450; }
.saudacao .quem { font-weight: 800; }
.aviso h1 { font-weight: 800; }
.aviso p {
  margin: 0;
  max-width: 32rem;
  font-size: 1.0625rem;
  line-height: 1.55;
  color: var(--suave);
}

.trilha {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 10px;
}
.seta {
  display: block;
  width: 44px;
  height: 24px;
  animation: onda 1.8s ease-in-out infinite;
}
.seta path {
  fill: none;
  stroke: var(--fita);
  stroke-width: 5;
  stroke-linecap: round;
  stroke-linejoin: round;
}
.seta:nth-child(2) { animation-delay: 0.15s; }
.seta:nth-child(3) { animation-delay: 0.3s; }
.seta:nth-child(4) { animation-delay: 0.45s; }
@keyframes onda {
  0%, 100% { opacity: 0.2; transform: translateY(0); }
  40% { opacity: 1; transform: translateY(5px); }
}

.enfeite {
  display: flex;
  align-items: center;
  gap: 14px;
}
.estrela {
  display: block;
  width: 40px;
  height: 40px;
  fill: var(--ouro);
}
.fita {
  display: block;
  width: 96px;
  height: 12px;
  border-radius: 6px;
  background: repeating-linear-gradient(-45deg, var(--fita) 0 8px, var(--listra) 8px 16px);
}
.amigo {
  margin: 0;
  font-family: var(--fonte-display);
  font-optical-sizing: auto;
  font-weight: 800;
  font-size: calc(clamp(3rem, 17vw, 8.5rem) * var(--escala, 1));
  line-height: 0.95;
  letter-spacing: -0.035em;
  color: var(--fita);
  overflow-wrap: anywhere;
  text-wrap: balance;
}
.pinheiros {
  display: block;
  width: min(15rem, 70%);
  height: auto;
  margin-top: 18px;
}
.pinheiros .copa { fill: var(--pinho); }
.pinheiros .copa-alta { fill: var(--pinho-claro); }
.pinheiros .tronco { fill: var(--cabo); }
.pinheiros .bola-1 { fill: var(--fita); }
.pinheiros .bola-2 { fill: var(--ouro); }

@media (prefers-reduced-motion: reduce) {
  .seta,
  .neve,
  .luzes::before,
  .luzes::after { animation: none; }
}
</style>

<div class="neve" aria-hidden="true"></div>
<div class="luzes" aria-hidden="true"></div>

<main>
  <div id="sorteio" hidden>
    <section class="abertura">
      <h1 class="saudacao"><span class="quem" id="quem"></span>, seu amigo secreto é:</h1>
      <div class="trilha" aria-hidden="true">
        <svg class="seta" viewBox="0 0 44 24"><path d="M4 4 L22 20 L40 4"/></svg>
        <svg class="seta" viewBox="0 0 44 24"><path d="M4 4 L22 20 L40 4"/></svg>
        <svg class="seta" viewBox="0 0 44 24"><path d="M4 4 L22 20 L40 4"/></svg>
        <svg class="seta" viewBox="0 0 44 24"><path d="M4 4 L22 20 L40 4"/></svg>
      </div>
    </section>
    <section class="revelacao" aria-label="Seu amigo secreto">
      <div class="enfeite" aria-hidden="true">
        <svg class="estrela" viewBox="0 0 24 24"><path d="M12 1.5l2.9 6.6 7.1.7-5.4 4.7 1.6 7-6.2-3.7-6.2 3.7 1.6-7L2 8.8l7.1-.7z"/></svg>
        <span class="fita"></span>
      </div>
      <p class="amigo" id="amigo"></p>
      <svg class="pinheiros" viewBox="0 0 220 64" aria-hidden="true">
        <g transform="translate(36 6) scale(0.7)">
          <rect class="tronco" x="-4" y="54" width="8" height="8" rx="1"/>
          <polygon class="copa" points="0,28 -30,56 30,56"/>
          <polygon class="copa-alta" points="0,16 -24,40 24,40"/>
          <polygon class="copa" points="0,4 -18,26 18,26"/>
        </g>
        <g transform="translate(184 6) scale(0.7)">
          <rect class="tronco" x="-4" y="54" width="8" height="8" rx="1"/>
          <polygon class="copa" points="0,28 -30,56 30,56"/>
          <polygon class="copa-alta" points="0,16 -24,40 24,40"/>
          <polygon class="copa" points="0,4 -18,26 18,26"/>
        </g>
        <g transform="translate(110 0)">
          <rect class="tronco" x="-4" y="54" width="8" height="8" rx="1"/>
          <polygon class="copa" points="0,28 -30,56 30,56"/>
          <polygon class="copa-alta" points="0,16 -24,40 24,40"/>
          <polygon class="copa" points="0,4 -18,26 18,26"/>
        </g>
        <circle class="bola-1" cx="104" cy="44" r="2.6"/>
        <circle class="bola-2" cx="117" cy="49" r="2.6"/>
        <circle class="bola-2" cx="109" cy="29" r="2.2"/>
        <circle class="bola-1" cx="114" cy="17" r="2"/>
      </svg>
    </section>
  </div>
  <section id="aviso" class="aviso" hidden>
    <h1 id="aviso-titulo"></h1>
    <p id="aviso-texto"></p>
  </section>
</main>

<script>
/*LIB*/

var DADOS = /*DADOS*/;

function porId(id) { return document.getElementById(id); }

function mostrarAviso(titulo, texto) {
  porId('sorteio').hidden = true;
  porId('aviso-titulo').textContent = titulo;
  porId('aviso-texto').textContent = texto;
  porId('aviso').hidden = false;
}

function mostrarSorteio(nome, amigo) {
  var maior = 1;
  amigo.split(/\s+/).forEach(function (palavra) {
    if (palavra.length > maior) maior = palavra.length;
  });
  porId('quem').textContent = nome;
  porId('amigo').textContent = amigo;
  porId('amigo').style.setProperty('--escala', String(Math.min(1, 8 / maior)));
  porId('aviso').hidden = true;
  porId('sorteio').hidden = false;
}

async function abrirPagina() {
  var hash = location.hash;
  if (!hash || hash === '#') {
    mostrarAviso('Amigo secreto', 'Abra o link pessoal que você recebeu para ver quem você tirou.');
    return;
  }
  if (!(window.crypto && window.crypto.subtle)) {
    mostrarAviso('Não foi possível abrir', 'Este navegador não consegue abrir o sorteio. Tente de novo no Chrome ou no Safari.');
    return;
  }
  var r = await abrirSorteio(DADOS, hash);
  if (r && r.nome && r.amigo) {
    mostrarSorteio(r.nome, r.amigo);
  } else {
    mostrarAviso('Link inválido', 'Copie o link inteiro, inclusive o final depois do #. Se continuar assim, peça um novo link a quem organizou o sorteio.');
  }
}

window.addEventListener('hashchange', abrirPagina);
abrirPagina();
</script>
```

### montar-neve.mjs

```js
import { readFileSync, writeFileSync } from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

// Gera duas telhas de flocos (SVG em data URI, sem emenda vertical) e grava template.html a partir de template.src.html.
const dir = path.dirname(fileURLToPath(import.meta.url));

function mulberry32(a) {
  return () => {
    a |= 0; a = (a + 0x6d2b79f5) | 0;
    let t = Math.imul(a ^ (a >>> 15), 1 | a);
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
  };
}

function telha(tam, qtd, rmin, rmax, opacidade, semente) {
  const rnd = mulberry32(semente);
  let c = '';
  for (let i = 0; i < qtd; i++) {
    const x = rnd() * tam, y = rnd() * tam, r = rmin + rnd() * (rmax - rmin);
    for (const dx of [-tam, 0, tam]) {
      for (const dy of [-tam, 0, tam]) {
        const cx = x + dx, cy = y + dy;
        if (cx > -r && cx < tam + r && cy > -r && cy < tam + r) {
          c += `<circle cx='${cx.toFixed(1)}' cy='${cy.toFixed(1)}' r='${r.toFixed(1)}'/>`;
        }
      }
    }
  }
  const svg = `<svg xmlns='http://www.w3.org/2000/svg' width='${tam}' height='${tam}' viewBox='0 0 ${tam} ${tam}'><g fill='white' fill-opacity='${opacidade}'>${c}</g></svg>`;
  return 'data:image/svg+xml,' + encodeURIComponent(svg);
}

const perto = telha(500, 12, 2.2, 3.6, 0.85, 7);
const longe = telha(300, 16, 1.1, 2.0, 0.6, 21);

const fonte = readFileSync(path.join(dir, 'template.src.html'), 'utf8');
if (!fonte.includes('/*NEVE-PERTO*/') || !fonte.includes('/*NEVE-LONGE*/')) throw new Error('marcadores de neve não encontrados');
const saida = fonte.replace('/*NEVE-PERTO*/', () => perto).replace('/*NEVE-LONGE*/', () => longe);
writeFileSync(path.join(dir, 'template.html'), saida);
console.log('template.html gerado:', saida.length, 'caracteres');
```

### gerar.mjs

```js
import { readFileSync, writeFileSync } from 'node:fs';
import { webcrypto, randomInt } from 'node:crypto';
import { fileURLToPath } from 'node:url';
import path from 'node:path';
import vm from 'node:vm';

// Uso: node gerar.mjs --ano=2026 [--nao-tira="Ana>Bia" ...] "Nome 1" "Nome 2" "Nome 3" ...
const dir = path.dirname(fileURLToPath(import.meta.url));
const ITER = 150000;
const ALF = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
const TAM_CODIGO = 8;

// ---------- Entrada ----------
let ano = String(new Date().getFullYear());
const nomes = [];
const proibidos = [];
for (const a of process.argv.slice(2)) {
  if (a.startsWith('--ano=')) ano = a.slice(6).replace(/[^0-9A-Za-z ]/g, '');
  else if (a.startsWith('--nao-tira=')) proibidos.push(a.slice(11).split('>').map((s) => s.trim()));
  else nomes.push(a.trim());
}

const slugBase = (n) =>
  n.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '');
const novoCodigo = () => Array.from({ length: TAM_CODIGO }, () => ALF[randomInt(ALF.length)]).join('');
const b64 = (u8) => Buffer.from(u8).toString('base64');
const falhar = (msg) => { console.error('ERRO: ' + msg); process.exit(1); };

if (nomes.length < 2) falhar('são necessários pelo menos 2 nomes');
if (new Set(nomes).size !== nomes.length) falhar('há nomes repetidos');
// Cada nome ganha um final de link único. Se dois nomes diferem só por acento ou maiúscula (João e Joao), o segundo recebe -2.
const slugUsado = new Set();
const slugPorNome = {};
for (const n of nomes) {
  const base = slugBase(n);
  if (!base) falhar('há nome sem letras ou números');
  let s = base;
  for (let k = 2; slugUsado.has(s); k++) s = `${base}-${k}`;
  slugUsado.add(s);
  slugPorNome[n] = s;
}
const slugDe = (n) => slugPorNome[n];
for (const p of proibidos) {
  if (p.length !== 2 || !nomes.includes(p[0]) || !nomes.includes(p[1])) {
    falhar('restrição inválida: use --nao-tira="Nome>Nome" com nomes da lista');
  }
}

// ---------- Sorteio: embaralha até ninguém tirar a si mesmo ----------
function sortear() {
  for (let t = 0; t < 20000; t++) {
    const alvo = [...nomes];
    for (let i = alvo.length - 1; i > 0; i--) {
      const j = randomInt(i + 1);
      [alvo[i], alvo[j]] = [alvo[j], alvo[i]];
    }
    if (nomes.some((n, i) => alvo[i] === n)) continue;
    const amigo = {};
    nomes.forEach((n, i) => { amigo[n] = alvo[i]; });
    if (!proibidos.some(([x, y]) => amigo[x] === y)) return amigo;
  }
  falhar('não foi possível respeitar as restrições');
}
const amigoDe = sortear();

// ---------- Cifra: AES-GCM, chave derivada do código por PBKDF2-SHA256 ----------
const sal = webcrypto.getRandomValues(new Uint8Array(16));
async function cifrar(codigo, texto) {
  const base = await webcrypto.subtle.importKey('raw', new TextEncoder().encode(codigo), 'PBKDF2', false, ['deriveKey']);
  const chave = await webcrypto.subtle.deriveKey(
    { name: 'PBKDF2', salt: sal, iterations: ITER, hash: 'SHA-256' },
    base,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt']
  );
  const iv = webcrypto.getRandomValues(new Uint8Array(12));
  const ct = new Uint8Array(await webcrypto.subtle.encrypt({ name: 'AES-GCM', iv }, chave, new TextEncoder().encode(texto)));
  return { iv: b64(iv), ct: b64(ct) };
}

const codigos = {};
const dados = { v: 1, iter: ITER, sal: b64(sal), pessoas: {} };
for (const n of nomes) {
  codigos[n] = novoCodigo();
  dados.pessoas[slugDe(n)] = await cifrar(codigos[n], JSON.stringify({ nome: n, amigo: amigoDe[n] }));
}
const links = Object.fromEntries(nomes.map((n) => [n, `#${slugDe(n)}-${codigos[n]}`]));

// ---------- Página ----------
const lib = readFileSync(path.join(dir, 'lib.js'), 'utf8');
const html = readFileSync(path.join(dir, 'template.html'), 'utf8')
  .replace('/*LIB*/', () => lib)
  .replace('/*DADOS*/', () => JSON.stringify(dados))
  .replace('/*ANO*/', () => ano);
const saida = path.join(dir, 'amigo-secreto.html');
writeFileSync(saida, html);

// ---------- Verificação (nunca imprime quem tirou quem) ----------
const falhas = [];
const checar = (nome, ok) => { if (!ok) falhas.push(nome); };

checar('ninguém tira a si mesmo', nomes.every((n) => amigoDe[n] !== n));
checar('todos são tirados uma vez', new Set(Object.values(amigoDe)).size === nomes.length);
checar('restrições respeitadas', proibidos.every(([x, y]) => amigoDe[x] !== y));

const script = html.match(/<script>([\s\S]*)<\/script>/)[1];
new vm.Script(script); // erro de sintaxe lança aqui

async function rodarPagina(hash) {
  const els = {};
  const get = (id) => (els[id] ??= {
    hidden: true,
    textContent: '',
    props: {},
    style: { setProperty(k, v) { els[id].props[k] = v; } },
  });
  const fn = new Function('document', 'window', 'location', 'crypto', script);
  fn({ getElementById: get }, { crypto: webcrypto, addEventListener() {} }, { hash }, webcrypto);
  for (let i = 0; i < 200; i++) {
    await new Promise((r) => setTimeout(r, 50));
    if (!get('sorteio').hidden || !get('aviso').hidden) break;
  }
  return {
    sorteio: !get('sorteio').hidden,
    aviso: !get('aviso').hidden,
    quem: get('quem').textContent,
    amigo: get('amigo').textContent,
    titulo: get('aviso-titulo').textContent,
  };
}

for (const n of nomes) {
  const r = await rodarPagina(links[n]);
  checar(`${n}: abre o sorteio`, r.sorteio && !r.aviso);
  checar(`${n}: saudação com o próprio nome`, r.quem === n);
  checar(`${n}: resultado bate com o sorteio`, r.amigo === amigoDe[n]);
}

const primeiro = nomes[0];
const segundo = nomes[1];
const trocaCaixa = codigos[primeiro].replace(/[A-Za-z]/, (c) => (c === c.toLowerCase() ? c.toUpperCase() : c.toLowerCase()));
const invalidos = {
  'código errado': `#${slugDe(primeiro)}-${novoCodigo()}`,
  'código de outra pessoa': `#${slugDe(primeiro)}-${codigos[segundo]}`,
  'sem código': `#${slugDe(primeiro)}`,
  'nome inexistente': `#nome-inexistente-${codigos[primeiro]}`,
  'nome especial': `#__proto__-${codigos[primeiro]}`,
};
if (trocaCaixa !== codigos[primeiro]) invalidos['maiúscula trocada no código'] = `#${slugDe(primeiro)}-${trocaCaixa}`;
for (const [nome, hash] of Object.entries(invalidos)) {
  const r = await rodarPagina(hash);
  checar(`link inválido (${nome}) mostra aviso`, r.aviso && !r.sorteio && r.titulo === 'Link inválido');
}
const maiusculo = await rodarPagina(`#${slugDe(primeiro).toUpperCase()}-${codigos[primeiro]}`);
checar('nome em maiúsculas no link ainda abre', maiusculo.sorteio && maiusculo.amigo === amigoDe[primeiro]);
const vazio = await rodarPagina('');
checar('sem hash mostra orientação', vazio.aviso && !vazio.sorteio && vazio.titulo === 'Amigo secreto');

if (falhas.length) {
  console.log('FALHOU: ' + falhas.join(' | '));
  process.exit(1);
}
console.log('verificações: todas ok');
console.log('arquivo: ' + saida);
for (const n of nomes) console.log(`${n}\t${links[n]}`);
```