# Supervisório industrial

Painel de planta que mostra o que os equipamentos da linha estão fazendo agora,
guarda o histórico e calcula exposição ocupacional a amônia contra os limites da
norma.

**Em produção desde 2026** · **Código fechado** — este é o estudo de caso.

```
Node.js · Express 5 · Socket.IO · SQLite · JWT · frontend sem bundler · runtime embarcado
```

---

## O problema

O que existia era o número no display do equipamento. Para saber a tensão de
saída do insensibilizador de meia hora atrás, alguém precisava ter estado na
frente dele naquela meia hora. Não havia histórico, não havia comparação entre
turnos, e não havia como responder à pergunta que a fiscalização faz: **qual foi
a exposição média a amônia na jornada de quem trabalha na câmara.**

Havia ainda uma restrição que descartava a solução óbvia: a máquina que ia rodar
isso na planta **não tem nada instalado e não vai ter**. Nada de instalar
runtime, nada de depender de internet no meio do turno.

## O que ele faz

- **Acompanha todos os registradores** de cada equipamento em tempo real, com a
  tela atualizando por WebSocket em vez de recarregar
- **Um cartão por equipamento**, criado sozinho quando o equipamento comunica
- **Guarda o histórico** em banco na própria máquina, para comparar turno com turno
- **Calcula a exposição a NH₃** por média ponderada no tempo, contra os limites
  de norma, e diz que fração da janela foi de fato medida
- **Alarme com nome:** a mensagem diz qual é a pior condição ativa, uma sirene
  por grandeza no máximo
- **Sobe por um executável**, com Node e PHP embarcados no próprio pacote
- **Hoje na linha:** insensibilizadores, hidrômetros e analisadores de amônia

## Como está montado

| peça | o que é |
|---|---|
| backend | Node com Express 5, Socket.IO para o tempo real, SQLite local e autenticação por JWT |
| frontend | HTML e JavaScript clássico servidos por PHP, **sem bundler** |
| banco | um arquivo SQLite na própria máquina — o dado não sai da planta |
| runtime | Node e PHP **embarcados no pacote**; a planta não instala nada |

O caminho por onde o dado chega é escolhido em configuração: **MQTT pela rede**
ou **Modbus serial** nos equipamentos Novus. Abaixo dessa escolha, o resto do
sistema recebe o mesmo formato e não sabe qual foi.

> **Autoria:** escrevi o sistema e as duas fontes acima. Existe uma terceira
> fonte no código, de um colaborador, que ele publica no repositório dele.

## Cinco decisões de engenharia

### 1. Lacuna é `null`, nunca zero

Quando a sonda não responde, o valor gravado é `null` — no banco, no JSON, no
gráfico e na tela.

**A alternativa óbvia era** gravar `0`, que simplifica tudo: o gráfico não tem
buraco, a média não tem caso especial, o tipo da coluna é sempre número.
**Não serviu porque** zero é uma medição válida. Um `0` de sonda ausente entra na
média e a puxa para baixo, e no gráfico ele desenha uma queda que nunca
aconteceu — o operador vê a amônia caindo a zero exatamente quando o sistema
perdeu a capacidade de medi-la. Hoje o gráfico **quebra a linha** em vez de
interpolar, a tela mostra `—`, e `null` não entra em média nenhuma.

### 2. Equipamento mudo não é silêncio

Desligar o equipamento com o gateway ligado gera leitura na cadência normal, com
todas as grandezas em `null`. Depois de três ciclos assim, isso vira um estado
próprio — **"Sem resposta"** — com alerta.

**A alternativa óbvia era** tratar ausência de dado como "sem novidade" e deixar
o último valor na tela. **Não serviu porque** é indistinguível de um equipamento
que está bem e estável, e é o pior modo de falha possível num supervisório: a
tela continua verde com um número velho, e ninguém vai olhar. Sonda ausente e
sonda quebrada também não podem parecer a mesma coisa: **grupo vazio diz por que
está vazio.**

### 3. A média de exposição é calculada no backend, não na tela

O analisador publica **cerca de 1 450 leituras por hora** — uma a cada 2,5
segundos. Uma jornada de 8 horas são **~11,6 mil linhas, ~3,4 MB**.

**A alternativa óbvia era** mandar a série para o navegador e deixá-lo somar,
como se faz em qualquer dashboard. **Não serviu porque** a conta não é uma soma:
é média **ponderada pelo tempo**, com **corte de silêncio de 5 minutos por
amostra** — período sem leitura não vira exposição presumida —, e a resposta
precisa dizer **que fração da janela foi realmente medida**. Média de 8 horas
calculada sobre 40 minutos de dado não é média de 8 horas, e uma tela que não
diz isso está mentindo com número redondo.

### 4. Comparar leitura instantânea com limite de média é errado nos dois sentidos

Os limites que o sistema aplica não são um número só:

| limite | valor | aplica-se a | origem |
|---|---|---|---|
| Limite de tolerância | 20 ppm | **média** da jornada | NR-15 Anexo 11 |
| Valor máximo | 30 ppm | **cada amostra instantânea** | NR-15 Anexo 11, com fator de desvio 1,5 |
| TLV-STEL | 35 ppm | média de 15 min | ACGIH / NIOSH |

**A alternativa óbvia era** um limite único e um alarme só. **Não serviu porque**
o erro acontece nas duas direções: comparar um pico isolado contra o limite de
média pinta alarme falso, e comparar a média contra o limite instantâneo esconde
violação de jornada numa caixa que nunca passa de 19 ppm.

Duas consequências que só aparecem lendo a norma na fonte, não de memória: a
amônia **não** está assinalada na coluna de "Valor Teto", então vale o fator de
desvio e não a regra de teto; e ultrapassar os 30 ppm instantâneos é, no texto,
**situação de risco grave e iminente** — o que muda o que a tela deve fazer, não
só a cor do número.

Acima do fundo de escala do sensor o número **não é medição**: a tela escreve
`> 100 PPM`. E temperatura e umidade internas da caixa não são "saúde do
equipamento" — elas definem se a leitura de amônia **vale**, contra o envelope
de operação do sensor.

### 5. Registrador de bits não é medição

Alguns registradores são campos de bits, não grandezas. A tela mostra o valor em
hexadecimal e quais bits estão altos — **sem rótulo por bit.**

**A alternativa óbvia era** traduzir cada bit para o nome que o manual do
fabricante dá. **Não serviu porque** aquele manual não foi validado contra o
equipamento. Um rótulo errado num alarme é pior que hexadecimal: o hex faz o
técnico ir conferir, o rótulo errado o faz agir com confiança na informação
errada.

## Como sei que não quebrou

Três suítes, e uma delas com uma decisão que vale contar:

```
teste_exposicao_nh3.js   as médias, sem banco e sem rede
teste_faixas_nh3.js      as faixas e vereditos do cartão do analisador
teste_fonte_gateway.js   contra o broker real da bancada
```

O teste das faixas **recorta o bloco de NH₃ do arquivo real do front e o avalia
num `vm`**, em vez de copiar as funções para dentro do teste. O front não tem
bundler, e uma segunda cópia dos limites de gás seria exatamente o risco a
evitar — um teste que aprova limites que não são os que rodam na planta. Se o
bloco for renomeado, é esse teste que quebra primeiro, e é o que se quer.

## O que eu faria diferente

**Eu teria lido a norma antes de escrever o alarme, não depois.** A primeira
versão comparava tudo contra um limite único, e reescrever isso significou
mexer no cálculo, no banco e na tela ao mesmo tempo. Requisito que vem de norma
publicada é o mais estável que existe num projeto — é o primeiro que deveria ter
sido lido, e eu o tratei como detalhe de acabamento.

**Eu não confiaria em vocabulário de equipamento que nunca comunicou.** Um dos
modelos previstos no sistema nunca enviou uma única leitura na bancada: todo o
agrupamento dele foi escrito a partir da documentação, e está marcado como não
validado. Escrever suporte para hardware que não se tem na mão gera código que
parece pronto e não é — hoje eu deixaria fora até existir um equipamento para
apontar.

---

*⚠️ Os cortes de exposição citados aqui são **ocupacionais** (pessoa na câmara).
Para casa de máquinas quem manda é outro conjunto de normas, e os números mudam.*
