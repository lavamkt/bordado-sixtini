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
