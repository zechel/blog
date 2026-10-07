---
title: "Meu fork do postgres_dba: um toolkit de DBA que posso entregar a um cliente"
date: 2026-10-06T20:00:00-03:00
description: "Fiz um fork do postgres_dba, de Nikolay Samokhvalov, e corrigi oito problemas que tornavam arriscado usá-lo em ambientes de clientes: índices, roles, senhas, entradas e testes."
tags: ["postgresql", "dba", "open-source", "segurança"]
categories: ["Infraestrutura"]
toc: true
math: false
comments: true
draft: false
---

Fiz um fork do postgres_dba e corrigi oito problemas que tornavam arriscado usá-lo em ambientes de clientes. O resultado está em [github.com/zechel/postgres_dba](https://github.com/zechel/postgres_dba).

Quando chego num ambiente novo, quero respostas rápidas: o que está lento, o que está inchado, quais índices sobram. O postgres_dba entrega isso direto no `psql`, sem instalar nada no servidor. Mas um relatório que sugere `DROP INDEX` ou uma rotina que mexe em roles precisa estar certo, porque alguém vai seguir o que ele diz.

Neste post conto de onde vem o projeto, o que encontrei numa revisão de prontidão e o que mudei.

## De onde vem: o postgres_dba do Nikolay Samokhvalov

Meu repositório é um fork direto do [NikolayS/postgres_dba](https://github.com/NikolayS/postgres_dba), criado por Nikolay Samokhvalov (Postgres.ai) em maio de 2017 e distribuído sob a licença BSD 3-Clause.

O projeto se define como _"the missing set of useful tools for Postgres DBA and mere mortals"_. Na prática, é uma coleção de consultas SQL organizadas num menu interativo do `psql`. Você digita `:dba`, escolhe um código como `b1` ou `i2` e recebe o relatório.

Três características me fizeram adotá-lo:

- **Nada a instalar no servidor.** Tudo roda no cliente, então funciona em RDS, Cloud SQL e servidores onde não tenho acesso ao sistema operacional.
- **Consultas consagradas.** Muitas vêm de trabalhos da comunidade, como as estimativas de bloat do ioguix, os pg-utils da Data Egret e os pgx_scripts da PostgreSQL Experts.
- **Um menu só.** Os scripts `start.psql` e `warmup.psql` são gerados a partir dos arquivos em `sql/` por `init/generate.sh`.

O crédito pela ideia e pela maior parte das consultas é do upstream. O que descrevo a seguir é o que mudei em cima dele.

## Por que um fork, e o que ele já tinha mudado

O fork nasceu para atender ao meu dia a dia com clientes e evoluiu em três etapas. A mais recente, e o foco deste post, foi corrigir o que uma revisão de prontidão encontrou.

| Quando | Etapa | O que mudou |
| --- | --- | --- |
| Out/2026 | Prontidão para clientes | Correção dos achados F01–F08 e testes de comportamento no CI (detalhes abaixo) |
| Ago/2026 | Expansão e integração seletiva do upstream | Novos relatórios `a3`, `a4`, `i7`, `r2`, `v3` e `b6`; progresso de `CREATE INDEX` em `p1`; parâmetros de storage em `t2`; diagnóstico de WAL em `0`; CI com matriz PostgreSQL 14–18 |
| Dez/2025–Jan/2026 | Adaptação ao meu uso | Relatório de replicação `r1`, roles movidas para o prefixo `u`, `t1` reescrito como relatório de parâmetros, ajustes em lock trees e `pg_stat_statements` |

Na integração de agosto não fiz um merge completo do upstream. Trouxe só as mudanças aprovadas, em commits pequenos e reversíveis, com a origem registrada em cada um.

Em 1º de outubro de 2026 revisei o toolkit com uma pergunta: dá para entregar isso a um cliente? A revisão encontrou oito problemas reproduzíveis.

| Achado | Prioridade | Problema |
| --- | --- | --- |
| F01 | P1 | `i5` sugeria `DROP INDEX` que podia remover garantias de unicidade |
| F02 | P1 | `u2` mantinha `SUPERUSER` e `LOGIN` quando o operador respondia "não" |
| F03 | P1 | Respostas inválidas podiam conceder `SUPERUSER` |
| F04 | P1 | Senhas vinham de `random()` e apareciam em mensagens do servidor |
| F05 | P1 | `i2` e `i3` confundiam índices com semânticas diferentes |
| F06 | P2 | Entradas do usuário eram interpoladas no SQL sem escape |
| F07 | P2 | O gerador do menu podia sobrescrever arquivos no diretório de quem o chamava |
| F08 | P2 | `a2` mostrava duração negativa para as consultas |

Nenhum deles aparece num teste de "o relatório roda sem erro". Todos aparecem quando alguém segue o que o relatório diz.

## Melhoria 1: índices redundantes que não quebram unicidade

Os relatórios `i2` (redundantes), `i3` (duplicados) e `i5` (migração com `DROP` e `UNDO`) agora só sugerem remover um índice quando outro entrega tudo o que ele entrega.

Na versão anterior, três casos davam errado em tabelas de teste:

- Com `(a)` e `(a,b)`, o `i5` dizia que `(a,b)` era redundante a `(a)`. A direção da cobertura estava invertida.
- Com um índice comum `(a)` e um `UNIQUE (a)`, sugeria remover o único. Apliquei esse `DROP` e a tabela passou a aceitar duas linhas com `a = 1`.
- A comparação de `indkey` como texto confundia a coluna 1 com a coluna 10.

A nova regra compara arrays do catálogo, não texto. Um índice B só é redundante a A, na mesma tabela, quando:

1. Usam o mesmo método de acesso, as mesmas expressões e o mesmo predicado.
2. As colunas-chave de B, com operator classes, collations e ordenação (`ASC`/`DESC`, `NULLS FIRST`/`LAST`), são um prefixo das de A. Fora do btree, exige igualdade exata.
3. As colunas `INCLUDE` de B estão em A.
4. Se B é único, A também é único, imediato, nas mesmas colunas e com o mesmo `NULLS [NOT] DISTINCT`.
5. B não é chave primária e não sustenta uma constraint.

O trecho central do `i2` fica assim:

```sql
and b.key_attnums = a.key_attnums[1:b.indnkeyatts]
and b.opclasses   = a.opclasses[1:b.indnkeyatts]
and b.collations  = a.collations[1:b.indnkeyatts]
and b.options     = a.options[1:b.indnkeyatts]
and b.include_attnums operator(pg_catalog.<@) a.all_attnums
```

O `operator(pg_catalog.<@)` explícito tem história: o CI pegou que, com a extensão `intarray` instalada, `smallint[] <@ smallint[]` fica ambíguo e o relatório falhava. Hoje os testes de índices rodam com `intarray` instalada.

Quando dois índices são equivalentes, só o mais novo é sugerido, e o mais antigo fica. O `i3` agrupa apenas índices idênticos em tudo, inclusive unicidade e ordenação. O `i5` cita os nomes corretamente e gera o `UNDO` com `CREATE UNIQUE INDEX CONCURRENTLY` quando o original era único. A lista de índices "não usados" também deixou de incluir índices únicos, chaves primárias e índices de constraints.

## Melhoria 2: roles com atributos explícitos e senhas que não vazam

As rotinas `u1` (criar role) e `u2` (trocar senha) agora fazem exatamente o que o operador respondeu e geram senhas de fonte criptográfica.

No código anterior, quatro coisas podiam dar errado:

- **`u2` ignorava o "não".** O ramo negativo gerava `ALTER ROLE nome` sem `NOSUPERUSER` nem `NOLOGIN`. Uma role superuser continuava superuser.
- **Qualquer resposta desconhecida virava "sim".** Responder `nao` à pergunta de superuser criava uma role com `rolsuper = true`.
- **Senhas previsíveis.** A senha vinha de `random()`. Duas sessões com o mesmo `setseed(0.125)` geraram a mesma senha para roles diferentes.
- **Senha no log.** `RAISE DEBUG` mostrava o SQL com a senha e `RAISE INFO` a senha em texto claro. Com `log_min_messages = info`, ela foi parar no log do servidor.

As correções:

- As respostas aceitas são uma lista fechada, em inglês e português: `1`, `y`, `yes`, `t`, `true`, `s`, `sim` ou `0`, `n`, `no`, `f`, `false`, `nao`, `não`. Qualquer outra aborta sem alterar nada.
- O estado final sempre corresponde às respostas. `NOSUPERUSER` só é emitido para uma role que hoje é superuser, porque o PostgreSQL reserva qualquer cláusula `SUPERUSER` a superusers; assim um operador com `CREATEROLE` continua podendo trocar senhas de roles comuns.
- A senha vem dos bytes de `gen_random_uuid()`, que usa `pg_strong_random()`. Os bytes de versão e variante são descartados, e uma amostragem por rejeição mantém todos os caracteres equiprováveis. `setseed()` não tem efeito sobre ela.
- A senha nunca passa por `RAISE`. Ela volta ao `psql` por um GUC local à transação, é lida com `\gset` e mostrada uma vez com `\echo`.
- Se algo falha, `\if :ERROR` faz `rollback` e avisa que nada mudou. O erro é relançado só com código e mensagem, sem o comando que contém a senha.

```sql
while length(pwd) < 16 loop
  random_bytes := uuid_send(gen_random_uuid());
  for i in 0..15 loop
    continue when i in (6, 8);
    b := get_byte(random_bytes, i);
    if b < 4 * length(allowed) and length(pwd) < 16 then
      pwd := pwd || substr(allowed, b % length(allowed) + 1, 1);
    end if;
  end loop;
end loop;
```

{{< callout type="warning" >}}
Um risco continua e está documentado no README: o `CREATE ROLE` executado contém a senha. Com `pg_stat_statements.track = all` ou `auto_explain` registrando comandos aninhados, ela pode ser capturada.
{{< /callout >}}

## Melhoria 3: entradas escapadas, gerador seguro e duração correta

Três correções menores fecham o restante: tudo o que o operador digita vira literal SQL escapado, o gerador do menu não toca arquivos fora do repositório e o `a2` mostra durações positivas.

**Entradas (F06).** O menu, o nome da role, os segundos do `a2` e o PID do `k1`/`k2` eram colados no SQL por concatenação de aspas. Um nome com apóstrofo quebrava a rotina. No menu, uma entrada com apóstrofo e um `SELECT` extra executava esse `SELECT`. Não é escalada de privilégio, porque o operador já tem a sessão, mas é SQL inesperado rodando.

Agora todas as entradas usam a sintaxe `:'var'` do `psql`, que escapa o literal, e os valores numéricos são validados antes de qualquer função administrativa. Antes e depois, no `k1`:

```sql
-- antes
SELECT pg_cancel_backend(:postgres_pid);

-- depois
select :'postgres_pid' ~ '^[0-9]+$' as postgres_dba_valid_pid \gset
\if :postgres_dba_valid_pid
  SELECT pg_cancel_backend(:'postgres_pid'::int);
\else
  \echo 'Invalid PID:' :'postgres_pid'
\endif
```

**Gerador (F07).** O `init/generate.sh` esvaziava `start.psql` e `warmup.psql` antes de entrar na raiz do repositório. Executado de outro diretório, apagava os arquivos com esses nomes ali e terminava com código zero. Agora ele resolve a raiz primeiro, roda com `set -euo pipefail`, escreve em arquivos temporários criados com `mktemp` e só então os move para o lugar.

**Duração no `a2` (F08).** O cálculo `age(query_start, clock_timestamp())` estava invertido, e uma sessão com `pg_sleep(2)` aparecia com duração negativa. Agora é `clock_timestamp() - query_start AS duration`, ordenado da mais longa para a mais curta. Uma resposta vazia significa 0 segundos.

## Melhoria 4: testes que verificam o efeito, não só a execução

Adicionei `test/behavior.sh`, que hoje tem 39 verificações cobrindo todos os achados. Na primeira versão, 23 das 32 verificações falhavam contra o código original. Contra o novo, todas passam.

O CI já rodava cada relatório no PostgreSQL 14 a 18 e falhava em erro de SQL. Isso prova que a consulta executa, mas não que o resultado está certo. Os novos testes olham o efeito:

- Montam uma tabela com índices armadilha, **aplicam de verdade** os `DROP` sugeridos pelo `i5` e confirmam que uma inserção duplicada continua sendo recusada.
- Executam `u1` e `u2` com respostas afirmativas, negativas (inclusive `nao`) e inválidas, como superuser e como um operador com `CREATEROLE`, e conferem `rolsuper` e `rolcanlogin` em `pg_roles`.
- Geram senhas em duas sessões com o mesmo `setseed()` e verificam que são diferentes, e confirmam que nenhum erro mostra a senha.
- Passam apóstrofos e SQL extra nos campos do menu, do `a2`, do `k1` e do `k2`.
- Conferem que o `a2` mostra duração positiva para uma sessão com `pg_sleep`.
- Rodam o gerador a partir de outro diretório e verificam que os arquivos de lá ficam intactos.

O script roda no GitHub Actions como o passo "Run behaviour tests", na mesma matriz PostgreSQL 14–18, ao lado da suíte que executa 33 relatórios como superuser e como `pg_monitor`, nos modos normal e wide.

Os testes também pegaram coisas que eu mesmo introduzi: o problema do `intarray` e uma regressão em que o `u2` deixava de funcionar para operadores sem superuser. É exatamente para isso que eles existem.

## Como experimentar

Você precisa de um `psql` 10 ou mais novo e de um servidor PostgreSQL 14 a 18. Não é preciso instalar nada no servidor.

1. Clone o repositório e registre o alias `:dba` no seu `~/.psqlrc`:

    ```bash
    git clone https://github.com/zechel/postgres_dba.git
    cd postgres_dba
    printf "%s %s %s %s\n" \\echo 'postgres_dba installed. Use ":dba" to see menu' >> ~/.psqlrc
    printf "%s %s %s %s\n" \\set dba \'\\\\i $(pwd)/start.psql\' >> ~/.psqlrc
    ```

2. Para diagnóstico de rotina, conecte com uma role que tenha `pg_monitor`, `CONNECT` no banco e `SELECT` nas tabelas que vai inspecionar. Só `k1`, `k2`, `u1` e `u2` alteram estado, e `b3`, `b4` e `b6` leem tabelas inteiras com `pgstattuple`.
3. Abra o `psql`, digite `:dba` e escolha um relatório. Comece por `0` (visão do nó) e `1` (bancos).

Antes de aplicar qualquer `DROP` sugerido pelo `i5`, leia as definições dos dois índices. Em réplicas, o uso de índices costuma ser bem diferente do primário.

## Para fechar

Um relatório de DBA só é útil se der para confiar no que ele manda fazer. Esse foi o critério de todas as mudanças.

O mérito do toolkit continua sendo do Nikolay Samokhvalov e de quem contribuiu com as consultas originais. Se você usa o postgres_dba original, vale conferir se os mesmos casos afetam a sua cópia, principalmente o `i5` e as rotinas de roles.

O fork está em [github.com/zechel/postgres_dba](https://github.com/zechel/postgres_dba). Issues e pull requests são bem-vindos.
