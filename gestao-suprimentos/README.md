# Gestão de suprimentos

Sistema interno de compras, estoque, produção e clientes — e de rastreabilidade
de cada equipamento que a fábrica monta, até onde ele está hoje.

**Em desenvolvimento** · **Código fechado** — este é o estudo de caso.

```
Node.js · TypeScript · React · Vite · Express · Prisma · PostgreSQL · Electron
```

---

## O problema

O controle era planilha, e planilha tem dois defeitos que só aparecem quando o
time cresce. O primeiro é conhecido: aberta em três máquinas, quem salva por
último apaga o trabalho dos outros dois. O segundo é pior e mais silencioso —
**saldo de estoque digitado à mão**. Ninguém digita um saldo errado de propósito;
o número simplesmente para de bater com o que existe na prateleira, e não há como
saber quando parou nem por quê.

E faltava a resposta para a pergunta que aparecia depois da venda: **onde está
aquele equipamento?** Emprestado a um cliente? Vendido? Voltou? Quem fez a última
manutenção nele? Isso vivia em ficha de papel na pasta do técnico.

## O que já está de pé

Servidor, banco e cliente funcionando ponta a ponta:

- **Compras**, do pedido ao recebimento parcial, com custo médio ponderado
- **Estoque**, onde saldo é **resultado de movimentação** — nunca número gravado
- **Produção**, com ordens que consomem material e creditam produto acabado
- **Clientes e propostas**
- **Cada equipamento montado nasce com número de série, lote e código**
- **Rastreabilidade:** em estoque, emprestado a um cliente com data de retorno,
  ou vendido — e aí o número de série fica amarrado a quem comprou
- **Os técnicos de campo preenchem os formulários de manutenção pelo sistema**,
  vinculados àquele equipamento: quem abre a ficha vê o histórico da peça
- **Importação por CSV** do que o fornecedor manda
- **Perfis de acesso** que mudam não só as telas, mas **o que vem dentro da
  resposta**
- **Roda no navegador pela rede local e também como aplicação desktop**

## Como está montado

| peça | o que é |
|---|---|
| servidor | Node com Express, TypeScript e Prisma sobre PostgreSQL; entrega a API e o cliente na mesma origem |
| cliente | React com Vite |
| desktop | Electron, empacotado como instalável |
| banco | PostgreSQL, e é ele quem carrega os invariantes — não a tela |

## Três decisões de engenharia

### 1. Reserve o estado antes de ler, não depois

O Postgres roda em `READ COMMITTED`, e o padrão que causou **os sete defeitos
mais graves do sistema** era sempre o mesmo: ler o estado fora da transação,
decidir com o que foi lido, e escrever dentro dela.

O sintoma era duplicidade em dois cliques. Dois cliques em "Concluir OP"
creditavam o produto acabado duas vezes. Dois em "Cancelar" estornavam
matéria-prima do nada. Dois recebimentos do mesmo produto faziam o segundo custo
médio sobrescrever o primeiro.

**A alternativa óbvia era** envolver tudo numa transação — que era exatamente o
que já estava feito, e não protegia nada: a checagem e a escrita estavam em
transações **diferentes**. **A forma certa é mudar o estado na mesma instrução
que o confere**, com a condição no `where`, e só depois ler o resto:

```ts
const { count } = await tx.pedidoCompra.updateMany({
  where: { id, status: { in: ["ENVIADO_FORNECEDOR", "RECEBIDO_PARCIAL"] } },
  data:  { status: "RECEBIDO_PARCIAL" },
});
if (count === 0) throw new ErroDeNegocio("...", 409);  // outra requisição chegou antes
```

A segunda requisição espera o lock da linha, relê o status já mudado, e não
encontra linha para atualizar. Quando a trava é sobre **quantidade acumulada** —
receber até o total do pedido, atender até o total solicitado — o ORM não compara
duas colunas da mesma linha, e ali vale SQL cru. Quando é leitura-e-escrita de um
número, como o custo médio ponderado, a linha é travada antes — **e em ordem de
id**, senão dois lotes com os mesmos produtos travam em cruz e o banco aborta um
por deadlock.

### 2. Nunca devolver o modelo inteiro numa resposta

O produto aparece como relação em quase todo módulo — pedido, ordem de produção,
requisição, proposta, movimentação — e carrega o **custo médio**.

**A alternativa óbvia era** `include: { produto: true }`, que é o caminho que o
ORM oferece e o que qualquer tutorial mostra. **Não serviu porque** ele traz o
modelo **inteiro**: todo campo novo do schema passa a vazar sozinho por todas
aquelas rotas, sem ninguém escrever uma linha. Foi assim que o custo apareceu em
três respostas que nenhuma revisão havia olhado.

Hoje existem seletores por perfil, e por serem `select` e não `include`, **campo
novo só aparece se alguém o escrever ali** — o silêncio passou a ser seguro.

### 3. Veredito, não ingrediente

O servidor devolve `abaixoDoMinimo: boolean`. Não devolve `saldo` e
`estoqueMinimo` para a tela subtrair.

**A alternativa óbvia era** mandar os números e calcular no front, que é mais
flexível e evita um campo no contrato. **Não serviu porque** foi assim que
"abaixo do mínimo" acabou com **quatro definições que discordavam entre si**, e o
mapa de status com quatro cópias. Regra de negócio calculável na tela é regra de
negócio que vai ser calculada em todas as telas, cada vez de um jeito, e a
divergência aparece como bug de relatório meses depois. Vale para qualquer regra
nova.

## Como sei que não quebrou

Três suítes, todas contra **PostgreSQL de verdade** e **pela porta HTTP** — não
contra as funções por dentro:

| suíte | o que cobre |
|---|---|
| permissões | **227 conferências** de rota × perfil, e o que sai *dentro* da resposta: nenhum perfil sem direito a custo pode ver custo médio ou preço da última compra, em nenhuma das rotas onde o produto aparece como relação |
| invariantes | saldo nunca negativo, nada entra ou sai sem movimentação, não se recebe além do pedido nem se atende além do solicitado, custo médio ponderado, estorno de ordem cancelada, importação que não zera campo nem cria saldo órfão |
| concorrência | as sete rotas que movimentam estoque e dinheiro, com requisições **simultâneas** |

**Por que contra o banco de verdade:** metade do que essas suítes verificam —
constraints, índice único parcial, travamento de linha, atualização condicional —
é comportamento do banco, e sumiria num dublê em memória. E os dois piores
defeitos já encontrados aqui **passaram por typecheck e build limpos**.

**Por que a de permissão é a que mais importa:** dos achados da auditoria,
**cinco eram vazamento de preço, e nenhum apareceu em revisão de código**. O que
impede o sexto é aquele arquivo.

As três **se recusam a rodar fora de um banco cujo nome termina em `_teste`** — a
de permissão trunca tabelas, e apontada para produção apagaria a empresa.

## Um detalhe que não parece importar e importa

Os dados de demonstração têm **190 produtos do negócio real** — componentes e
semicondutores das placas, eletrodos e quadros das cubas, transdutores da
instrumentação — e os 20 equipamentos acabados com ficha técnica ligando aos
materiais. Fornecedores e clientes são inventados.

E o seed obedece ao mesmo invariante do sistema: **saldo é resultado de
movimentação, não número gravado direto.** Isso não é purismo — a primeira carga
gravava saldo solto e deixou **61 dos 65 produtos irreconciliáveis**. Um seed que
desrespeita o invariante do sistema produz um ambiente de teste que mente, e
todo teste que roda em cima dele mente junto.

## O que eu faria diferente

**Eu teria escrito a suíte de concorrência antes das rotas, não depois da
auditoria.** Corrida não aparece em typecheck, não aparece em revisão de código e
não aparece em teste sequencial — só aparece disparando requisições de verdade e
olhando o saldo depois. Foram sete defeitos do mesmo padrão, e todos poderiam ter
sido pegos pelo primeiro teste desse tipo.

**Eu teria começado pelos seletores de resposta.** A escolha entre `include` e
`select` parece de estilo no primeiro módulo e vira dívida de segurança no
quinto. Se o padrão tivesse sido `select` desde a primeira rota, os cinco
vazamentos de preço não teriam como existir.
