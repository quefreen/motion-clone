# Stack — documento de apoio

Onde as informações vivem no produto original (`app.arqe.ai`) e a lógica que
usamos para reconstruir a cena **Stack** no clone local (`clone/index.html`).

> Reconstrução a partir de dados/modelo. Nenhum código-fonte do original foi
> copiado — extraímos schema, curvas e o modelo de coordenadas, e reescrevemos
> o motor.

---

## 1. Mapa dos arquivos do original

O `Arqé.html` salvo só trouxe o **chunk de entrada**. A lógica rica está em
chunks lazy que foram buscados do site (`https://app.arqe.ai/assets/...`).

| Arquivo | O que contém | Usado para |
|---|---|---|
| `Arqé_files/index-Cba5D7He_E0ua.js` | Bundle de entrada. **Registry de presets** (`stack01`, params, flags), **tabela de bezier default por preset** (`g_`), **motor 2D de expressões** (`T_`, parser `E_`/`Gs`) | Schema do Stack + modelo de coordenadas + curva default |
| `assets/easingPresets-*.js` | Evaluador **cubic-bezier** (Newton-Raphson) + curvas nomeadas | Curva de velocidade (Flow, Glide, Smooth...) |
| `assets/videoTiming-*.js` | Resolver de config: `animModel:"step"\|"continuous"`, loop `trimIn/trimOut`, `bezier` por preset | Confirmar modelo stepped + easing por preset |
| `assets/planner-*.js` | Arquétipos do chat "Motion" (expressões `x,y,scale,opacity`). Mostra a forma canônica do **stagger**: `mod(t + u*stagger, 1)` | Modelo de stagger individual por card |
| `assets/App-*.js` / `assets/MobileShell-*.js` | UI dos controles. Confirma `stagger`/`delay` do stack são convertidos para **frames** (×fps) | Semântica dos sliders |
| `assets/demo-assets-*.js` | URLs das imagens de exemplo (CDN DigitalOcean) | Baixamos p/ `clone/img/sample01..10.png` |

**Não localizados** (runtime-gerados/ofuscados): a função imperativa exata que
desenha o Stack (define `window.__animatorRenderAt`). Reconstruímos o
comportamento a partir do modelo + descrição visual.

---

## 2. Dados extraídos do original

### 2.1 Schema do `stack01` (bundle de entrada)
Formato de param: `[min, max, default, step]`.
```
stack01: {
  Scene:     { count[2,20], planeSize, cornerRadius, visible[2,8],
               perspective, zoom, offsetX, offsetY }
  Animation: { cycles, duration, stagger[0,.5], delay[0,5] }
  flag:      stackScene
  hasBezier: true
}
```
`visible` = quantos cards na pilha ao mesmo tempo. `count` = pool de imagens
que **cicla** pela pilha.

### 2.2 Bezier default por preset (tabela `g_`)
```
stack01:     h1(.76, 0)  h2(.24, 1)   → "Smooth"
carousel01:  h1(.33, 0)  h2(0, 1)     → "Glide"
wheel01:     h1(.86,.14) h2(.14,.86)  → "Flow"
spin01:      h1(.7,.101) h2(.3,.899)  → "Sweep"
```

### 2.3 Motor de coordenadas 2D (função `T_` no bundle)
```
h    = min(W,H)/2 * 0.82      // escala das coords normalizadas (x*h)
size = min(W,H)  * 0.16       // tamanho base do plano (× scale)
u    = i/(n-1)                // índice normalizado 0..1
vars por card: { i, n, t, u }  + params
draw: translate(cx + x*h, cy + y*h); rotate; scale; globalAlpha=op; drawImage
ordem: sort por scale crescente (painter — fundo antes da frente)
```

### 2.4 Easing (chunk `easingPresets`)
Cubic-bezier com p0=0, p3=1, resolvido por Newton-Raphson (12 iterações).
Reimplementado em `bezier(x, [x1,y1,x2,y2])` no clone.

### 2.5 Stagger (chunk `planner`)
Forma canônica: cada card desloca o próprio tempo por `u*stagger` →
`mod(t + u*stagger, 1)`. Ou seja, **cards se movem individualmente**, não em bloco.

---

## 3. Lógica aplicada no clone (`renderStack`)

Arquivo: `clone/index.html`, função `renderStack(p, tg, C, tRaw)`.

### 3.1 Ponteiro / passo
```
raw  = frac(tRaw*cycles - delay/dur) * count   // ponteiro 0..count no loop
base = floor(raw)                              // passo atual (inteiro)
s    = frac(raw)                               // progresso dentro do passo 0..1
```
A cada passo a pilha avança **um card**. `count` controla quantas imagens
passam; `visible` controla quantas aparecem.

### 3.2 Slot contínuo por card + STAGGER individual
```
L = (i - base) mod count            // slot inteiro no início do passo (0 = frente)

se L == 0  (card da frente, saindo):
    cs = -clamp(s / exitFrac, 0, 1) // vai de 0 → -1 rápido (exitFrac = 0.35 do passo)

senão (seguidores):
    delay = (L-1) * stagger         // card mais à frente move ANTES
    local = clamp((s - delay) / (1 - delay), 0, 1)
    cs    = L - bezier(local, C)    // eased individual: L → L-1
```
`cs` = slot contínuo. É o que dá o movimento **individual** — cada card seguidor
entra no lugar num tempo diferente (defasado por `stagger`).

### 3.3 Render por `cs`
```
cs < 0        → SAINDO:
                 ex = -cs (0..1); desce opaco e faz fade só no fim:
                 fadeStart = 0.9
                 op = ex < fadeStart ? 1 : 1 - smooth((ex-fadeStart)/(1-fadeStart))
                 y  = dir * ex * stepY        (continua descendo na direção)
                 z  = 3000 (na frente)

0 ≤ cs ≤ vis  → PILHA VISÍVEL:
                 scale = zoom/100 * (1 - cs*0.07)   (frente maior, fundo menor)
                 y     = dir * (-cs * stepY)         (empilha p/ cima atrás)
                 op    = 1
                 z     = 2000 - cs*10

vis < cs ≤ vis+1 → ENTRANDO pelo fundo (staggered):
                 fade-in curto (op = clamp(vis+1-cs, 0, 1))

cs > vis+1    → escondido
```

### 3.4 Constantes de motion
```
sz    = planeSize * 2.4          // tamanho do plano em px
stepY = sz*0.13 + perspective*0.30   // gap vertical entre slots
dir   = direction=="up" ? -1 : 1
```

### 3.5 Comportamento resultante (o que foi calibrado com o usuário)
- Pilha sobe **eased** (arranca/assenta) — o card da frente "perde velocidade".
- Cards de trás **movem individualmente** em sequência (stagger), depois do da
  frente assumir o lugar.
- Card da frente **desce opaco** e só faz **fade rápido nos últimos 10%** da
  descida (`fadeStart=0.9`), depois some. Sem fade desde o topo, sem corte seco.

---

## 4. Pontos de ajuste rápido

| Efeito | Onde |
|---|---|
| Início do fade na descida | `fadeStart` (0.9) |
| Duração da saída | `exitFrac` (0.35 do passo) |
| Defasagem entre cards | slider `stagger` / fórmula `(L-1)*stagger` |
| Gap vertical da pilha | `stepY` |
| Curva de velocidade | seletor de easing (default `Smooth` = bezier do original) |
| Valores iniciais | `REGISTRY.stack01.sections` (planeSize 200, duration 8, cornerRadius 0, perspective 0) |

---

## 5. Como continuar (outras cenas)

Mesmo método: schema no `index` bundle (`slideScene`, `wheelScene`, ...),
bezier default em `g_`, coordenadas via `T_`, stagger via forma do `planner`.
Para as cenas **WebGL** (Orbit, Field, Globe, Spiral): schema em `videoTiming`
(`projectionModel`, `distance`, `sphereRadius`...) e shaders em
`Ring3DScene-*.js` / `DeviceScene-*.js` (ainda não portados).
