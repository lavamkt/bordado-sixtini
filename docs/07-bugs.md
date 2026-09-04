# 07. Registro de bugs

Todo bug encontrado e corrigido neste protótipo, na ordem em que aconteceram. Ver a regra completa
de quando e como registrar em [06-processo.md](06-processo.md).

Formato de cada entrada: sintoma, causa raiz, correção, como reconhecer se voltar.

---

## BUG-001 — densidade não deixava o tecido aparecer entre os fios (corrigido em 2026-09-04)

**Reportado pelo Kewin:** "quando altera a densidade o fundo entre os fios fica a cor do tecido
mesmo" — ou seja, o esperado era o tecido aparecer nos vãos quando a densidade fica espaçada, e
isso nunca acontecia, por mais alto que o slider "Densidade (espaço entre fios)" fosse.

**Causa raiz:** a camada de bordado, em `render`, desenhava uma "base" (fio a 50% de brilho,
pensada pra evitar o tecido vazando nas emendas entre passadas de fio vizinhas — antisserrilhado
entre sprites que se tocam) preenchendo **toda a região com aquela cor** (`q.labels[i]===l`), não
só a vizinhança real de cada ponto gerado. Resultado: mesmo com pontos bem espaçados (densidade
alta), a região inteira continuava opaca por baixo — a densidade mudava quantos pontos existiam,
mas nunca o que aparecia no fundo entre eles, porque o fundo nunca era o tecido, sempre a base.

**Correção:** trocado o preenchimento por região (`ImageData` cobrindo `q.labels`) por um
`stroke()` de `x1,y1` a `x2,y2` de CADA `stitch` gerado, na mesma cor a 50% de brilho, com largura
`1,15 * espessura do fio` (levemente mais larga que o sprite, pra continuar cobrindo a emenda
entre pontos vizinhos sem deixar aparecer nada por baixo bem junto ao fio). Como o traço só existe
onde um ponto de verdade passa, qualquer vão real maior que a largura do traço mostra o tecido.

**Como reconhecer se voltar:** logo com uma área de tatami grande (círculo sólido, letra grossa)
com densidade no máximo (0,6 mm) deveria mostrar textura de tecido entre as linhas de
preenchimento — se voltar a ficar uma mancha lisa e opaca da cor do fio (sem nenhuma variação/
brilho do tecido por baixo), o bug voltou. Testado com o logo padrão da Sixtini (SIXTINI
CAMISARIA) forçando tatami com `Largura máxima do cetim` reduzida pra 3 mm — as letras passaram a
mostrar tecido nas faixas entre as linhas diagonais de preenchimento com densidade alta.

## BUG-002 — caixa de soltar o logo sobrepondo o título "Logo" (corrigido em 2026-09-04)

**Reportado pelo Kewin:** "a parte de soltar o logo tá bugado" — a caixa tracejada "Solte o logo
aqui ou clique pra escolher" aparecia visualmente colada/sobreposta ao título "Logo" acima dela,
em vez de vir separada por um espaço normal como as outras seções.

**Causa raiz:** `#drop` é um `<label>`, que por padrão do navegador é `display:inline` — mesmo
tendo filhos `display:block` (`.drop strong`, `.drop small`), o CSS de `.drop` nunca declarava
`display` nenhum pra si mesma. Um elemento inline com padding/borda visíveis (`.drop` tem
`border:2px dashed` + `padding:18px 14px`) DESENHA esse padding/borda, mas eles não empurram os
elementos vizinhos no fluxo normal do jeito que um bloco empurraria — resultado: a caixa
renderizava ~20px mais alta do que devia, invadindo o espaço do `<h3>Logo</h3>` acima. Confirmado
medindo `getBoundingClientRect()` antes/depois de forçar `display:block` via JS: a caixa saltou de
`y=78` pra `y=98` (a posição correta, logo depois da margem do `h3`).

**Correção:** adicionado `display:block` no início da regra `.drop` em `index.html`. Um-linha,
sem efeito colateral (os filhos já eram `display:block`, então o comportamento visual deles não
muda — só o container passa a reservar o espaço vertical certo).

**Como reconhecer se voltar:** medir `document.querySelector('#drop').getBoundingClientRect().y`
e comparar com `document.querySelector('aside h3').getBoundingClientRect()` (bottom + margem) —
se a caixa começar ANTES do título "Logo" terminar (mais a margem de 10px), o bug voltou. Vale
como lição geral: qualquer `<label>`/`<span>`/`<a>` (inline por padrão) usado como container com
padding/borda visíveis e filhos em bloco precisa de `display:block` explícito desde a criação —
ver nota em [02-arquitetura.md](02-arquitetura.md).

## BUG-003 — soltar o logo direto na caixa processava o arquivo duas vezes (corrigido em 2026-09-04)

**Achado ao investigar o BUG-002** (não reportado separadamente pelo Kewin, mas achado revisando o
mesmo código): existem dois handlers de `drop` — um específico em `#drop`
(`drop.addEventListener('drop', ...)`) e um genérico em `window`
(`window.addEventListener('drop', ...)`, pensado só pra impedir o navegador de navegar pra longe
da página quando o usuário solta um arquivo em QUALQUER lugar fora da caixa). Como o evento de
`drop` disparado na caixa **borbulha** (bubbling) até o `window`, soltar o arquivo bem na caixa
disparava os dois handlers, chamando `handleFile`/`loadImage` (e portanto `run('full')`) duas
vezes seguidas pro mesmo arquivo.

**Correção:** `e.stopPropagation()` no handler específico de `#drop`, antes de chamar
`handleFile`. O handler genérico do `window` continua funcionando normalmente pra quem solta o
arquivo fora da caixa (em qualquer outro lugar da página).

**Como testar:** simular um `drop` sintético (`DragEvent` com `DataTransfer` contendo um `File`)
direto no `#drop` e contar quantas vezes o status passa por "Preparando a imagem..." (início de um
`run('full')`) — tem que ser exatamente 1. Testado também soltando fora da caixa (em `<aside>`):
continua processando 1 vez, via o handler do `window`.
