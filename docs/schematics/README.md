# Esquema elétrico / board scan

`ps2-90006-90010-board-scan.pdf`: scan da placa **GH-072-42** (série SCPH-90006/90010), com mapeamento de trilhas de alimentação por cor nas duas camadas de cobre (top/bottom) e uma foto com valores de componentes anotados (resistores, capacitores).

Não fiz esse scan, peguei de: https://bitbuilt.net/forums/threads/board-scan-ps2-90006-90010.4340/, postado por **Mister M** (Nexus-M). Créditos a ele pelo trabalho. É um print da página 1 de 4 do tópico original, se precisar de mais discussão/detalhe vale abrir o link direto, tem mais respostas nas páginas seguintes que não peguei aqui.

## O que dá pra tirar de útil dali

- **Legenda de tensões** (cor no scan): 7.5V, 7.5V LDO, 5.0V, 3.5V, 3.5V STBY, 2.5V, 1.8V, 1.8V STBY, 1.25V, GND.
- O 900xx usa **7.5V como trilho principal**, com um regulador que o próprio Mister M chama de "LDO" pro standby do mechacon. É diferente da linha 7900x, que usa 8.5V via MOSFET, mesmo os dois sendo, segundo ele, praticamente idênticos em tudo mais (só muda o filtro anti-estático de controle/memory card).
- **Aviso importante pra quem for mexer na bateria (Subsistema 4):** o próprio autor tentou alimentar os reguladores mapeados a partir de fonte externa nessa família 900xx e não deu vídeo, só esquentou. Ele mesmo recomenda não fazer isso sem entender o risco. Só que, no mesmo tópico, outras pessoas confirmaram numa unidade 90001 que **bateria 2S (Li-ion/LiPo, 7.4V nominal, 8.4V cheia) funciona normal**, bem próximo de onde a fonte interna já entrega ~7.5V. Um outro usuário tentou rebaixar direto pra 5V (bypassando os reguladores 7805) e queimou o MOSFET. Isso sugere que o caminho mais seguro é entrar com 2S no ponto onde a fonte interna hoje entrega os 7.5V, e deixar a cadeia de reguladores já existente na placa fazer o resto, não tentar alimentar cada trilho manualmente do zero.
- **O que NÃO tem nesse material:** localização dos pontos de solda do IDE (pro mod de SSD). Isso ainda precisa ser achado por inspeção física da placa (ou visto direto no PDF comparando com a foto da placa real, mas ainda não confirmei).
