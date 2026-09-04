# 04. Parâmetros e controles

Todos os controles são lidos por `readParams()` num objeto `p`. A coluna "Dispara" diz se
mexer no controle refaz a geometria (`full`) ou só a pintura (`render`). Ver regra em
[02-arquitetura.md](02-arquitetura.md#fluxo-de-dados).

## Logo

| Controle | Id | Padrão | Dispara | Efeito |
|---|---|---|---|---|
| Arquivo / soltar / colar | `#file`, `#drop`, `paste` | | `full` | Carrega em `state.img` e roda tudo |
| Remover fundo (cor das bordas) | `#rmbg` | ligado se a imagem não tem alpha | `full` | Flood fill a partir das bordas com tolerância 42 + erosão de 1 px (`prepareSource`) |
| Usar logo de exemplo | `#sample` | | `full` | Desenha um logo de 3 cores num canvas 900x520 e carrega |

## Bordado

| Controle | Id | Faixa | Padrão | Dispara | Efeito |
|---|---|---|---|---|---|
| Largura do bordado | `widthCm` | 3 a 25 cm | 8 cm | `full` | Define `pxPerMm`; todos os outros mm dependem dele. Logo menor = fios maiores em relação ao logo |
| Manter proporção do logo | `aspectLock` | ligado/desligado | ligado | `render`* | Trava a altura no valor proporcional à largura. *Só dispara `render` ao MARCAR (resincroniza); desmarcar não muda nada na hora |
| Altura do bordado | `heightCm` | 3 a 25 cm | igual à largura, sincronizado depois do 1º `full` | `render` | Só ativo com a proporção destravada. Diferente da altura natural, estica o resultado final na vertical (Etapa 12 em [03](03-pipeline-bordado.md)) |
| Cores de fio | `k` | 1 a 8 | 4 | `full` | K máximo do k-means. O resultado pode ter menos cores (fusão e descarte) |
| Ângulo do preenchimento | `angle` | 0 a 180° | 45° | `full` | Direção das linhas do tatami |
| Densidade (espaço entre fios) | `density` | 0,25 a 0,6 mm | 0,38 mm | `full` | Espaçamento entre linhas de tatami e entre pontos de cetim. Menor = mais pontos, mais lento. Valores altos deixam o tecido visível nos vãos entre os fios (a base só cobre a vizinhança de cada ponto, não a região inteira — ver Etapa 10 em [03](03-pipeline-bordado.md)) |
| Espessura do fio | `thread` | 0,3 a 0,7 mm | 0,45 mm | `render` | Altura do sprite. Maior que a densidade = fios se sobrepõem (esperado) |
| Comprimento do ponto | `stitch` | 2 a 6 mm | 3,5 mm | `full` | Comprimento `L` dos pontos de tatami |
| Relevo | `relief` | 0 a 2,5 mm | 1,0 mm | `render` | Altura `pad` do mapa de altura. 0 desliga a etapa |
| Emendas entre letras | `seam` | 0 a 100% | 30% | `render` | Distância máxima (2 a 40 mm) até onde um "pulo" de linha entre dois elementos da mesma cor fica visível em vez de aparado. 0% = tudo aparado. Candidatos calculados uma vez no `full` (Etapa 11 em [03](03-pipeline-bordado.md)), filtrados a cada `render` |

## Ajustes finos (dentro de `<details>`)

| Controle | Id | Faixa | Padrão | Dispara | Efeito |
|---|---|---|---|---|---|
| Largura máxima do cetim | `satinMax` | 3 a 12 mm | 7 mm | `full` | Forma mais estreita que isso vira cetim; mais larga vira tatami. Também define os raios grandes da orientação |
| Contorno em cetim | `border` | `none` / `outer` / `all` | `outer` | `full` | Anel de cetim ao redor das áreas de tatami: nenhum, só na borda externa do bordado, ou também nas emendas entre cores |
| Largura do contorno | `borderW` | 0,6 a 3 mm | 1,0 mm | `full` | Largura do anel |
| Brilho do fio | `sheen` | 0 a 100% | 55% | `render` | Intensidade do reflexo especular no sprite |
| Direção da luz | `light` | 0 a 360° | 315° | `render` | Ângulo da luz no plano (0 = da direita, 90 = de baixo, 315 = de cima à esquerda). Afeta sprites, relevo e sombra |

## Fios (cores detectadas)

Um `<input type=color>` por cor em `state.palette`. Mudar dispara `render` (a geometria não
depende da cor). "Restaurar" volta pra `state.paletteAuto`. Os índices são os mesmos de
`q.labels` e de `stitch.c`.

## Tecido

| Controle | Id | Opções / padrão | Dispara |
|---|---|---|---|
| Tipo | `fabric` | `pique` (padrão), `sarja`, `malha`, `tricoline`, `jeans` | `render` |
| Cor | `fabricColor` | `#1d2a44` (marinho) | `render` |
| Presets | `#presets` | Branco `#f4f4f1`, Cinza mescla `#9a9ca1`, Marinho `#1d2a44`, Preto `#1b1b1d`, Vermelho `#a8202a`, Verde `#1f6b3a`, Royal `#1d4f9c`, Bege `#d9c7a5` | `render` |

## Orçamento

| Controle | Id | Padrão | Efeito |
|---|---|---|---|
| Preço por ponto | `pricePerPoint` | R$ 0,02 | Multiplica por `geom.stitches.length` em `updateBudget()`. Não afeta o bordado, só o cálculo — não passa por `readParams`/`schedule` |

Saída em `#budgetValue` (total em R$) e `#budgetDetail` (contagem de pontos × preço).
`updateBudget()` roda ao fim de todo `run()` bem-sucedido e a cada edição do preço.

## Barra de ações

| Botão | Id | O que faz |
|---|---|---|
| Aplicar bordado | `#apply` | `run('full')` imediato (sem debounce) |
| Baixar PNG | `#download` | `#out.toBlob` -> link temporário `bordado-simulado.png` na resolução de trabalho |
| Segurar pra ver o original | `#compare` | Enquanto pressionado, mostra `<img id="orig">` (com xadrez de transparência) por cima do canvas |
| Lupa | `#lupaOn` | Liga/desliga o círculo de zoom 3x (raio 110 px CSS) que segue o mouse sobre o canvas |

## Constantes internas (não expostas na interface)

| Onde | Valor | Significado |
|---|---|---|
| `RES` | 1000 | Lado maior do canvas de preparo e do trabalho |
| `buildWork` `margin` | 0,14 | Fração de `RES` de margem em cada lado |
| `quantize` | 30000 amostras, 14 iterações, fusão < 26, descarte < 0,4% de área, cluster de borda > 50% e distância ao segmento < 45, 2 passes de moda | Limpeza da paleta |
| `analyze` `underlap` | 0,5 mm | Sobreposição da cor de baixo sob a vizinha |
| `orientation` raios | `[1,8·densidade, 0,6·satinHalf, 1,2·satinHalf]` (mínimos 3, 6, 9 px) | Escalas de suavização |
| `genSatin` | passo `densidade/3`, cobertura `0,85·densidade`, mínimo 1,5 px | Espaçamento dos pontos de cetim |
| `genTatami` | escalonamento `j·0,37 mod 1`, jitter ±0,35 px | Padrão de agulha |
| `getSprite` | 32 ângulos, 3 variantes, especular expoente 28, torção 10% | Aparência do fio |
| `render` sombra | blur 0,35 mm + 0,25 mm, deslocamento 0,7 mm, 55% | Sombra projetada |
| `render` franzido | 2,2 mm, até 9% | Escurecimento em volta |
| `render` relevo | `k = 1,3·pad`, `0,62 + 0,55·(n·L)` | Iluminação do mapa de altura |
| `findComponents` | mínimo 8 px por componente | Ignora ruído de antisserrilhado como "letra" |
| `planJumps` | até 120 pontos de contorno amostrados por lado | Custo do par mais próximo entre dois componentes |
| `render` emendas | `maxJumpPx = (2 + intensidade/100·38) mm` | Faixa de 2 a 40 mm conforme o slider "Emendas entre letras" |
| `render` altura | `vScale = clamp(heightCm/naturalHeightCm, 0.2, 5)` | Limite do esticamento vertical independente |
| `schedule` | 250 ms | Debounce |
| `render` | `await tick()` a cada 4096 pontos | Respiro da UI |

Pra expor uma dessas constantes na interface: criar um `.row` com `input[type=range]` (com
`data-full` se muda geometria), adicionar a leitura em `readParams()`, a formatação do
`<output>` no listener genérico dos sliders e usar `p.nome` no ponto certo.
