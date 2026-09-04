# 01. Visão geral

## O que é

Uma página web que recebe a imagem de um logo (PNG, JPG, WebP, SVG, GIF) e mostra como ele
ficaria **bordado em tecido**: pontos de cetim nos traços finos, preenchimento tatami nas áreas
largas, contorno em cetim, relevo, brilho de fio, sombra e textura de tecido (piquê, sarja,
malha, tricoline, jeans) na cor escolhida.

É um **protótipo** feito em 2026-09-04 pela Lava pra Sixtini (cliente, uniformes profissionais
B2B). A ideia de produto é virar ferramenta de conversão na landing page de uniformes
(`uniformes.sixtini.com.br`): o comprador sobe o logo da empresa, vê o bordado e pede orçamento.
Hoje não está integrado a nada: é um arquivo que abre sozinho.

## Stack

| Item | Escolha | Motivo |
|---|---|---|
| Linguagem | JavaScript puro (ES2020), HTML5, CSS3 | Rodar em qualquer navegador sem build |
| Renderização | Canvas 2D (`CanvasRenderingContext2D`) | Suporte universal; sem WebGL pra manter simples |
| Dependências | Nenhuma. Sem npm, sem bundler, sem CDN | O arquivo pode ser aberto por duplo clique ou colado em qualquer site |
| Fontes | Fonte de sistema (Inter se existir, senão system-ui) | Sem requisição externa |
| Estado | Objeto `state` num IIFE, exposto em `window.__sim` | Depuração fácil, sem framework |
| Processamento pesado | `Float32Array`, `Uint8Array`, `Int16Array`, `Int32Array` | Loops sobre ~600 mil pixels em dezenas de ms |
| Concorrência | Nenhuma (thread principal), com `await tick()` entre etapas | Simplicidade; ver "próximos passos" pra Web Worker |

Não existe servidor, banco, API nem telemetria. Nada sai do navegador do usuário.

## Arquivos da pasta

```
Prototipo Bordado/
  index.html      o protótipo inteiro (HTML + CSS + JS, ~700 linhas)
  README.md       resumo pra humanos (uso, pipeline, limitações)
  docs/           esta documentação
```

Só o `index.html` é código. Ele é autocontido de propósito: copiar o arquivo é copiar o
produto inteiro.

## Como abrir e rodar

- Duplo clique no `index.html`, ou arrastar pro navegador. Funciona em `file://`.
- Abrir com `index.html#exemplo` já carrega um logo fictício de 3 cores (círculo vermelho com
  "S" branco + texto "SIXTINI" azul-marinho + "UNIFORMES" vermelho + barra). Serve pra teste e
  captura automatizada.
- Também aceita colar imagem com Ctrl+V em qualquer lugar da página e soltar arquivo em qualquer
  lugar da janela.

Não há passo de instalação, build ou teste automatizado. Verificação é visual (ver
[05-como-trabalhar.md](05-como-trabalhar.md)).

## Navegadores

Usa só APIs amplamente suportadas: Canvas 2D, `getImageData`/`putImageData`, `createPattern`,
`globalCompositeOperation` (`overlay`), `FileReader`, `Image.decode` implícito via `onload`,
`toBlob`. **Evitou de propósito** `ctx.filter` (blur) porque o suporte no Safari é recente; os
blurs são feitos à mão em `boxBlurF`. `roundRect` só é usado num script de teste, não no
produto.

## O que está pronto

- Upload por clique, arrastar, colar. Remoção automática de fundo em JPG.
- Redução a até 8 cores com limpeza de antisserrilhado. Troca manual da cor de cada fio.
- Cetim, tatami, contorno, camadas (cor de baixo preenche, detalhe borda por cima).
- Sprites de fio com iluminação, brilho especular e torção; relevo por mapa de altura; sombra
  projetada; franzido do tecido; cinco tecidos procedurais; 8 cores de tecido prontas + seletor.
- Lupa 3x, comparação com o original, download PNG, atualização automática ao mexer nos
  controles (com debounce e fila de "pendente" se já estiver processando).
- Emendas entre letras: fio não aparado entre elementos próximos da mesma cor (comum no bordado
  real), com slider de intensidade — de tudo aparado (0%) a emendas em qualquer distância até
  4 cm (100%).
- Altura do bordado independente da largura (opcional, trava por padrão): estica o resultado pra
  caber numa altura fixa diferente da proporção natural do logo.
- Orçamento estimado ao vivo: preço por ponto configurável × contagem de pontos já calculada.

## O que não está pronto (limitações conhecidas)

- Não gera arquivo de máquina de bordar (DST, PES etc.). É só visualização.
- O contador de "pontos" é de segmentos desenhados, não de pontos de máquina.
- Processa na thread principal: 1 a 4 s por atualização completa; a interface trava durante
  os loops mais pesados apesar dos `await tick()` entre etapas.
- Junções complexas de cetim (serifa de um "S", por exemplo) ainda cruzam alguns pontos.
- Logos com degradê/foto viram cores chapadas (é o esperado de bordado, mas o k-means pode
  escolher tons estranhos; ajustar "Cores de fio" ajuda).
- Sem mockup em peça (polo, jaleco); o resultado é o bordado num retângulo de tecido.
- Sem versão mobile específica (o layout empilha abaixo de 900 px, mas não foi otimizado).
- Orçamento é só pontos × preço por ponto — não soma taxa de digitalização, mínimo de pedido nem
  troca de cor/parada de máquina, que num orçamento real de bordado costumam entrar.
- Altura independente é um esticamento do resultado final (imagem), não uma redigitalização —
  cetim/tatami não são regerados pra caber no novo formato, só a composição inteira é esticada.
