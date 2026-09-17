# Decisões técnicas e trade-offs

---

## Postgres com Prisma, e não apenas o cliente do Supabase

Com 59 modelos e relações densas, escrever SQL solto viraria fonte de erro silencioso. O Prisma
dá schema versionado, migrations e tipagem ponta a ponta — o banco e o TypeScript deixam de
poder discordar.

O Supabase segue como infraestrutura de banco e storage. A escolha não foi entre os dois: foi
usar cada um para o que ele resolve melhor.

**Trade-off:** mais uma camada e um passo de migration a cada mudança de schema. Em compensação,
refatorar com 59 modelos deixa de ser adivinhação.

---

## O cliente final não é usuário da plataforma

Ele não deve ver o painel da agência, não deve ter permissões, não deve aparecer em contagem de
assento. Precisa de uma porta estreita: ver, aprovar, comentar.

Sessão separada torna essa fronteira explícita na arquitetura, em vez de depender de checagem de
papel repetida em cada tela — que é onde esse tipo de regra costuma vazar.

**Trade-off:** dois sistemas de autenticação para manter. O ganho é que um bug no painel não abre
o portal, e vice-versa.

---

## Aprovação por WhatsApp, não por e-mail

É a decisão de produto que originou o projeto. Cliente de agência não abre e-mail e não aprende
ferramenta nova; abre WhatsApp. A notificação com link direto elimina exatamente a etapa em que o
processo morria.

**Trade-off:** dependência de um canal que não foi feito para isso, e a complexidade de manter
uma integração de WhatsApp funcionando. Foi assumida porque é o diferencial do produto — sem ela,
o Regma seria mais um painel que o cliente não abre.

---

## Servidor de WhatsApp próprio, decidido por margem

Sessão persistente de WhatsApp não sobrevive em serverless — a função sobe, responde e morre, e
a conexão morre junto. Então o WhatsApp sempre precisou morar fora da aplicação. A pergunta era
onde.

**Primeira versão:** motor próprio com Baileys, rodando local. Caía sempre que a máquina
desligava.

**A alternativa óbvia era um provedor pago** — e foi descartada por conta: Z-API e similares
cobram **por número** (~R$100/mês cada). Numa plataforma cujo papel é fornecer a conexão para
todos os clientes, custo por unidade estoura a margem conforme a base cresce. Um servidor próprio
segura dezenas de números por custo fixo.

**Decisão:** servidor Evolution auto-hospedado, uma instância por conta, webhook apontando de
volta para a aplicação.

**Trade-off:** infraestrutura própria para monitorar, em troca de custo que não escala com a
base. Foi o custo unitário que determinou a arquitetura — não o contrário.

---

## Escopo largo em uma primeira versão

O Regma cobre CRM, editorial, WhatsApp, e-mail e financeiro. É muito para uma v1, e é escolha
consciente: **a dor que originou o produto é exatamente a fragmentação entre ferramentas.** Um
produto que resolvesse só uma parte devolveria o usuário ao problema original.

**Trade-off assumido:** mais superfície para manter e mais tempo até o lançamento, em troca de
resolver a dor inteira em vez de uma fatia dela.

---

## Aprendizados

**Como eu descobri: usando.** Testei a cada construção, mas os problemas de verdade não
apareceram em teste — apareceram quando comecei a operar um cliente real dentro da ferramenta
ainda em desenvolvimento. Nenhum dos três abaixo teria aparecido de outro jeito.

### Rate limiting em memória não existe em serverless

**O que eu esperava:** guardar as tentativas de login num `Map` em memória seria suficiente para
bloquear força bruta.

**O que aconteceu:** não bloqueava. Na Vercel cada instância tem o seu próprio `Map`, o estado
some no cold start e não é compartilhado entre instâncias — então o contador nunca chegava ao
limite.

**E tinha um segundo problema, pior:** a chave estava no slug, não no IP. Isso permitiria a um
atacante contornar o limite trocando o identificador — e, no sentido inverso, travar o acesso de
um cliente legítimo.

**O que mudei:** contagem persistida, com chave por IP + identificador.

**O que fica:** em serverless, *nenhum* estado sobrevive em memória entre requisições. Parece
óbvio depois; não é antes.

### Migration do Prisma não passa pelo pooler do Supabase

**O que eu esperava:** usar a mesma URL de conexão para runtime e para migration.

**O que aconteceu:** a migration travava. O schema engine do Prisma não funciona através do
pooler (porta 6543, pgbouncer) — precisa de conexão direta.

**O que mudei:** `DIRECT_URL` na porta 5432 para as migrations; o runtime segue pelo pooler, com
o driver adapter `@prisma/adapter-pg`.

**O que fica:** conexão de aplicação e conexão de migração têm requisitos diferentes. O pooler
otimiza uma e quebra a outra.

### Processo com estado precisa de uma máquina que não durma

**O que eu esperava:** rodar o motor de WhatsApp (Baileys) localmente seria suficiente para a
fase de testes.

**O que aconteceu:** toda vez que meu computador desligava, a conexão caía. E não era bug — era
a natureza da coisa: sessão persistente exige um host sempre de pé.

**O que eu avaliei:** contratar um provedor pago resolveria a estabilidade, mas cobra por número
(~R$100/mês cada) — o que quebra a margem de uma plataforma que precisa fornecer a conexão para
todos os seus clientes.

**O que mudei:** servidor Evolution auto-hospedado, custo fixo, uma instância por conta.

**O que fica:** "onde isso vai rodar" é decisão de arquitetura, não detalhe de deploy. E o
modelo de cobrança de um fornecedor pode ser incompatível com o modelo de negócio do seu produto
— vale checar a conta antes de escolher a ferramenta.

### A primeira correção não era a causa

**O sintoma:** a aplicação derrubava conexões com o banco de forma intermitente. Nada óbvio nos
logs, e o erro não acontecia sempre.

**Primeira hipótese:** limite de conexões na URL do banco. Ajustei o `connection_limit` na
`DATABASE_URL` e subi o deploy. Parecia resolvido.

**Não estava.** As quedas voltaram. A causa real era outra camada: o pool do driver adapter
`pg`, que abria mais conexões do que o banco aguentava, independente do que estivesse escrito
na URL. Só fechei no dia seguinte.

**O que fica:** o mais perigoso não é o bug que você não consegue resolver — é o que você acha
que resolveu. A primeira correção baixou o sintoma o suficiente para eu acreditar que tinha
acabado, e foi isso que custou o tempo. Hoje eu desconfio de correção que funciona sem eu
conseguir explicar exatamente *por que* funcionava antes e não funciona agora.

> Registro no repositório: tentativa em 21/07 às 13h23 (`connection_limit` na URL); causa real
> identificada em 22/07 às 20h30 (`pool do adaptador pg — a causa real das quedas`).

---
