# Guia de revisão — Copa da API Finanças

**Aluno:** Eduardo Caversan da Silva Rocha — **RA:** 2475391  
Material complementar para estudo individual e em equipe. Referências: [roteiro da Copa](https://costasilvati.github.io/IntroNodeJS/copa-api-financas.html) e [Projeto 2](https://costasilvati.github.io/IntroNodeJS/projeto-api-financas.html).

## 1. Mapa dos requisitos

| RF | O que é preciso explicar |
|---|---|
| RF1 | Cadastro com validações de nome, e-mail, CPF com dígitos verificadores e senha; normalização, `UNIQUE`, bcrypt e transação com categorias e conta padrão; sucesso 201. |
| RF2 | Login por e-mail normalizado, `bcrypt.compare`, JWT de uma hora; `/auth/me` protegido e sem hash de senha. |
| RF3 | Isolamento em toda consulta e operação; dono vindo do token; categoria e fonte/destino pertencentes ao mesmo usuário do lançamento. |
| RF4 | CRUD de categorias; `entrada`/`saida`; nome único por usuário e tipo; exclusão com vínculos retorna 409. |
| RF5 | CRUD de fontes/destinos; tipos `renda`, `conta`, `cartao_credito`; cartão exige fechamento e vencimento de 1 a 31 e limite positivo. |
| RF6 | Lançamentos com valor positivo, data válida, categoria compatível e entidades do próprio usuário; filtros, paginação de 20 itens e resumo; parcelas de 1 a 48 no cartão. |
| RF7 | Relatório por categoria com período obrigatório, `SUM`/`GROUP BY`, percentuais e comparação com período anterior. |
| RF8 | Entradas por fonte, saídas por destino e saldo das contas no período. |
| RF9 | Previsão das faturas e limites; `meses` de 1 a 12, padrão 3; fechamento, vencimento e referência de cada parcela. |

Categorias e fontes/destinos têm unicidades diferentes: categoria usa `(usuario_id, nome, tipo)`; fonte/destino usa `(usuario_id, nome)`. “Poupança” pode ser o **nome** de uma fonte/destino do tipo `conta`.

## 2. Status HTTP e respostas

| Status | Quando responder | Exemplo |
|---|---|---|
| **200 OK** | Consulta, atualização ou login bem-sucedido | `GET /auth/me` com token válido e usuário existente |
| **201 Created** | Criação de recurso | Cadastro ou compra parcelada criada |
| **204 No Content** | Exclusão concluída | `return res.status(204).end();` — sem JSON |
| **400 Bad Request** | Dados ou regras de negócio inválidos | Valor negativo, período ausente no RF7, saída com categoria de entrada |
| **401 Unauthorized** | Autenticação inválida | Sem token, token expirado ou senha incorreta no login |
| **404 Not Found** | Registro ausente, alheio ou rota inexistente | Tentar acessar lançamento de outra pessoa |
| **409 Conflict** | Conflito de unicidade ou exclusão com vínculos | E-mail duplicado; categoria com lançamentos |
| **500 Internal Server Error** | Falha inesperada | Falha imprevista no banco; resposta genérica |

**401 × 403 × 404:** 401 indica falha de autenticação. Em HTTP, 403 indica recusa de autorização; porém, no Projeto 2, **recurso de outro usuário recebe 404**, evitando confirmar sua existência. Token válido identifica a pessoa, mas não dá acesso a todos os registros.

Erros usam JSON com `erro` e, quando necessário, detalhes em `campos`:

```javascript
return res.status(400).json({
  erro: 'Dados inválidos.',
  campos: { valor: 'O valor deve ser maior que zero.' }
});
```

`return` encerra a execução daquele caminho da função. O middleware final registra erros inesperados com `console.error` e devolve mensagem genérica; não envia SQL, stack trace ou detalhes internos ao cliente. `UnauthorizedError` deve virar 401 em JSON.

## 3. Isolamento e SQL parametrizado

O identificador do **recurso** pode vir de `req.params.id`. O identificador do **dono** vem exclusivamente de `req.auth.id`, preenchido após a validação do JWT. Desconsidere qualquer `usuario_id` enviado no corpo, URL ou query como fonte de identidade.

Exemplo dentro de um método em `models/`, com `db` na interface de promises:

```javascript
const [rows] = await db.execute(
  'SELECT id, descricao FROM lancamentos WHERE id = ? AND usuario_id = ?',
  [id, usuarioId]
);
```

A rota chama o método com `(req.params.id, req.auth.id)`. Se a consulta de um recurso específico não retornar linha, responde 404. Em uma listagem válida sem resultados, a resposta é 200 com coleção vazia, conforme seu formato.

Os placeholders separam os valores da instrução SQL. Não use `${req.params.id}` ou concatenação para construir a consulta. Uma template string **estática**, usada apenas para escrever SQL em várias linhas com `?`, é compatível com a regra: o problema é interpolar valores.

Autenticação, isolamento e parametrização resolvem problemas distintos. Uma consulta parametrizada pode vazar dados se o filtro lógico estiver errado. Exemplo da carta: `dono AND entrada OR saída` permite qualquer saída; `dono AND (entrada OR saída)` aplica o dono às duas alternativas.

Na criação e atualização, valide também a posse da categoria e da fonte/destino. Uma FK simples verifica que o identificador existe, mas não garante que o registro pertença ao mesmo usuário. Recurso relacionado inexistente ou alheio → 404; categoria própria com tipo incompatível → 400.

## 4. Senhas e JWT

```javascript
const bcrypt = require('bcrypt');
const senhaHash = await bcrypt.hash(senha, 10);       // cadastro
const confere = await bcrypt.compare(senha, usuario.senha_hash); // login
```

O hash protege a senha armazenada. `bcrypt.compare` verifica a senha utilizando o hash salvo; não se faz comparação de senhas no SQL nem se gera um novo hash para compará-lo diretamente, pois o salt faz hashes variarem. Use `await`: uma Promise não é o resultado booleano da comparação. A senha do cadastro deve ter ao menos oito caracteres, com letras e números. `senha_hash` nunca integra respostas nem payloads de token.

Normalização, depois de validar o tipo dos campos recebidos:

```javascript
const emailNormalizado = email.trim().toLowerCase();
const cpfNormalizado = cpf.replace(/\D/g, '');
```

Normalizar CPF não comprova sua validade: é preciso verificar os dígitos verificadores e rejeitar sequências repetidas. No login, a mensagem de erro é genérica para e-mail desconhecido e senha errada.

Trecho da emissão do token, **somente após** credenciais confirmadas:

```javascript
const jwt = require('jsonwebtoken');
const token = jwt.sign(
  { id: usuario.id, nome: usuario.nome, email: usuario.email },
  process.env.JWT_SECRET,
  { expiresIn: '1h' }
);
```

`JWT_SECRET` fica no `.env`, carregado com `dotenv` no início do servidor. Não deve ficar no código nem no Git. O `.env.example` contém nomes de variáveis e valores fictícios. O token é enviado como `Authorization: Bearer <token>`; assinar não torna seu conteúdo secreto, por isso não se inclui senha ou hash nele.

Middleware do padrão oficial, em `middleware/auth.js`:

```javascript
const { expressjwt } = require('express-jwt');
module.exports = expressjwt({
  secret: process.env.JWT_SECRET,
  algorithms: ['HS256']
});
```

`express-jwt` valida assinatura e expiração e disponibiliza o payload em `req.auth`. Todas as rotas de recursos, inclusive GETs e `/auth/me`, são protegidas. Cadastro e login são públicos. Não basta decodificar o JWT sem verificar sua autenticidade.

## 5. Transações, UNIQUE e concorrência

O cadastro cria usuário, categorias padrão e conta padrão como uma única unidade. `beginTransaction` inicia; `commit` confirma tudo; `rollback` desfaz as alterações da transação em caso de falha. Com pool, **todas** as instruções dessa transação devem usar a mesma conexão obtida por `pool.getConnection()`.

Esqueleto conceitual de um método em `models/`: as duas funções auxiliares representam os INSERTs parametrizados do cadastro e recebem a mesma conexão.

```javascript
const conn = await pool.getConnection();
try {
  await conn.beginTransaction();
  const usuarioId = await inserirUsuario(conn, dados);
  await inserirCategoriasEContaPadrao(conn, usuarioId);
  await conn.commit();
  return usuarioId;
} catch (erro) {
  await conn.rollback();
  throw erro;
} finally {
  conn.release();
}
```

Usar `pool.execute()` no meio desse bloco pode executar fora da conexão transacional. Liberar a conexão em `finally` permite devolvê-la ao pool tanto no sucesso quanto na falha. As tabelas precisam de suporte a transações, como InnoDB.

**Condição de corrida:** A consulta um e-mail e não o encontra; B também não o encontra; A e B tentam inserir. O `SELECT` não reserva a unicidade. `UNIQUE(email)` e `UNIQUE(cpf)` impedem que ambas as gravações duplicadas sejam confirmadas; a API converte `ER_DUP_ENTRY` em 409 e informa o campo correspondente, sem expor a mensagem SQL. A normalização precisa ocorrer antes da gravação para representar a mesma identidade de modo consistente.

Transação e `UNIQUE` têm papéis diferentes: a primeira mantém uma operação composta íntegra; a segunda impõe a unicidade. A consulta prévia pode melhorar a mensagem, mas a garantia permanece no banco.

## 6. Parcelas em centavos — RF6

Para um total validado em centavos e uma quantidade válida:

```javascript
const totalCentavos = 25000;
const quantidade = 7;
const base = Math.floor(totalCentavos / quantidade);
const resto = totalCentavos % quantidade;
const parcelas = Array.from({ length: quantidade }, (_, i) =>
  base + (i === 0 ? resto : 0)
);
const soma = parcelas.reduce((total, valor) => total + valor, 0);
// parcelas: [3574, 3571, 3571, 3571, 3571, 3571, 3571]
// soma: 25000 centavos
```

Na entrada, a referência do projeto converte reais com `Math.round(valor * 100)`; a divisão e a conferência são feitas em inteiros. O resto vai **inteiro para a primeira parcela**. Divida por 100 apenas quando precisar representar reais. Arredondar cada parcela isoladamente pode alterar o total: três parcelas de R$ 33,33 somam R$ 99,99.

Todas as parcelas mantêm a **data original da compra**, o mesmo `grupo_parcelamento` gerado por `crypto.randomUUID()` e os números `parcela_atual` de 1 a N. `DELETE /lancamentos/:id?todasParcelas=true` exclui todas da mesma compra, sempre respeitando o dono. Para valores muito pequenos, também é preciso respeitar a exigência de valor positivo em cada lançamento; não se deve gravar parcela de zero centavos.

## 7. Fechamento, vencimento e referência — RF9

1. Encontre o fechamento efetivo do mês da compra: dia configurado, limitado ao último dia daquele mês.
2. Compra **antes** desse fechamento: primeira parcela no mesmo mês. Compra **no dia ou depois**: primeira parcela no mês seguinte.
3. Parcela k: acrescente **k − 1 meses** ao mês da primeira fatura.
4. Compare os **dias configurados** no cartão: se vencimento > fechamento, vence no mesmo mês do fechamento; caso contrário, no seguinte.
5. Monte cada data limitando o dia ao último dia de seu mês. Calcule novamente para cada mês, preservando a configuração do cartão.
6. A referência `AAAA-MM` vem do **vencimento**, nunca do mês da compra ou do fechamento por suposição.

| Compra e configuração | Primeira fatura |
|---|---|
| 18/08/2026; fecha 18, vence 3 | Fecha 18/09; vence 03/10; referência 2026-10 |
| 31/01/2026; fecha 31, vence 8 | Fecha 28/02; vence 08/03; referência 2026-03 |

No segundo caso, a próxima parcela fecha em **31/03**, não em 28/03. Use o dia original do cartão para cada mês. Não some 30 dias para simular um mês e não deixe o construtor `Date` transbordar uma data impossível para o mês seguinte. A referência oficial usa UTC para os cálculos e recomenda `dateStrings: true` no mysql2 para receber `DATE` como `AAAA-MM-DD`.

**Limite utilizado:** soma de todas as parcelas cuja fatura ainda não venceu, inclusive parcelas futuras fora da janela de faturas exibida. **Limite disponível:** limite total menos utilizado. O projeto não pede deduzir pagamentos de uma entidade de pagamento inexistente na especificação mínima.

## 8. Relatórios, DECIMAL e SUM

Agregue no SQL, com `SUM` e `GROUP BY`, filtrando o dono e o período. O mysql2 normalmente devolve `DECIMAL` e `SUM` como texto para preservar a representação decimal. Converta antes de calcular ou devolver números no JSON:

```javascript
const total = Number(row.total ?? 0);
const percentual = totalGeral === 0 ? 0 : total / totalGeral * 100;
```

O exemplo pressupõe `totalGeral` já numérico. Sem conversão, `"10.00" + "5.00"` concatena textos. `toFixed(2)` também retorna string; não o use como substituto de `Number()` para um campo numérico no JSON. A conversão não dispensa os centavos inteiros para repartir valores monetários.

No RF7: saídas por categoria, em ordem decrescente; entradas por categoria; percentuais; comparação com o período anterior. No RF8: entradas por fonte, saídas por destino e saldo das contas = entradas − saídas. Se a base de uma variação percentual for zero, a fórmula usual não define um percentual: a equipe deve documentar o tratamento, pois o enunciado não fixa um valor único para esse caso.

## 9. Organização e checklist rápido

| Local | Responsabilidade |
|---|---|
| `routes/` | `express.Router()`, validações de entrada, chamadas aos models e respostas HTTP |
| `models/` | Classes de acesso a dados, SQL parametrizado, agregações e transações |
| `middleware/` | Proteção JWT; o nome **no singular** é o usado na estrutura oficial |
| `utils/` | Funções puras: CPF, parcelas e datas de fatura |
| `database/` | Conexão/configuração, `schema.sql` e `seed.sql` |
| `server.js` | Configuração do Express, montagem dos routers, 404 e tratamento final de erros |

`middlewares/` é uma denominação comum para a mesma função arquitetural, mas os exemplos deste material adotam `middleware/` para corresponder ao enunciado. CommonJS significa `require` e `module.exports`. SQL executado pela aplicação fica em `models/`; os scripts de criação e carga pertencem a `database/`.

Itens do checklist final a memorizar: e-mail/CPF normalizados e únicos mesmo sem SELECT prévio; nenhum `senha_hash` nas respostas; 401 em JSON; isolamento ao trocar id; compatibilidade de tipo; centavos preservados; compra no fechamento passa para a fatura seguinte; relatórios com números; `.env` fora do Git e `.env.example` presente; README suficiente para executar o projeto.

## 10. Perguntas para responder sozinho

| Pergunta provável | Resposta curta e justificativa |
|---|---|
| A senha errada recebe 400 ou 401? | 401: as credenciais não autenticam o usuário. Mensagem genérica. |
| Token válido permite ler lançamento alheio? | Não. A consulta filtra `usuario_id` e retorna 404 se não houver registro próprio. |
| Por que não devolver 403 nesse caso? | O contrato do projeto usa 404 para não confirmar a existência do recurso alheio. |
| O usuário pode indicar seu próprio id no corpo? | A API deve obter a identidade de `req.auth.id`, independentemente do valor enviado. |
| Placeholder resolve todo problema de segurança? | Não. Evita interpolação de valores, mas não corrige filtro lógico, autenticação ou posse. |
| Uma saída com categoria própria de entrada recebe qual código? | 400, por incompatibilidade de tipo. Se a categoria for alheia, 404. |
| Por que o login usa `bcrypt.compare`? | Para verificar a senha contra o hash armazenado, considerando o salt; não compara hashes novos diretamente. |
| JWT válido para sempre é aceito no projeto? | Não. O RF2 exige `expiresIn: '1h'`; o middleware verifica a expiração. |
| O cadastro falhou ao criar categorias: pode manter o usuário? | Não. A mesma transação deve sofrer rollback. |
| Duas conexões do pool compartilham automaticamente a transação? | Não. Use a conexão obtida por `getConnection` para todas as instruções. |
| SELECT antes do INSERT dispensa UNIQUE? | Não. Requisições concorrentes podem observar a mesma ausência; o banco garante a unicidade. |
| Pode excluir categoria ou fonte com lançamentos? | Não. Responde 409; as FKs do schema preservam os vínculos. |
| DELETE bem-sucedido pode enviar `{ mensagem: 'ok' }`? | Não no padrão do projeto: 204 é sem corpo. |
| Quanto dá R$ 100,00 em três parcelas? | R$ 33,34 + R$ 33,33 + R$ 33,33, com resto na primeira. |
| Compra no dia do fechamento entra na fatura atual? | Entra na que fecha no mês seguinte; o vencimento pode avançar mais um mês. |
| Ajustar fevereiro para dia 28 fixa os meses seguintes no dia 28? | Não. Cada mês reaplica o dia originalmente configurado. |
| O que determina a referência da fatura? | O mês do vencimento, no formato AAAA-MM. |
| Por que não avançar a data de cada lançamento parcelado? | RF6 mantém a data da compra; RF9 já desloca pela parcela. Avançar ambos duplica o deslocamento. |
| Por que usar Number no SUM? | O driver pode entregar texto; o projeto pede cálculos e JSON numéricos. |
| SQL em routes funciona; por que está errado? | Viola a separação exigida: SQL nas classes de models, HTTP nas rotas. |

## 11. Ensaio para a competição

Cada integrante deve resolver os seis treinos sem consultar os demais e explicar **resultado, regra e motivo**. Na rodada de bugs, identificar a linha, citar o RF ou regra de organização e mostrar a correção. Na rodada de faturas, escrever primeiro os centavos e depois fechamento, vencimento e referência.

O roteiro prevê respostas individuais por sorteio/rodízio; ajuda durante a resposta anula o ponto. A entrega é uma carta por equipe até a véspera da aula; o bônus de dois pontos depende de ela ser usada no jogo. Confirmar o envio e levar pelo menos um celular por equipe com acesso ao Kahoot.
