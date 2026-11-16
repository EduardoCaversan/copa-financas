# Carta-bug — O filtro está lá, mas o isolamento falha

**Instituição:** Universidade Tecnológica Federal do Paraná — UTFPR  
**Disciplina:** Programação Web Back-end  
**Aluno:** Eduardo Caversan da Silva Rocha  
**RA:** 2475391  
**Equipe:** ______________________________  
**Arquivo do exemplo:** `models/Relatorio.js`

## Contexto

Método interno que calcula os totais por fonte/destino e tipo, usados na montagem do relatório do RF8. `database/connection.js` exporta uma conexão ou pool com a interface de promises do `mysql2`. A rota protegida passa `req.auth.id` como `usuarioId`, valida `inicio` e `fim` como datas `AAAA-MM-DD`, com início não posterior ao fim, e encaminha erros ao middleware da aplicação. A tabela `lancamentos` segue o schema do Projeto 2.

## Trecho incorreto — 15 linhas

A numeração começa no `const` da primeira linha; as cercas Markdown não fazem parte do código.

```javascript
const db = require('../database/connection');
class Relatorio {
  static async totaisPorFonte(usuarioId, inicio, fim) {
    const [rows] = await db.execute(
      `SELECT fonte_destino_id, tipo, SUM(valor) AS total
         FROM lancamentos
        WHERE usuario_id = ? AND data BETWEEN ? AND ?
          AND tipo = 'entrada' OR tipo = 'saida'
        GROUP BY fonte_destino_id, tipo`,
      [usuarioId, inicio, fim]
    );
    return rows.map(r => ({ ...r, total: Number(r.total) }));
  }
}
module.exports = Relatorio;
```

## Resposta

**Linha incorreta:** 8 — `AND tipo = 'entrada' OR tipo = 'saida'`.

**Regra violada:** **RF3 — Isolamento dos dados entre usuários**: todas as consultas devem restringir os dados ao dono autenticado. A falha também compromete os totais por período do **RF8**, como consequência do mesmo erro de agrupamento lógico.

**Problema:** em SQL, `AND` tem precedência sobre `OR`. Assim, a condição equivale a `(dono correto AND período correto AND entrada) OR saída`. Qualquer saída é incluída, mesmo de outro usuário ou fora do período. Os placeholders impedem a interpolação de valores no SQL, mas não corrigem essa falha lógica.

**Consequência verificável:** em uma consulta de agosto do usuário A, considere somente três saídas: R$ 40,00 de A em agosto, R$ 90,00 de B em agosto e R$ 20,00 de A em julho. O trecho inclui as três e permite que a soma dos grupos de saída resulte em R$ 150,00. Apenas os R$ 40,00 deveriam participar. Isso expõe totais e identificadores de fontes/destinos alheios e distorce o relatório.

## Código corrigido

Os parênteses da linha 8 fazem os dois tipos obedecerem ao filtro de usuário e período. O módulo abaixo substitui integralmente o trecho, no contexto descrito.

```javascript
const db = require('../database/connection');
class Relatorio {
  static async totaisPorFonte(usuarioId, inicio, fim) {
    const [rows] = await db.execute(
      `SELECT fonte_destino_id, tipo, SUM(valor) AS total
         FROM lancamentos
        WHERE usuario_id = ? AND data BETWEEN ? AND ?
          AND (tipo = 'entrada' OR tipo = 'saida')
        GROUP BY fonte_destino_id, tipo`,
      [usuarioId, inicio, fim]
    );
    return rows.map(r => ({ ...r, total: Number(r.total) }));
  }
}
module.exports = Relatorio;
```

Agora, nenhuma alternativa do `OR` pode contornar `usuario_id` ou o período. O SQL permanece no model, os valores continuam parametrizados e os totais são convertidos de `DECIMAL` para números na saída do método. Sem lançamentos correspondentes, o método retorna `[]`; cabe à montagem do relatório representar os totais vazios.

**Referências:** [Copa, seção 13.4 — Carta-bug](https://costasilvati.github.io/IntroNodeJS/copa-api-financas.html#copa-carta-bug); [Projeto 2 — RF3 e RF8](https://costasilvati.github.io/IntroNodeJS/projeto-api-financas.html).
