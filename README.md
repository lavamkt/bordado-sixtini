# Protótipo: Simulador de Bordado

Página única (`index.html`), sem dependência nenhuma: abre direto no navegador (Chrome, Edge,
Firefox, Safari), inclusive por duplo clique no arquivo. Nada é enviado pra servidor; todo o
processamento roda em Canvas 2D no próprio navegador.

**Prévia pública (pra mandar pro cliente):** https://lavamkt.github.io/bordado-sixtini/
Publicado no GitHub (`github.com/lavamkt/bordado-sixtini`, conta da Lava) via GitHub Pages.
Repositório isolado só desta pasta — não é o mesmo Drive/CLAUDE.md do resto do `_Dev`. Regra de
manutenção (quem for mexer, ver [`docs/06-processo.md`](docs/06-processo.md), Regra 4): toda
mudança commitada e publicada aqui é também enviada (`git push`) pra esse repositório, pra o link
sempre mostrar a versão mais recente.

O que faz: o usuário sobe o PNG (ou JPG) de um logo e a página mostra como ele ficaria bordado
num tecido, com pontos de cetim, preenchimento tatami, relevo, brilho de fio e textura de tecido.

Documentação completa pra quem for mexer no código (arquitetura, pipeline, parâmetros, como
testar e depurar): pasta [`docs/`](docs/README.md).

## Como usar

1. Abrir `index.html`.
2. Soltar o logo na área pontilhada, clicar pra escolher, ou colar (Ctrl+V) uma imagem.
   O botão "Usar logo de exemplo" carrega um logo fictício de 3 cores pra testar.
   Abrir a página com `index.html#exemplo` já carrega o exemplo sozinho.
3. Ajustar largura do bordado, cores, ângulo, densidade, tecido. A prévia atualiza sozinha.
4. "Baixar PNG" salva a imagem. "Segurar pra ver o original" compara com o logo original.
   A lupa (ligada por padrão) amplia 3x onde o mouse passa.
5. "Emendas entre letras" simula o fio que a máquina não corta entre elementos próximos da mesma
   cor (ex.: entre o "S" e o "I" de SIXTINI) — 0% deixa tudo aparado.
6. Desligando "Manter proporção do logo", a altura vira independente da largura e o bordado
   estica pra caber num tamanho exato (útil pra simular um espaço fixo, tipo bolso ou aba).
7. "Orçamento" mostra o valor estimado ao vivo: pontos gerados × preço por ponto (editável).

## Como funciona (pipeline)

1. **Preparo**: a imagem é reduzida pra 1000 px no lado maior. Se não tem transparência
   (JPG), o fundo é removido por preenchimento a partir das bordas (cor dos cantos, tolerância
   fixa), com erosão de 1 px pra tirar o halo. Recorta na caixa do conteúdo e adiciona margem.
   A escala física vem da "largura do bordado" em cm (px/mm), e todos os parâmetros em mm
   (fio, densidade, ponto) são convertidos por ela.
2. **Cores dos fios**: k-means (até 8) sobre os pixels opacos, com fusão de centros parecidos,
   descarte de clusters de antisserrilhado (quase só borda, cor intermediária entre duas outras)
   e filtro de moda 3x3 pra tirar pontinhos. Cada cor vira um fio; o usuário pode trocar a cor.
3. **Regiões por cor**: a cor de maior área é bordada primeiro. Como no bordado real, a região
   de uma cor inclui o que ela envolve de cores posteriores (ex.: círculo vermelho com "S" branco
   dentro: o vermelho preenche o disco todo, o "S" é bordado por cima) mais 0,5 mm de
   sobreposição nas vizinhas.
4. **Cetim ou tatami**: mapa de distância até a borda (chamfer) + filtro de máximo dizem a
   largura local. Área mais estreita que "largura máxima do cetim" (7 mm por padrão) vira cetim;
   o resto vira preenchimento tatami no ângulo escolhido, com contorno em cetim na borda
   (opcional). A direção do cetim vem do gradiente da distância (ângulo duplo, suavizado em 3
   escalas escolhidas pela largura local, pra não fazer leque nas pontas). Os pontos de cetim
   são gerados por amostragem com mapa de cobertura, então o espaçamento segue a densidade.
5. **Render**: cada ponto é desenhado com um sprite de fio (cilindro iluminado, com brilho
   especular e torção), em 32 ângulos e 3 variações por cor, com cache. Por baixo vai uma base
   escura da cor (pra não vazar tecido entre fios). Depois: furos de agulha no tatami, relevo por
   mapa de altura (normal x luz), tecido procedural (piquê, sarja, malha, tricoline, jeans) com
   grão, sombra projetada e franzido em volta do bordado.

## Limitações conhecidas (é protótipo)

- Não gera arquivo de máquina (DST/PES); é só visualização.
- Logos com degradê ou foto viram poucas cores chapadas (é o comportamento esperado de bordado,
  mas o k-means pode escolher cores estranhas; ajustar o número de cores ajuda).
- Letras muito finas pra escala escolhida (menos de ~1 mm) ficam grosseiras, como ficariam no
  bordado real.
- Processamento síncrono na thread principal: 1 a 4 s por atualização dependendo do logo e do
  número de cores. Dá pra mover pra Web Worker se virar produto.
- Anti-aliasing entre cores: em alguns joins (ex.: serifa de um "S") os pontos de cetim cruzam.

## Próximos passos possíveis

- Colocar dentro da LP de uniformes (Lava Pages) como ferramenta de conversão: "veja seu logo
  bordado" + botão de orçamento com o PNG anexado.
- Mockup em peça (polo, jaleco) com o bordado posicionado no peito.
- O orçamento (pontos × preço por ponto) já existe, mas a contagem é dos segmentos desenhados,
  não dos pontos reais de uma máquina de bordar — uma estimativa de pontos de máquina de verdade
  deixaria o valor mais preciso.
