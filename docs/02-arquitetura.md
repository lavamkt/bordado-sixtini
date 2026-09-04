# 02. Arquitetura

Tudo está em `index.html`. Números de linha abaixo são da versão de 2026-09-04 (821 linhas, depois
de embutir o logo padrão em base64 e adicionar emendas, altura independente e orçamento) e podem
deslocar com edições; use `grep -n "^function \|^async function "` pra reconferir.

## Estrutura do arquivo

| Linhas | Bloco |
|---|---|
| 1 a 69 | `<head>`: meta, título e todo o CSS (variáveis de cor em `:root`, layout em grid, painel lateral, palco, lupa, caixa de orçamento) |
| 70 a 159 | HTML: `<header>`, `<aside>` (painel de controles) e `<section class="palco">` (barra de botões, palco com canvas `#out`, `<img id="orig">` e canvas `#lupa`) |
| 160 em diante | `<script>` com um IIFE `(() => { 'use strict'; ... })()` |

Dentro do script, os blocos estão separados por comentários `// ---------- nome ----------`:

| Bloco | Linhas | Conteúdo |
|---|---|---|
| estado | 168 a 178 | `RES`, helpers `$`, `clamp`, `fmt`, objeto `state`, `window.__sim = state`, `DEFAULT_LOGO_URI` (logo padrão da Sixtini em base64) |
| utilidades numéricas | 179 a 244 | `mulberry32`, `distInside`, `boxBlurF`, `maxFilter`, `orientation`, `discOffsets` |
| preparação da imagem | 245 a 303 | `prepareSource`, `buildWork` |
| quantização de cores | 304 a 357 | `quantize` |
| geração dos pontos | 358 a 480 | `genSatin`, `findComponents`, `planJumps` (emendas), `genTatami`, `analyze` |
| sprites de fio | 481 a 539 | `spriteCache`, `getSprite`, `drawJumpThread` (linha de emenda), `drawStitch` |
| tecido | 540 a 579 | `hexToRgb`, `makeFabric` |
| render | 580 a 666 | `render` (base traçada só sob os pontos, altura independente/estica, desenho das emendas) |
| pipeline | 667 a 721 | `tick`, `setStatus`, `readParams`, `naturalHeightCmFor`, `syncHeightSlider`, `updateBudget`, `run`, `schedule` |
| UI | 722 a 821 | swatches, sliders, presets, upload, logo padrão (auto) e logo de exemplo, download, comparação, lupa, orçamento, `#exemplo` |

## Objeto `state`

```js
const state = {
  img: null,          // HTMLImageElement do logo carregado (fonte original)
  hasAlpha: true,     // a imagem tinha transparência? (decidido em prepareSource)
  src: null,          // canvas recortado na caixa do conteúdo, já com fundo removido
  work: null,         // {W,H,rgba,mask,count,pxPerMm,contentW,contentH} (buildWork)
  q: null,            // {labels,palette,counts} (quantize)  [criado em run()]
  geom: null,         // {stitches,Dop,order,jumps} (analyze)
  palette: [],        // cores de fio atuais, [r,g,b] por índice de cor (editáveis)
  paletteAuto: [],    // cores detectadas, pra o botão "Restaurar"
  result: null,       // não usado hoje
  busy: false,        // run() em andamento
  pending: null,      // 'full' | 'render' enfileirado enquanto busy
  load: loadImage     // atalho pra automação: state.load(dataURL)
};
```

`window.__sim` aponta pro mesmo objeto, então no console do navegador dá pra inspecionar
`__sim.geom.stitches`, `__sim.geom.jumps`, `__sim.q.palette`, `__sim.work.pxPerMm` etc.

## Fluxo de dados

```
arquivo / paste / exemplo / logo padrão (auto)
        |
        v
   loadImage(dataURL)  -> state.img, decide #rmbg (liga se não tem alpha) -> run('full')
        |
        v
run('full')
   prepareSource(img, rmbg)  -> state.src   (canvas RGBA recortado, fundo removido)
   buildWork(src, p)         -> state.work  (canvas de trabalho com margem, máscara, px/mm)
   se aspectLock: sincroniza a altura exibida com naturalHeightCmFor(work, widthCm)
   quantize(work, K, seed)   -> state.q     (rótulo por pixel + paleta + contagens)
   renderSwatches()          -> UI dos fios
   analyze(work, q, p)       -> state.geom  (lista de pontos + jumps candidatos + mapa de distância)
   spriteCache.clear()
run('render')  (ou continuação do full)
   render(work, q, geom, p)  -> desenha em #out (compõe, estica se altura for independente, emendas)
   setStatus(...)            -> "Pronto em X s. N pontos, K cores, P px/mm."
   updateBudget()             -> "R$ X,XX / N pontos × R$ preço/ponto"
```

Regra de recomputação: cada controle da interface tem ou não o atributo `data-full`.

- **Com `data-full`** (largura, cores, ângulo, densidade, comprimento do ponto, largura máxima
  do cetim, contorno e sua largura, remover fundo): muda geometria, dispara `schedule('full')`.
- **Sem** (espessura do fio, relevo, brilho, luz, tecido, cor do tecido, cor de cada fio,
  **emendas entre letras**, **altura do bordado**): só muda a pintura/composição final, dispara
  `schedule('render')`. Altura funciona assim de propósito — ela não muda a geometria dos pontos,
  só estica o composto final (ver Etapa 11 em [03-pipeline-bordado.md](03-pipeline-bordado.md)),
  então recalcular tudo de novo seria desperdício.
- **`#pricePerPoint`** não passa por `readParams`/`schedule` nenhum — só chama `updateBudget()`
  direto, porque não afeta o bordado, só um cálculo aritmético sobre `geom.stitches.length` já
  pronto.
- **`#aspectLock`**: ao MARCAR (travar), sincroniza a altura pro valor natural e desenha de novo
  (`schedule('render')`); ao desmarcar, não dispara nada (o valor da altura já está igual ao
  natural nesse instante, então destravar sozinho não muda a imagem).

`schedule(kind)` faz debounce de 250 ms. `run(kind)` recusa reentrância: se já está `busy`,
guarda em `state.pending` (um `full` pendente nunca é rebaixado pra `render`) e, ao terminar,
roda de novo uma vez.

## Estruturas de dados centrais

Todas as imagens de trabalho são planas, indexadas por `i = y*W + x`:

| Nome | Tipo | Significado |
|---|---|---|
| `work.rgba` | `Uint8ClampedArray` (4 por pixel) | pixels do canvas de trabalho |
| `work.mask` | `Uint8Array` 0/1 | pixel opaco (alpha > 127) |
| `q.labels` | `Int16Array` | índice da cor do fio, ou -1 se transparente |
| `Dop`, `Dc` | `Float32Array` | distância em px até a borda (dentro da máscara; 0 fora) |
| `mx` | `Float32Array` | máximo de `Dc` numa janela quadrada (largura local) |
| `thin`, `ring`, `fill` | `Uint8Array` 0/1 | regiões de cetim, contorno e tatami de uma cor |
| `orient` | `{levels:[{c2,s2}], sel:Uint8Array, at(i)}` | campo de orientação em 3 escalas |
| `geom.stitches` | `Array<{x1,y1,x2,y2,c,k}>` | um ponto = segmento; `c` índice de cor; `k` = `'fill'`, `'satin'` ou `'border'` |
| `geom.jumps` | `Array<{x1,y1,x2,y2,c,dist}>` | candidato de emenda entre dois componentes da mesma cor `c`, com a distância real `dist` (px) entre eles — filtrado no `render` pelo slider "Emendas entre letras" |

Canvas de trabalho: lado maior do logo = `RES*(1-2*0.14)` = 720 px, mais margem de 140 px de
cada lado (`buildWork`). Então `W` fica em torno de 1000 e `H` depende da proporção do logo.
**Importante:** `W`/`H`/`pxPerMm` continuam sempre baseados só na LARGURA escolhida — a altura
independente (ver 03, Etapa 11) é um esticamento aplicado só no composto final em `render`, nunca
recalcula essas estruturas.

## Mapa de funções

| Função | Linha | Assinatura e papel |
|---|---|---|
| `mulberry32(seed)` | 180 | PRNG determinístico; usado pra jitter reprodutível |
| `distInside(mask,W,H)` | 183 | Transformada de distância (chamfer 1 / 1,414) dentro da máscara, dois passes |
| `boxBlurF(src,W,H,r)` | 202 | Box blur separável em `Float32Array` com somas correntes |
| `maxFilter(src,W,H,r)` | 215 | Filtro de máximo separável (van Herk / Gil-Werman), janela `2r+1` |
| `orientation(D,W,H,radii,mx)` | 231 | Campo de orientação por ângulo duplo do gradiente de `D`, suavizado em cada raio de `radii`; `sel[i]` escolhe a escala mais próxima de `2*mx[i]`; `at(i)` devolve o ângulo |
| `discOffsets(r)` | 243 | Lista de deslocamentos `[dx,dy,...]` de um disco de raio `r` |
| `prepareSource(img,removeBg)` | 246 | Reduz pra 1000 px, remove fundo por flood fill se pedido, recorta no conteúdo. Retorna canvas ou `null` |
| `buildWork(src,p)` | 288 | Canvas de trabalho com margem, `rgba`, `mask`, `pxPerMm` |
| `quantize(work,K,seed)` | 305 | k-means++ em amostra de 30 mil pixels, fusão de centros, descarte de clusters de borda, filtro de moda |
| `genSatin(region,orient,W,H,spacing,maxLen,ci,kind,out,rng)` | 359 | Pontos de cetim por amostragem + marcha na direção do campo + mapa de cobertura |
| `findComponents(mask,W,H)` | 377 | Componentes conectados (4-vizinhos) de uma máscara; devolve centróide e pixels de contorno de cada um. Usado pra achar letras/elementos separados de uma mesma cor |
| `planJumps(comps,ci,out)` | 404 | Caminho guloso pelo vizinho mais próximo entre os componentes de uma cor; empurra em `out` o par de pontos de contorno mais próximos entre cada dupla consecutiva (candidatos de emenda) |
| `genTatami(region,W,H,angleDeg,spacing,L,ci,out,rng)` | 423 | Linhas paralelas no ângulo, quebradas em pontos de comprimento `L` com escalonamento |
| `analyze(work,q,p)` | 438 | Por cor (maior área primeiro): acha componentes/emendas na máscara crua, monta região com camadas, classifica cetim/contorno/tatami, gera pontos |
| `getSprite(rgb,angle,variant,w,light,sheen)` | 483 | Sprite de fio (capa + meio + capa) iluminado pra um dos 32 ângulos; cacheado |
| `drawJumpThread(ctx,j,rgb,pxPerMm,rng)` | 515 | Desenha uma linha de emenda (sombra + fio escurecido + brilho fino) com leve folga entre dois pontos |
| `drawStitch(ctx,st,sp)` | 530 | Desenha um ponto rotacionado: capa esquerda, meio repetido, capa direita |
| `makeFabric(type,hex,pxPerMm,W,H,rng)` | 542 | Tecido procedural: cor base + padrão por tipo + grão em `overlay` + vinheta |
| `render(work,q,geom,p)` | 581 | Compõe tudo (inclusive o esticamento de altura independente e as emendas) no canvas `#out` (ver 03) |
| `readParams()` | 670 | Lê todos os controles num objeto `p` |
| `naturalHeightCmFor(work,widthCm)` | 678 | Altura proporcional (cm) do conteúdo recortado pra uma dada largura |
| `syncHeightSlider(cm)` | 679 | Atualiza o slider e o `<output>` de altura sem disparar recomputação |
| `updateBudget()` | 681 | Lê `geom.stitches.length` e `#pricePerPoint`, escreve o total em `#budgetValue`/`#budgetDetail` |
| `run(kind)` | 688 | Orquestra o pipeline; trata `busy`/`pending`; sincroniza altura; atualiza status, botões e orçamento |
| `renderSwatches()` | 723 | Cria os `<input type=color>` dos fios e o botão Restaurar |
| `loadImage(src)` | 758 | Cria `Image`, detecta alpha numa amostra 200 px, define `#rmbg`, chama `run('full')` |
| `handleFile(f)` | 769 | `FileReader` -> `loadImage` |
| `syncLupa()` | 799 | Alinha o canvas da lupa ao canvas de saída (leva em conta `devicePixelRatio`) |

## Elementos de interface (ids)

| Id | Elemento |
|---|---|
| `#drop`, `#file` | área de soltar e input de arquivo |
| `#rmbg` | checkbox "Remover fundo" |
| `#sample` | botão do logo de exemplo |
| `#widthCm #aspectLock #heightCm #k #angle #density #thread #stitch #relief #seam` | sliders principais (largura, trava de proporção, altura, cores, ângulo, densidade, espessura, comprimento do ponto, relevo, emendas entre letras) |
| `#satinMax #border #borderW #sheen #light` | ajustes finos (dentro de `<details>`) |
| `#swatches` | cores dos fios |
| `#fabric #fabricColor #presets` | tecido |
| `#pricePerPoint #budgetValue #budgetDetail` | orçamento (preço por ponto e total calculado) |
| `#apply #download #compare #lupaOn #status` | barra de ações |
| `#stage #empty #out #orig #lupa` | palco, mensagem vazia, canvas de saída, imagem original, lupa |

Cada `.row input[type=range]` tem um `<output>` irmão que é formatado no listener genérico
(bloco de `document.querySelectorAll('.row input[type=range]')` em `readParams`/UI), com regras
por id (cm, graus, %, mm com 1 ou 2 casas). `#heightCm` reusa o mesmo formato de `#widthCm` (cm);
`#seam` reusa o de `#sheen` (%).
