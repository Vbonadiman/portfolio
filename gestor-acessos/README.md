# Gestor de acessos e licenças

Controle de contas de e-mail, senhas e licenças de software das quatro empresas
do grupo, com três níveis de permissão, alcance por empresa, registro de
auditoria de tudo e sincronização com o provedor de identidade.

**Estado:** em produção desde 2026 · **Código fechado** — este é o estudo de caso.

```
Node.js · Microsoft Graph · Google Admin SDK · exceljs · qrcode · zero framework de front
```

---

## O problema

O controle de quem tem acesso a quê era uma planilha. Ela respondia "qual a
senha do financeiro@", mas não respondia as perguntas que apareciam sempre:
*quais licenças estão neste e-mail*, *esta conta tem autenticador*, *quem mexeu
nisso na semana passada* e *este notebook é de quem*. Quatro empresas do grupo,
quatro planilhas, e nenhuma delas sabia da outra — o mesmo e-mail aparecia em
duas sem ninguém notar.

## O que ele faz

- Cada **e-mail** é a unidade: senha, 2FA, e-mail de recuperação e situação.
- As **licenças** (M365, Google Workspace, AutoCAD, SolidWorks) são linhas
  presas ao e-mail — um e-mail pode ter várias, com plano, custo e data de
  renovação. É isso que responde "quais licenças estão nesta conta, com que
  senha, e quanto custam por mês".
- **Ativos de TI e pessoas** na mesma base, porque "de quem é este notebook?"
  não se responde sem as duas.
- **Três níveis de permissão**, e **alcance por empresa**: quem não tem uma
  empresa liberada não a vê em lugar nenhum — nem na lista, nem no relatório,
  nem no consolidado.
- **Sincroniza** contas, situação, 2FA e licenças direto do provedor de
  identidade, com **prévia do que vai mudar antes de gravar**.
- **Etiqueta com QR** em cada equipamento: quem está na frente do equipamento
  lê com o celular e abre um chamado, sem ter conta no sistema.
- **Desativar uma pessoa remaneja os ativos dela na mesma operação**, exigindo
  destino de cada item.
- **Alertas calculados:** conta sem 2FA, senha antiga, licença vencida,
  renovação próxima, licença órfã, ativo sem responsável, pessoa desligada com
  equipamento na mão.
- **Registro de auditoria** de tudo, e planilha `.xlsx` sempre espelhada — o
  sistema não sequestra o dado de quem preferir o Excel.

<!-- AS TELAS — entram aqui quando as capturas com dado de demonstração
     estiverem prontas (etapa 5 do roteiro). Chamada de imagem inexistente
     renderiza como ícone quebrado no GitHub, o que é pior que não ter seção:
     por isso ficam fora até existirem de verdade.

     Ordem planejada: 01-empresas (a home, um cartão por empresa),
     02-contas (o cofre, com contagem de licenças na linha),
     03-ativos (cartões com foto e responsável),
     04-relatorios (alertas calculados). -->

## Como está montado

| peça | o que é |
|---|---|
| servidor | Node HTTP em interface local, com token por execução e sessão |
| atendimento por QR | **processo separado**, o único que escuta na rede, com duas rotas e nada mais |
| dados | um documento por empresa, fonte da verdade, com backup rotativo a cada gravação |
| planilha | `.xlsx` regravado inteiro a partir dos dados — espelho, nunca fonte |
| front | HTML/CSS/JS servido localmente, **sem framework e sem bundler** |

## Três decisões de engenharia

### 1. Senha de provedor não vem por API — e a resposta não foi procurar outra API

Nem o Google nem a Microsoft devolvem senha, nem para super admin: elas são hash
e o próprio provedor não as lê de volta.

**A alternativa óbvia era** deixar o campo lá, vazio, e continuar cobrando o
preenchimento. **Não serviu porque** o sistema passaria a pedir eternamente um
dado que não existe. A conta sincronizada é marcada como *gerenciada*, e os
avisos "sem senha" **calam** para ela. Sistema que finge saber é pior que
sistema que admite não saber.

### 2. O formulário público de chamados é um servidor separado, não um `if` no roteamento

Para o celular de quem está no chão de fábrica abrir o formulário do QR, é
preciso escutar na rede e não exigir login. Todo o resto do sistema escuta só na
interface local **porque as respostas trafegam senhas**.

**A alternativa óbvia era** um `if` liberando aquelas duas rotas no servidor
principal. **Não serviu porque** proteger senha com uma condição de roteamento
é o tipo de coisa que quebra numa refatoração distraída, em silêncio. Com dois
servidores, um erro de rota no público não vaza o privado: aquele processo não
conhece rota de senha nenhuma. Vem **desligado por padrão**, e a tela pede
confirmação para ligar.

### 3. Os vínculos entre listas são por texto, não por id

Pessoa → conta, licença → conta, ativo → pessoa: tudo ligado por e-mail e por
nome.

**A alternativa óbvia era** id interno, como manda o manual. **Não serviu
porque** tudo aqui faz ida e volta por Excel, e id interno não sobrevive a
alguém editando a planilha no LibreOffice. O preço é que renomear uma pessoa
desgarra os equipamentos dela — pago com um alerta *Responsável desconhecido* e
uma rotina que **move** o registro quando ele é só um esboço, e **não mexe**
quando já tem senha ou licença. Melhor duas contas visíveis que uma senha
apagada em silêncio.

### 4. Vínculo entre listas só cria o que falta, e só preenche campo vazio

Cadastrar uma pessoa com e-mail e setor cria a conta correspondente, já com nome
e setor preenchidos. Duas regras governam isso: **só cria o que falta** — nunca
apaga, nunca renomeia — e **só preenche campo vazio**.

**A alternativa óbvia era** sincronizar as listas nos dois sentidos, mantendo os
campos iguais. **Não serviu porque** se a conta diz setor "TI" e a pessoa diz
"Financeiro", uma sincronização de mão dupla sobrescreve o trabalho de quem
digitou — e escolhe o vencedor pela ordem em que as coisas foram salvas, que é
uma regra que ninguém entende olhando a tela. O vínculo **não arbitra
divergência, só cobre silêncio.**

O caso difícil é a correção de e-mail depois. A pessoa foi cadastrada com um
endereço, o vínculo criou a conta, e depois alguém corrige o endereço dela: sem
tratamento, nasce uma segunda conta e a primeira fica órfã com cara de
duplicata. O sistema **move** a conta antiga — mas só quando ela é um **esboço**,
que nasceu do vínculo e nunca recebeu nada próprio. Se ela já tem senha, 2FA,
licença ou observação, é registro de verdade: não se mexe nela, a nova nasce ao
lado, e o alerta de "e-mail duplicado" avisa que há duas. **Melhor duas contas
visíveis do que uma senha apagada em silêncio.**

### 5. Uma sincronização que nunca apaga conta, e nunca cobra o impossível

Senha não vem por API — nem do Google, nem da Microsoft, nem para super admin:
elas são hash, e o próprio provedor não as lê de volta. A coluna que marca a
conta como *gerenciada* existe por causa disso, e nela os avisos de "sem senha"
**calam**.

Três recusas deliberadas na sincronização, cada uma corrigindo um jeito de
destruir cadastro:

- **Conta que só existe aqui nunca é apagada.** Pode ser acesso de ERP, de banco
  ou de portal, que jamais esteve no provedor. Ela aparece na prévia como
  informação, não como pendência.
- **Licença só é "sumida" se o produto for do provedor.** Sem essa condição, o
  AutoCAD digitado à mão apareceria como removível em toda sincronização —
  empurrando o usuário a apagar exatamente o cadastro manual que o sistema
  existe para guardar.
- **Remover licença vem desmarcado** na prévia. Acrescentar e atualizar vêm
  marcados; remover exige um clique consciente.

E há uma distinção que só aparece na primeira conexão real: **`null` não é `""`**.
Vindo do provedor, `null` quer dizer "não sei" e preserva o que você digitou;
`""` quer dizer "está vazio lá" e sobrescreve. É o que faz uma data digitada à
mão sobreviver a um provedor que simplesmente não publica aquele campo.

## Como sei que não quebrou

```
teste_vincular.js      o vínculo entre as listas
teste_sincronizar.js   o plano de mudanças, com um provedor falso
```

O de vínculo cobre justamente o que é difícil: idempotência (rodar duas vezes não
duplica), caixa alta e espaço no e-mail, duas pessoas apontando para o mesmo
endereço, e **os dois casos em que a conta NÃO pode ser movida** — quando ela já
tem senha, e quando ela já tem licença.

O de sincronização usa um **provedor falso**, e é o que trava as três recusas
descritas acima: que conta local não seja apagada, que produto de fora do
provedor não seja marcado como sumido, e que o trabalho digitado à mão
sobreviva. A máquina de planejar e aplicar está provada; **as chamadas reais só
foram exercitadas contra um provedor** — o outro caminho foi escrito conforme a
documentação e está marcado como não validado.

Há ainda um conferidor de contraste que lê os tokens direto do CSS e **sai com
erro se algum par reprovar em AA** — os 20 pares passam, o pior está em 5,24.
Ele não guarda cópia das cores de propósito: uma segunda lista divergiria do
tema no primeiro ajuste e passaria a aprovar um tema que não existe mais.

## O que eu faria diferente

**Eu teria criptografado o armazenamento antes de escrever a sincronização.**
A ordem certa era essa, e eu inverti: entreguei a integração com os provedores
primeiro e deixei a criptografia em repouso para depois. A migração para
`scrypt` + AES-256-GCM com senha-mestra pedida na abertura é a próxima entrega —
e o que essa inversão me ensinou é que **uma funcionalidade que aumenta o valor
do que está guardado muda o requisito de como guardar**, e portanto não pode
entrar antes dele.

**Eu escreveria o front com um framework.** Sem bundler foi ótimo para não ter
build e rodar em qualquer máquina, mas o `app.js` cresceu até o ponto em que uma
substituição de bloco apagou oito funções de uma vez — e foi o que me fez pôr o
projeto em git. Módulos de verdade teriam evitado.
