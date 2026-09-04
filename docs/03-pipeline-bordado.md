# 03. Pipeline do bordado, etapa por etapa

Este é o coração do projeto. Cada seção descreve o algoritmo, por que foi escolhido e onde está
no código. Unidades: tudo que é "físico" está em mm e é convertido pra px por `pxPerMm`.

## Escala física

`pxPerMm = larguraDoConteudoEmPx / (widthCm * 10)`, calculado em `buildWork`. O conteúdo tem
720 px de lado maior, então um logo de 8 cm dá 9 px/mm; de 4 cm dá 18 px/mm (fios maiores em
relação ao logo, como no bordado real de um logo pequeno). Isso é o que faz o resultado parecer
bordado de verdade: fio de 0,45 mm, densidade de 0,38 mm, ponto de 3,5 mm, cetim até 7 mm.

## Etapa 1: preparo da imagem (`prepareSource`, `buildWork`)

1. Desenha o logo com lado maior de 1000 px num canvas temporário.
2. Detecta se tem alpha (algum pixel com alpha < 250). Guarda em `state.hasAlpha`.
3. Se "Remover fundo" está ligado:
   - Cor de referência = média dos 4 cantos se eles concordam (distância RGB < 40), senão o
     canto superior esquerdo.
   - Flood fill (pilha explícita em `Int32Array`, 4-vizinhança) a partir de **todos** os pixels
     da borda cuja cor está a menos de 42 da referência. Só pixels parecidos com a referência
     são visitados. Isso mantém áreas internas da mesma cor do fundo (ex.: branco dentro de um
     círculo fechado) porque elas não estão conectadas à borda.
   - Erosão de 1 px: pixels adjacentes ao fundo removido e com cor a menos de 2,2x a tolerância
     também viram transparentes (tira o halo antisserrilhado).
4. Recorta na caixa delimitadora dos pixels com alpha > 127. Se não sobra nada, devolve `null`
   e `run` mostra "Não achei conteúdo no logo".
5. `buildWork` redesenha esse recorte com lado maior de 720 px num canvas com margem de 140 px
   (a margem serve pra sombra e franzido) e extrai `rgba`, `mask` e `pxPerMm`.

## Etapa 2: cores dos fios (`quantize`)

Objetivo: reduzir o logo a K cores chapadas, uma por fio, sem ruído de antisserrilhado.

1. Amostra até 30 mil pixels opacos, espaçados uniformemente.
2. **k-means++** (inicialização proporcional à distância ao quadrado) e até 14 iterações de
   Lloyd em RGB. Para quando os centros se movem menos de 0,5.
3. **Fusão**: centros a menos de 26 de distância RGB viram um só (tons de borda).
4. Atribui todos os pixels opacos ao centro mais próximo (`assignAll`), contando por cor.
5. **Descarte** de clusters:
   - área menor que 0,4% dos pixels opacos, ou
   - cluster "de borda": mais de 50% dos seus pixels têm um 4-vizinho de outro rótulo **e** sua
     cor fica a menos de 45 do segmento entre duas outras cores (é uma mistura de vizinhas,
     típico de antisserrilhado). Só se sobrarem mais de 2 cores.
   Depois do descarte, reatribui tudo.
6. **Filtro de moda 3x3** duas vezes sobre os rótulos (só pixels opacos) pra apagar pontinhos.
7. Devolve `labels` (Int16, -1 = transparente), `palette` (inteiros 0 a 255) e `counts`.

Observação: a paleta detectada vai pra `state.paletteAuto`; a editável, pra `state.palette`.
`render` usa sempre `state.palette`.

## Etapa 3: regiões por cor e camadas (`analyze`, primeira metade)

Ordem: cores por área decrescente (`order`). A de maior área é bordada primeiro; as menores
ficam por cima, como um bordador faz (fundo, depois detalhe).

Pra cada cor `ci`, a máscara `M` começa como "pixels com rótulo ci" — é nesse ponto, **antes**
de qualquer expansão, que os candidatos de emenda são calculados (`findComponents`/`planJumps`,
ver Etapa 11) — e só depois é **expandida** com pixels de cores posteriores na ordem (`later`):

- **Envolvidos**: flood fill a partir da borda da imagem por todos os pixels que **não** são de
  `ci`. O que o flood não alcança está cercado por `ci`. Pixels não alcançados, opacos e de cor
  posterior entram em `M`. Exemplo: círculo vermelho com "S" branco por cima. O vermelho vira um
  disco inteiro (tatami + contorno) e o "S" é bordado em cetim por cima. Buracos transparentes
  (miolo de um "O") não entram, porque só pixels opacos são adicionados.
- **Sobreposição de 0,5 mm** (`underlap`): pixels de cores posteriores a menos de 0,5 mm da
  borda de `M` (distância calculada na máscara invertida) também entram. Evita tecido aparecendo
  na emenda entre duas cores.

A última cor da ordem não expande (não há "posterior").

## Etapa 4: cetim, contorno ou tatami (`analyze`, segunda metade)

1. `Dc = distInside(M)`: distância até a borda da região da cor.
2. `mx = maxFilter(Dc, r)` com `r = satinMax/2` em px. Se numa janela quadrada de raio `r`
   ao redor do pixel nenhum ponto tem distância maior ou igual a `r`, a forma ali é mais
   estreita que `satinMax`. Regra:
   - `mx[i] < satinHalf` -> **`thin`** (cetim)
   - senão, se está a menos de `borderW` da borda considerada -> **`ring`** (contorno em cetim)
   - senão -> **`fill`** (tatami)

   A "borda considerada" depende do modo de contorno: `outer` usa `Dop` (distância na máscara
   global, logo só a borda externa do bordado), `all` usa `Dc` (também as emendas entre
   cores), `none` não gera anel.
3. Gera pontos nesta ordem, pra que fiquem empilhados certo: tatami do `fill`, cetim do `thin`,
   cetim do `ring`.

## Etapa 5: orientação do cetim (`orientation`)

Pontos de cetim atravessam o traço, perpendiculares ao eixo dele. A direção "atravessar" é o
gradiente da distância `Dc` (aponta da borda pro meio). Como o gradiente troca de sinal no eixo
central, usa-se **ângulo duplo**: `c2 = gx² - gy²`, `s2 = 2 gx gy`, que são iguais nos dois
lados. Suavizando `c2` e `s2` com box blur e tirando `0,5 * atan2(s2, c2)` sai a orientação.

Problema clássico: nas pontas de um traço, o gradiente aponta pra ponta e os pontos viram
"leque". Solução adotada: três suavizações com raios diferentes
(`[1,8 * densidade, 0,6 * satinHalf, 1,2 * satinHalf]`), e cada pixel usa a escala cujo raio é
mais próximo da **largura local** `2 * mx[i]`. Com raio da ordem da largura do traço, as
laterais longas dominam a votação até a ponta e os pontos ficam perpendiculares. `sel[i]`
guarda a escala escolhida; `orient.at(i)` devolve o ângulo.

## Etapa 6: geração dos pontos de cetim (`genSatin`)

1. Varre uma grade com passo `densidade/3` px (com jitter aleatório de meio passo).
2. Se o ponto está na região e ainda não está **coberto**, pega a orientação, marcha 1 px por
   vez nos dois sentidos até sair da região ou atingir `maxLen`. As duas pontas são o segmento.
3. Marca como coberto um "tubo" de raio `0,85 * densidade` ao longo do segmento (disco de
   `discOffsets` carimbado a cada px). Amostras seguintes dentro do tubo são ignoradas.
   Resultado: espaçamento entre pontos entre 0,85 e 1,2 vezes a densidade, seguindo curvas.
4. Segmentos com menos de 1,5 px são descartados (mas ainda marcam cobertura).

`maxLen` é `2,6 * satinHalf` pro `thin` e `2,5 * borderW` pro `ring`.

## Etapa 7: geração do tatami (`genTatami`)

1. Eixo do preenchimento no ângulo escolhido; `d` = direção da linha, `n` = normal.
2. Linhas paralelas espaçadas pela densidade, cobrindo o canvas inteiro (raio `R = hypot(W,H)/2`).
3. Cada linha é percorrida px a px; trechos contínuos dentro da região viram "runs".
4. Cada run é quebrado em pontos de comprimento `L` com **escalonamento** por linha
   (`stag = (j * 0,37 mod 1) * L`), que é o padrão de agulha do tatami e evita as marcas
   alinhadas. Quebras a menos de 1 px das pontas são puladas.
5. Jitter perpendicular de ±0,35 px na origem de cada ponto pra não ficar sintético.

## Etapa 8: sprites de fio (`getSprite`, `drawStitch`)

Um sprite é um fio horizontal com três partes: capa esquerda (semicírculo), meio de comprimento
`mid = 2P` (com `P = 0,8 * espessura`, período da torção) e capa direita. A altura é a
espessura mais 2 px de folga. Pra cada pixel:

- `dist` até o eixo (ou até o centro da capa), cobertura antisserrilhada `clamp(r - dist + 0,5)`.
- Normal de cilindro `(nt, nz)` = `(dy/r, sqrt(1 - q²))`.
- Luz no espaço do sprite: componente lateral `ll = L · perp(angle)`, vertical `lz`.
- Difusa `max(0, nt*ll + nz*lz)`, especular `pow(max(0, n·h), 28) * sheen` (meio-vetor com
  a câmera em `(0,0,1)`).
- Torção: `1 + 0,10 * sin(2π (x + 1,7 y) / P)`, listras diagonais contínuas entre tiles.
- Cor final `rgb * (0,28 + 0,82 * difusa) * torção * brilhoDaVariante + branco * spec * 0,5`.

Há 32 baldes de ângulo e 3 variantes (largura 0,93/1/1,07, brilho 0,95/1/1,05) por cor, em
`spriteCache` (chave `"r,g,b|balde|variante"`). O cache é limpo em cada `run` porque cor do
fio, espessura, luz e brilho podem ter mudado.

`drawStitch` translada pro `(x1,y1)`, rotaciona pro ângulo real do segmento e desenha: capa
esquerda em `[-cap, 0]`, o meio repetido em fatias de `mid` até o comprimento, capa direita em
`[len, len+cap]`. As capas passam meia espessura além das pontas, como o fio real que abraça o
furo da agulha.

## Etapa 9: tecido (`makeFabric`)

Canvas do tamanho do trabalho: cor base sólida, depois um `createPattern` de um tile pequeno
por tipo (tamanhos derivados de `pxPerMm` pra manter a escala física):

| Tipo | Tile |
|---|---|
| `pique` | célula de 0,85 mm, 5 "gomos" com gradiente radial (luz em cima à esquerda, sombra embaixo à direita), em colmeia |
| `sarja` e `jeans` | diagonais a 45° com período 0,28 mm (escura + clara deslocada) |
| `malha` | costelas verticais de 0,36 mm com uma linha horizontal fraca |
| `tricoline` | xadrez de 0,16 mm (trama plana) |

Por cima: grão em `overlay` (tile 160 px de ruído; mais forte no jeans) e uma vinheta radial
(claro no centro, escuro nas bordas).

## Etapa 10: composição final (`render`)

1. Tecido -> `getImageData`. Pra cada pixel:
   - **Sombra projetada**: máscara global borrada duas vezes (raios 0,35 mm e 0,25 mm),
     deslocada 0,7 mm no sentido oposto à luz, escurece até 55%.
   - **Franzido**: fora do bordado, escurece até 9% numa faixa de 2,2 mm (distância da máscara
     invertida `Dout`), com ruído multiplicativo.
2. Camada de fios (canvas transparente):
   - **Base** opaca com a cor de cada fio a 50% de brilho em todos os pixels rotulados (não
     deixa o tecido vazar entre fios).
   - Todos os `stitches` na ordem gerada, com `getSprite`/`drawStitch`. A cada 4096 pontos há
     um `await tick()` pra atualizar o status.
   - **Furos de agulha**: um disco escuro de raio `0,16 * espessura` no fim de cada ponto de
     tatami.
   - **Relevo**: mapa de altura `H = smoothstep(min(Dop, pad) / pad)` com `pad = relevo` em px,
     borrado com raio 1. Normal `(-gx*k, -gy*k, 1)` com `k = 1,3 * pad`. Multiplica RGB por
     `0,62 + 0,55 * max(0, n·L)`. Só onde a camada tem alpha. Com relevo 0 a etapa é pulada.
3. Desenha tecido e depois a camada de fios num canvas intermediário `composed` (mesmo `W,H` do
   trabalho) — é esse composto que a Etapa 11 estica, e é sobre `#out` já esticado que a Etapa 12
   desenha as emendas.

Luz: ângulo `light` em graus, `L = normalize(cos, sin, 0,85)`. 315° = de cima à esquerda.

## Etapa 11: emendas entre elementos (`findComponents`, `planJumps`, `drawJumpThread`)

No bordado real, quando a máquina termina um elemento (ex.: a letra "S") e o próximo da mesma cor
está perto (ex.: o "I" ao lado), muitas vezes ela **não corta a linha** — deixa um fio fino
esticado entre os dois, por cima do tecido. É uma imperfeição real e comum, principalmente em
texto com letras separadas. Este slider ("Emendas entre letras") simula isso.

1. **Componentes** (`findComponents`, chamada em `analyze` logo que a máscara `M` de uma cor é
   criada, ainda sem a expansão de camadas da Etapa 3): flood fill 4-vizinhos sobre `M`, cada
   "ilha" de pixels vira um componente com centróide e lista de pixels de contorno (só os pixels
   com algum vizinho fora da máscara — suficiente pra medir distância entre formas sem guardar a
   forma inteira). Componentes com menos de 8 pixels são ignorados (ruído).
2. **Caminho** (`planJumps`): se a cor tem mais de um componente, um caminho guloso pelo vizinho
   mais próximo (por centróide) aproxima a ordem que um digitalizador seguiria (ex.: S -> I -> X
   -> T -> I -> N -> I, da esquerda pra direita). Entre cada par consecutivo do caminho, acha o
   par de pontos de contorno mais próximo entre os dois componentes (amostra até 120 pontos de
   cada lado — suficiente pra um logo com poucas dezenas de componentes, rápido o bastante pra
   não precisar de `full` toda vez). O resultado vira um candidato `{x1,y1,x2,y2,c,dist}` em
   `geom.jumps`, com `dist` = distância real em px entre as formas.
3. **Filtro por intensidade** (em `render`, não em `analyze` — por isso o slider dispara só
   `render`, não `full`): `maxJumpPx = (2 + intensidade/100 * 38) * pxPerMm`, ou seja, de 2 mm
   (0%) a 40 mm (100%). Só os candidatos com `dist <= maxJumpPx` viram linha desenhada. Cores
   diferentes nunca se conectam (o candidato já nasce por cor). Em 0% nada é desenhado — logo
   comportamento idêntico ao de antes desse recurso existir.
4. **Desenho** (`drawJumpThread`): três traços com `quadraticCurveTo` (sombra sutil por baixo,
   fio na cor do fio escurecida 45%, brilho fino por cima), com uma leve folga perpendicular ao
   segmento (`sag`, proporcional ao comprimento, sinal aleatório via `mulberry32(13)`) pra não
   ficar uma linha reta e sintética — fio de verdade nunca fica perfeitamente esticado.

## Etapa 12: altura independente e orçamento

**Altura**: a largura (`widthCm`) sempre define a escala física (`pxPerMm`) e a geometria inteira
(igual antes). A altura só entra no fim de `render`: calcula a altura "natural" (proporcional,
`naturalHeightCm = contentH/contentW * widthCm`) e um fator `vScale`:

- **"Manter proporção do logo" ligado** (padrão): `vScale = 1`, sem esticar — é o comportamento
  de sempre, e o slider de altura fica desabilitado, só mostrando o valor natural (sincronizado em
  `run`, logo depois do `buildWork`, via `naturalHeightCmFor`/`syncHeightSlider`).
- **Desligado**: `vScale = clamp(heightCm / naturalHeightCm, 0.2, 5)`. O canvas composto (Etapa
  10) é desenhado em `#out` com a altura de destino igual a `H * vScale` (`drawImage` com
  destino de tamanho diferente da origem — é um esticamento simples, não uma repetição de
  pipeline). Simula uma situação real: bordado precisando caber numa altura fixa (bolso, aba)
  diferente da proporção original do logo.

As emendas (Etapa 11) são desenhadas **depois** do esticamento, direto no `#out` já esticado —
por isso suas coordenadas Y (não X, só a largura nunca estica) são multiplicadas por `vScale` na
hora de desenhar, senão ficariam desalinhadas com o bordado embaixo.

**Orçamento**: não faz parte da simulação visual — é só `updateBudget()` multiplicando
`geom.stitches.length` (já calculado pela Etapa 6/7) pelo valor de `#pricePerPoint`. Roda a cada
`run()` bem-sucedido (a contagem de pontos pode mudar) e a cada edição do preço (não mexe no
canvas, não precisa de `schedule`/debounce).
