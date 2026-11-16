# Entrega — Copa da API Finanças

**Aluno:** Eduardo Caversan da Silva Rocha  
**RA:** 2475391  
**Disciplina:** Programação Web Back-end — UTFPR

## Arquivos

| Arquivo | Finalidade |
|---|---|
| [01-carta-bug.md](01-carta-bug.md) | Documento principal editável: identificação, contexto, trecho de 15 linhas, erro na linha 8, RF violado, explicação e correção. |
| [01-carta-bug.pdf](01-carta-bug.pdf) | Versão da carta formatada para leitura e envio. |
| [02-gabarito-treinos.md](02-gabarito-treinos.md) | Material complementar: resolução dos seis exercícios com justificativas, implementação do DELETE, cálculos e tabelas de faturas. |
| [03-guia-revisao.md](03-guia-revisao.md) | Material complementar: regras do projeto, exemplos e perguntas com respostas para estudo individual. |
| `README.md` | Orientações de uso, fontes e limites das verificações. |

## O que enviar pelo Moodle

**Envie `01-carta-bug.pdf` como a carta-bug da equipe.** O enunciado exige uma carta por equipe, até a véspera da aula; não especifica extensão obrigatória. Se a atividade aceitar ou solicitar texto, o mesmo conteúdo está em `01-carta-bug.md`.

O campo **Equipe** permanece em branco porque o número/nome não foi informado. Preencha com a identificação real antes de enviar; se editar o Markdown, exporte novamente o PDF para mantê-los equivalentes. Não invente integrantes ou numeração.

Os treinos, o guia e este README são **materiais complementares de preparação**, não entregas adicionais exigidas pela seção 13.4. Não é necessário enviar a pasta inteira. O envio ao Moodle não foi realizado automaticamente.

## Fontes e limitação de acesso ao projeto

A pasta de trabalho estava vazia, sem arquivos de aplicação e sem repositório Git. Portanto, não foi possível examinar rotas, models, middlewares, autenticação, parcelamento ou relatórios da implementação anterior. Os exemplos foram construídos a partir do schema, da arquitetura e das regras oficiais; não representam uma auditoria do código da equipe.

O acesso direto ao site falhou durante a consulta. Foram lidos integralmente o roteiro e a especificação no **repositório público da própria professora**, incluindo o HTML publicado da Copa e os textos-fonte:

- [Enunciado da Copa — página oficial](https://costasilvati.github.io/IntroNodeJS/copa-api-financas.html), especialmente seções 13.3 e 13.4.
- [HTML da Copa no repositório oficial](https://github.com/costasilvati/IntroNodeJS/blob/main/docs/copa-api-financas.html).
- [Texto-fonte do roteiro da Copa](https://github.com/costasilvati/IntroNodeJS/blob/main/13-copa-api-financas-roteiro-estudo.Rmd), versão consultada com blob SHA `d2a303059a52b8443c326f33b76d71ea6d831d2c`.
- [Projeto 2 — página oficial](https://costasilvati.github.io/IntroNodeJS/projeto-api-financas.html).
- [Texto-fonte do Projeto 2](https://github.com/costasilvati/IntroNodeJS/blob/main/12-projeto-api-controle-financeiro.Rmd), versão consultada com blob SHA `4ce9499830f2da9ba114722441a96a27dde955d9`.

Consulta em **15/11/2026**. A seção pública do gabarito ainda exibia o aviso de disponibilização após 13/11/2026 às 23:59; o texto-fonte continha as respostas comentadas. As resoluções foram calculadas pelas regras e confrontadas com esse conteúdo, sem supor que a publicação visível já estivesse atualizada.

## Revisão e limites

- Carta com 15 linhas de JavaScript no trecho incorreto, um erro principal de precedência lógica e correção restrita à linha 8.
- Três possibilidades de bug avaliadas antes da escolha. O exemplo foi redigido para esta atividade e não reproduz os trechos fornecidos; não há acesso às cartas de outras equipes para garantir ausência de coincidência.
- RF1–RF9, schema, padrão de respostas, estrutura e checklist final consultados. O nome oficial da pasta de middleware é `middleware/`, no singular.
- Valores conferidos em centavos inteiros e datas conferidas com a regra de faturas, incluindo a igualdade no fechamento e fevereiro de 2026.
- Sintaxe dos blocos JavaScript verificada com Node.js. A consulta da carta foi exercitada em SQLite com dados controlados para verificar precedência, isolamento, período inclusivo e conjunto vazio; isso não valida diferenças de tipos ou de driver do MySQL.
- A chamada ao model foi simulada para conferir parâmetros e conversão de `SUM` textual em número. O handler do DELETE foi exercitado com doubles locais para 204, 404, 409, identificador inválido (400) e encaminhamento de erro inesperado; autenticação JWT real e conexão MySQL não foram executadas.
- Não foi executada uma API Express com MySQL real, pois a aplicação e suas dependências não estavam disponíveis. As verificações locais não equivalem a testes de integração da equipe.
- Nenhum código de aplicação foi alterado. Nenhum push foi realizado. Não houve commits incrementais porque a pasta não era um repositório Git; não foi iniciado um repositório apenas para estes documentos.

O PDF foi gerado com o Chrome já disponível no ambiente, sem instalar dependências. Suas duas páginas foram renderizadas e inspecionadas visualmente para conferir legibilidade, acentuação e ausência de cortes nos códigos. Os códigos nos documentos são exemplos de estudo, com as dependências e premissas descritas em cada contexto.
