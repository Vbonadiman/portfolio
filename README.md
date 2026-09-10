# Portfólio — Victor Bonadiman

Sistemas que projetei e escrevi do zero: dois em produção diária numa indústria,
e um ERP em desenvolvimento. **São de código fechado** — o que está aqui é o
estudo de caso: o problema que cada um resolve, o que ele faz, as decisões de engenharia, as
alternativas óbvias que descartei e por quê.

> Código aberto meu fica em
> [Animacoes](https://github.com/Vbonadiman/Animacoes) — peças de animação e de
> interface, todas sem dependência.

---

### [Supervisório industrial](supervisorio/)

Painel de planta em tempo real: acompanha todos os registradores de cada
equipamento, guarda o histórico e exibe alertas personalizados de acordo com equipamento.
Comunica por MQTT ou Modbus serial.

Node.js · Express 5 · Socket.IO · SQLite · JWT

*Decisões que valem a leitura: por que lacuna é `null` e nunca zero, por que
equipamento mudo não é silêncio, e por que comparar leitura instantânea com
limite de média erra nos dois sentidos.*

### [Gestão de suprimentos](gestao-suprimentos/) · *em desenvolvimento*

Compras, estoque, produção e clientes para o time inteiro — e rastreabilidade de
cada equipamento que a fábrica monta, do número de série até o cliente que
comprou, com os formulários de manutenção do técnico de campo vinculados à peça.

Node.js · TypeScript · React · Express · Prisma · PostgreSQL · Electron


### [Gestor de acessos e licenças](gestor-acessos/)

O cofre de contas, senhas e licenças de software das quatro empresas do grupo,
com permissão por papel, alcance por empresa, auditoria de tudo e sincronização
com o provedor de identidade. Inclui ativos de TI com etiqueta QR e abertura de
chamado pelo celular.

Node.js · Microsoft Graph · exceljs · qrcode

*Decisões que valem a leitura: por que o formulário público é um processo
separado e não um `if` no roteamento, por que os vínculos são por texto e não
por id, e por que a sincronização nunca apaga uma conta.*

---

📫 bonadiman2001@gmail.com · [LinkedIn](https://www.linkedin.com/in/victor-bonadiman-961106251) · [GitHub](https://github.com/Vbonadiman)
