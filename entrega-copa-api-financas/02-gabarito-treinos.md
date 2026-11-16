# Resolução dos seis treinos — Copa da API Finanças

**Aluno:** Eduardo Caversan da Silva Rocha — **RA:** 2475391  
Material complementar de estudo. Base: [seção 13.3 do roteiro oficial](https://costasilvati.github.io/IntroNodeJS/copa-api-financas.html#copa-estudo) e [Projeto 2](https://costasilvati.github.io/IntroNodeJS/projeto-api-financas.html).

## Treino 1 — Status HTTP

| Item | Situação | Status | Justificativa no Projeto 2 |
|---|---|---|---|
| a | `POST /categorias` com nome já usado pelo usuário para o mesmo tipo | **409 Conflict** | Duplicidade de categoria no escopo `(usuario_id, nome, tipo)`; RF4. |
| b | `PUT /fontes-destinos/12` com `tipo: "poupanca"` | **400 Bad Request** | RF5 aceita apenas `renda`, `conta` e `cartao_credito`. “Poupança” pode ser o nome de uma conta, mas não esse valor do campo `tipo`. |
| c | `GET /relatorios/categorias` sem `inicio` e `fim` | **400 Bad Request** | Os dois parâmetros são obrigatórios e devem definir um intervalo válido; RF7. |
| d | `POST /auth/login` com senha errada | **401 Unauthorized** | Credenciais inválidas; RF2. Usar mensagem genérica, sem revelar se o e-mail existe. |
| e | `GET /rota-que-nao-existe` | **404 Not Found** | Rota inexistente; resposta em JSON conforme o padrão do projeto. |
| f | `POST /lancamentos` válido | **201 Created** | Houve criação de lançamento, inclusive quando a compra gera várias parcelas; RF6. |

Exemplo para d: `{"erro":"E-mail ou senha inválidos."}`. Neste exercício, aplicam-se os códigos definidos pelo projeto, sem substituí-los por alternativas HTTP de outros contratos.

## Treino 2 — Caça ao bug

No trecho oficial, contando o comentário `// routes/fontesDestinos.js` como linha 1, os três problemas são:

| Local | Problema | Regra e consequência |
|---|---|---|
| Linha 4 | Interpolação de `req.params.id` e `req.auth.id` no SQL | As consultas devem usar placeholders `?`. Dados da requisição interpolados podem alterar a instrução e permitir injeção de SQL. |
| Linhas 3–5, dentro da rota | Acesso SQL implementado em `routes/` | A estrutura do Projeto 2 exige SQL somente nas classes de `models/`; a rota valida, chama o model e monta a resposta. |
| Linha 7 | Resposta 404 sem `return` | A execução continua até a tentativa de resposta 204 na linha 9. `res.json()` não encerra a função JavaScript; é necessário interromper esse caminho. |

O filtro `usuario_id` já aparece no original; sua ausência não é um quarto erro a apontar. A parametrização precisa preservá-lo. O comportamento concreto de uma segunda tentativa de resposta depende da operação e da versão do Express/Node, podendo causar erro de resposta já enviada; o defeito é continuar depois do 404.

### Implementação corrigida

Contexto: `database/connection.js` exporta a interface de promises de `mysql2`; `middleware/auth.js` exporta `protegerRota`. O banco utiliza as chaves estrangeiras do schema oficial: lançamentos referenciam fontes/destinos **sem exclusão em cascata**. A FK impede a exclusão com vínculos, inclusive se um vínculo surgir concorrentemente.

**`models/FonteDestino.js`**

```javascript
const db = require('../database/connection');

class FonteDestino {
  static async excluir(id, usuarioId) {
    const [resultado] = await db.execute(
      'DELETE FROM fontes_destinos WHERE id = ? AND usuario_id = ?',
      [id, usuarioId]
    );
    return resultado.affectedRows > 0;
  }
}

module.exports = FonteDestino;
```

**`routes/fontesDestinos.js`**

```javascript
const express = require('express');
const FonteDestino = require('../models/FonteDestino');
const protegerRota = require('../middleware/auth');
const router = express.Router();

router.delete('/:id', protegerRota, async (req, res, next) => {
  const id = Number(req.params.id);
  if (!/^\d+$/.test(req.params.id) || !Number.isSafeInteger(id) || id < 1) {
    return res.status(400).json({
      erro: 'Dados inválidos.',
      campos: { id: 'Informe um identificador inteiro positivo.' }
    });
  }
  try {
    const excluiu = await FonteDestino.excluir(id, req.auth.id);
    if (!excluiu) {
      return res.status(404).json({ erro: 'Fonte/destino não encontrada.' });
    }
    return res.status(204).end();
  } catch (erro) {
    if (erro.code === 'ER_ROW_IS_REFERENCED_2') {
      return res.status(409).json({
        erro: 'Fonte/destino possui lançamentos vinculados.'
      });
    }
    return next(erro);
  }
});

module.exports = router;
```

Para integrar o exemplo, montar o Router em `/fontes-destinos`. O `server.js` deve carregar o `.env` antes dos módulos de autenticação e terminar com o tratamento de erros abaixo, conforme o padrão do projeto:

```javascript
app.use('/fontes-destinos', require('./routes/fontesDestinos'));
app.use((req, res) => {
  return res.status(404).json({ erro: 'Rota não encontrada.' });
});
app.use((erro, req, res, next) => {
  if (erro.name === 'UnauthorizedError') {
    return res.status(401).json({ erro: 'Token ausente, inválido ou expirado.' });
  }
  console.error(erro);
  return res.status(500).json({ erro: 'Erro interno do servidor.' });
});
```

`app` é a instância Express já criada no servidor. Os trechos são material de estudo e não foram aplicados a uma API existente.

**Resultado:** recurso próprio sem vínculos → 204 sem corpo; inexistente ou de outro usuário → 404; próprio com lançamentos vinculados → 409, conforme RF5; falha inesperada → 500 genérico, com detalhe apenas no log. A proteção JWT, a validação do identificador e o tratamento de vínculos completam o contexto; os três defeitos pedidos continuam sendo interpolação, SQL na rota e ausência de `return`.

## Treino 3 — R$ 250,00 em sete parcelas

Pelo RF6, primeiro convertemos o total para **25.000 centavos**:

```text
base = piso(25.000 / 7) = 3.571 centavos
resto = 25.000 − (3.571 × 7) = 3 centavos
primeira = 3.571 + 3 = 3.574 centavos
demais = 3.571 centavos cada
```

| Parcela | Centavos | Valor |
|---|---:|---:|
| 1/7 | 3.574 | R$ 35,74 |
| 2/7 | 3.571 | R$ 35,71 |
| 3/7 | 3.571 | R$ 35,71 |
| 4/7 | 3.571 | R$ 35,71 |
| 5/7 | 3.571 | R$ 35,71 |
| 6/7 | 3.571 | R$ 35,71 |
| 7/7 | 3.571 | R$ 35,71 |
| **Total** | **25.000** | **R$ 250,00** |

Conferência exata: **3.574 + 6 × 3.571 = 25.000 centavos**. Todo o resto vai para a primeira parcela; não se distribui um centavo por várias parcelas.

## Treino 4 — Compra no dia do fechamento

Compra em **18/08/2026**, total de **25.000 centavos**, em duas parcelas de **12.500 centavos**. Cartão fecha no dia **18** e vence no dia **3**.

1. A compra ocorre no dia do fechamento: a primeira parcela fecha no mês seguinte, setembro.
2. A segunda parcela fecha um mês depois da primeira, outubro.
3. Como `3 <= 18`, cada vencimento fica no mês posterior ao respectivo fechamento.
4. A referência é o mês do **vencimento**.

| Parcela | Valor | Fechamento | Vencimento | Referência AAAA-MM |
|---|---:|---|---|---|
| 1/2 | R$ 125,00 | 18/09/2026 | 03/10/2026 | **2026-10** |
| 2/2 | R$ 125,00 | 18/10/2026 | 03/11/2026 | **2026-11** |

Conferência: **12.500 + 12.500 = 25.000 centavos = R$ 250,00**.

## Treino 5 — Fechamento no dia 31 e fevereiro

Compra em **31/01/2026**, total de **12.000 centavos**, em duas parcelas de **6.000 centavos**. Cartão fecha no dia **31** e vence no dia **8**.

1. A compra é no dia do fechamento: o primeiro mês de fechamento é fevereiro.
2. Fevereiro de 2026 tem **28 dias**. Ajustar 31/02 para **28/02**, sem avançar para março.
3. A segunda parcela fecha em março; como março tem 31 dias, o fechamento volta a **31/03**.
4. Como `8 <= 31`, os vencimentos ficam nos meses seguintes aos fechamentos.

| Parcela | Valor | Fechamento | Vencimento | Referência AAAA-MM |
|---|---:|---|---|---|
| 1/2 | R$ 60,00 | **28/02/2026** | 08/03/2026 | **2026-03** |
| 2/2 | R$ 60,00 | **31/03/2026** | 08/04/2026 | **2026-04** |

Conferência: **6.000 + 6.000 = 12.000 centavos = R$ 120,00**. O ajuste de fevereiro não altera permanentemente o dia de fechamento configurado no cartão.

Nos treinos 4 e 5, todas as parcelas são gravadas com a **data original da compra**; `parcela_atual` determina o deslocamento da fatura. Alterar também a data de cada parcela deslocaria os meses duas vezes.

## Treino 6 — Integridade e concorrência

Duas requisições concorrentes podem executar `SELECT` e ambas concluir que o registro não existe.  
Entre a consulta e a inserção ocorre uma condição de corrida: as duas tentam gravar o mesmo valor.  
A constraint `UNIQUE` no banco impede que as duas inserções duplicadas sejam confirmadas.  
A API trata `ER_DUP_ENTRY` como **409 Conflict**; o `SELECT` prévio não substitui essa garantia.
