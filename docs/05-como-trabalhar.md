# 05. Como trabalhar neste projeto

## Convenções

- **Um arquivo só.** Não quebrar em módulos, não adicionar bundler nem dependência sem pedido
  explícito. O valor do protótipo é copiar `index.html` e funcionar em qualquer lugar.
- **Sem APIs de suporte duvidoso** (`ctx.filter`, `OffscreenCanvas` obrigatório, WebGL). Se
  precisar de blur, usar `boxBlurF`. Se precisar de outra coisa, escrever à mão em TypedArray.
- **Unidades físicas em mm** na interface e nas constantes; converter com `pxPerMm` no ponto
  de uso. Nunca chumbar px que dependam da escala.
- **Determinismo**: aleatoriedade sempre via `mulberry32(seed)` (seeds 7 em `analyze`, 11 em
  `render`, 3 em `quantize`), pra dois renders com os mesmos parâmetros darem a mesma imagem.
- **Índice de pixel** é sempre `i = y*W + x`, e RGBA é `i*4`. Loops de borda começam em 1 e
  terminam em `W-1`/`H-1` quando leem vizinhos.
- **Texto da interface** em português do Brasil, seguindo `_Dev/_docs/escrita-geral.md` e
  `_Dev/Sixtini/ESCRITA.md`: sem travessão, sem emoji, frases curtas, tom B2B direto.
  Comentários de código também sem travessão.
- Ao adicionar um controle: `data-full` se muda geometria; ler em `readParams()`; formatar o
  `<output>` no listener genérico; documentar em [04-parametros.md](04-parametros.md).
- Ao mudar algo do pipeline: atualizar [03-pipeline-bordado.md](03-pipeline-bordado.md) e, se
  mexer em assinaturas ou ordem de funções, a tabela de linhas em
  [02-arquitetura.md](02-arquitetura.md).
- **Toda mudança (qualquer uma) ganha uma entrada no [`../CHANGELOG.md`](../CHANGELOG.md), e todo
  bug corrigido ganha uma entrada em [06-processo.md](06-processo.md) →
  [07-bugs.md](07-bugs.md).** Regra obrigatória, não opcional — ver 06-processo.md pro formato.

## Como testar

Não há testes automatizados. O teste é abrir e olhar. Três jeitos:

### 1. No navegador, à mão

Abrir `index.html#exemplo`, ver o logo de exemplo bordado, passar a lupa pelas letras, testar
"Segurar pra ver o original", trocar tecido e cores. Depois soltar um logo real (PNG com alpha)
e um JPG com fundo branco.

### 2. No console do navegador (depuração)

`window.__sim` é o `state`. Coisas úteis:

```js
__sim.geom.stitches.length                       // total de pontos
__sim.geom.stitches.reduce((a,s)=>(a[s.k]=(a[s.k]||0)+1,a),{})   // por tipo: fill/satin/border
__sim.q.palette; __sim.q.counts                  // cores detectadas e área de cada uma
__sim.work.pxPerMm; __sim.work.W; __sim.work.H   // escala e tamanho do trabalho
__sim.load(dataURL)                              // carregar uma imagem por código (ex.: canvas.toDataURL())
document.querySelector('#status').textContent    // tempo, pontos, cores
```

Exemplo de "JPG sintético" pra testar a remoção de fundo sem arquivo:

```js
const c=document.createElement('canvas'); c.width=700; c.height=400; const x=c.getContext('2d');
x.fillStyle='#fff'; x.fillRect(0,0,700,400);
x.fillStyle='#0a5c36'; x.fillRect(60,60,220,280);
x.fillStyle='#e8a100'; x.font='bold 90px Arial'; x.fillText('LAVA',320,220);
__sim.load(c.toDataURL('image/jpeg',0.9));
```

Pra ver uma máscara intermediária (ex.: `thin` de uma cor), o jeito mais rápido é, dentro de
`analyze`, copiar a `Uint8Array` pra um `ImageData` e `putImageData` num canvas de debug
anexado ao `body`. Não deixar isso no código final.

### 3. Captura automatizada (Edge headless, nesta máquina)

Não existe Chrome instalado, só Edge. Serve pra comparar antes/depois de uma mudança no
pipeline sem depender do painel do navegador:

```powershell
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --headless=new --disable-gpu `
  --hide-scrollbars --no-first-run --user-data-dir="C:\Users\kewin\AppData\Local\Temp\claude\edge-shot" `
  --virtual-time-budget=20000 --window-size=1700,1100 --screenshot="C:\caminho\saida.png" `
  "file:///G:/Meu%20Drive/CLIENTES/_Dev/Sixtini/Simulador/Prototipo%20Bordado/index.html#exemplo"
```

Detalhes que já custaram tempo:
- Rodar pelo **PowerShell**, não pelo Git Bash (o Bash converte o caminho do `--screenshot` e o
  arquivo não é gravado).
- `--user-data-dir` isolado é obrigatório: sem ele o headless disputa com o Edge aberto do
  Kewin. **Nunca** matar `msedge.exe` pra "destravar".
- O `#exemplo` funciona em `file://` direto. No painel de navegador do Claude Code a página é
  carregada como `data:` e o hash se perde; lá é preciso clicar em "Usar logo de exemplo" ou
  rodar `document.querySelector('#sample').click()`.
- Pra ver detalhe, recortar e ampliar um trecho com `System.Drawing` no PowerShell
  (`Graphics.DrawImage` com retângulo de origem) antes de abrir a imagem; a captura inteira em
  1700 px esconde os fios.

## Como depurar problemas típicos

| Sintoma | Onde olhar |
|---|---|
| "Não achei conteúdo no logo" | `prepareSource` removeu tudo: fundo da mesma cor do logo ou tolerância 42 alta demais. Desligar `#rmbg` |
| Cor extra parecida com a borda ("rosa" entre vermelho e branco) | Critério de descarte em `quantize` (borda > 50%, distância ao segmento < 45). Ajustar limiares |
| Cor legítima sumiu | Mesmo critério pegou uma cor real fina, ou área < 0,4%. Subir "Cores de fio" não ajuda; relaxar o limiar |
| Área que devia ser tatami virou cetim radial | Largura local abaixo de `satinMax`, ou a região foi recortada por outra cor sem ser "envolvida" (só cores **posteriores** na ordem por área entram na expansão). Ver `analyze` |
| Leque nas pontas dos traços | Orientação. Ver raios em `orientation` e a escolha por `2*mx` |
| Tecido aparecendo entre fios | **Esperado desde 2026-09-04** se a densidade estiver espaçada (é o pedido do Kewin — ver Etapa 10 em [03-pipeline-bordado.md](03-pipeline-bordado.md)). Só é bug se aparecer com densidade baixa/apertada: conferir se a largura do `stroke` da base (`1,15 * espessura`) ficou menor que o espaçamento real entre pontos vizinhos |
| Lento | Número de cores (cada cor roda chamfer + max filter + 6 blurs + flood fill) e densidade baixa. Tempos normais: 1 a 4 s |
| Lupa desalinhada | `syncLupa` depende do `getBoundingClientRect` do canvas; chamar depois de qualquer mudança de layout |
| Status mostra tempo muito baixo | O tempo exibido é do último `run`; se houve um `render` pendente logo após o `full`, é o tempo dele |

## Armadilhas do código

- `analyze` usa `idx` incrementado **antes** do `slice`: `order.slice(idx)` são as cores
  posteriores à atual. Não trocar por `idx-1`.
- `quantize` cria `counts` com o tamanho de antes do descarte; por isso o `slice` no `return`.
- `maxFilter` é van Herk: a janela tem exatamente `2r+1` e a fórmula `max(h[a], g[b])` só vale
  com esse tamanho. Não "otimizar" com janela diferente sem refazer os blocos.
- `distInside` trata a borda da imagem como distância 1 (por isso a margem de 14% em
  `buildWork`: formas nunca tocam a borda).
- Em `getSprite`, o comprimento do meio é `2P` pra torção fechar entre tiles. Se mudar a
  fórmula da torção, manter `mid` múltiplo do período.
- `render` limpa `spriteCache` sempre. Se um dia separar "mudou cor" de "mudou luz", dá pra
  reaproveitar o cache, mas hoje é barato (algumas centenas de sprites pequenos).
- `loadImage` decide `#rmbg` olhando uma amostra de 200 px; um PNG opaco com fundo será
  tratado como JPG (remove fundo), o que costuma ser o desejado.

## Ideias de evolução (não começadas)

1. **Web Worker**: mover `prepareSource` (parte dos loops), `quantize`, `analyze` e os loops de
   `render` pra um worker com `OffscreenCanvas` onde houver suporte, mantendo fallback.
2. **Mockup em peça**: camada de foto de polo/jaleco com o bordado deformado no peito
   (mapa de deslocamento simples) e escala real pela largura escolhida.
3. **Integração na LP** (Lava Pages, `uniformes.sixtini.com.br`): embutir como bloco, com
   botão "Pedir orçamento com este logo" que anexa o PNG gerado e o PNG original ao formulário.
   Ver skill `lava-pages-design` e a regra do Kewin de preferir edição via painel.
4. **Contagem de pontos de máquina**: estimar pontos reais (cetim: comprimento/densidade;
   tatami: comprimento total / comprimento do ponto) pra dar uma faixa de custo.
5. **Exportar arquivo de bordado**: DST é um formato simples (comandos de deslocamento em
   0,1 mm); a lista `stitches` já tem quase tudo, faltaria ordenar em trajeto e inserir trims.
6. **Melhorar cetim em junções**: seguir o esqueleto (medial axis) em vez do gradiente da
   distância, ou dividir letras em segmentos por curvatura.
7. **Modo "aplique"**: área grande vira tecido recortado com contorno em cetim (comum em
   uniformes), em vez de tatami.
