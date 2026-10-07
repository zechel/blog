---
title: "Transaction ID Wraparound no PostgreSQL (1/4): fundamentos, XIDs, congelamento e Visibility Map"
date: 2026-10-07T08:55:00-03:00
description: "Por que o wraparound acontece: MVCC, XIDs de 32 bits, congelamento, relfrozenxid, Visibility Map e MultiXact, com foco nas versões 14 a 18."
tags: ["postgresql", "dba", "vacuum", "wraparound"]
categories: ["Infraestrutura"]
toc: true
math: false
comments: true
draft: false
---

> **Parte 1 de 4 — Fundamentos, XIDs, congelamento, Visibility Map e MultiXact**
>
> **Navegação da série:** **Parte 1** · Parte 2 (em breve) · Parte 3 (em breve) · Parte 4 (em breve)

> ### Antes de começar
>
> Este artigo ficou longo. Bem longo.
>
> Ele também levou bastante tempo para ser escrito, revisado e testado. A intenção não era produzir apenas mais uma explicação rápida sobre o *transaction ID wraparound*, mas reunir, em um único material, os diferentes aspectos desse problema que encontrei ao longo dos anos administrando ambientes PostgreSQL — uma trajetória que começou ainda na versão 8.
>
> Nesse período, vi o wraparound aparecer de formas muito diferentes: às vezes começava com uma idade de XID que parecia inofensiva; em outras, surgia no meio de um autovacuum pesado, de uma configuração feita apenas para ganhar tempo ou simplesmente porque alguma sessão ficou aberta por tempo demais. Também encontrei muita orientação antiga sendo seguida no automático, mesmo depois de o PostgreSQL já ter mudado bastante.
>
> Por isso, o texto vai além da definição do problema. Ele passa pela mecânica interna, pelas diferenças entre versões, pelo Visibility Map, por MultiXacts, parâmetros, monitoramento, planejamento de capacidade, laboratórios e procedimentos de emergência.
>
> Você não precisa ler tudo de uma vez. Use o sumário, avance para a parte que corresponde ao problema que está enfrentando e volte aos fundamentos quando precisar entender por que determinado comportamento acontece.
>
> O objetivo é que este material sirva tanto para quem está conhecendo o assunto quanto para quem precisa tomar uma decisão técnica com um cluster em produção.

> **Escopo desta revisão:** os procedimentos operacionais foram validados para as versões atualmente suportadas do PostgreSQL, da 14 à 18, com ênfase no PostgreSQL 18. Diferenças históricas de versões anteriores aparecem apenas para contextualização, e o PostgreSQL 19 é tratado somente na seção 18, com marcação explícita de conteúdo beta. Sempre confirme a versão exata do servidor antes de executar um runbook.
>
> **Diferença importante entre versões:** as mensagens de aviso e de erro relacionadas a wraparound foram reescritas no ciclo do PostgreSQL 17 e a mudança não foi retroportada. Em PostgreSQL 14, 15 e 16 o servidor ainda recomenda modo mono-usuário. Essa recomendação é obsoleta; veja a seção 13, Fase 5.
>
> **Critério de fontes:** a documentação oficial e as notas de versão do PostgreSQL são normativas para comportamento, configuração e recuperação. O InterDB é utilizado como apoio conceitual e histórico, não como substituto da documentação da versão instalada.

---

> ### 🚨 Está em incidente agora?
>
> Se o seu banco já recusa escritas com `database is not accepting commands...`, vá direto para a **seção 16 — Runbook de emergência**.
>
> Três coisas para não fazer antes de chegar lá: **não reinicie o servidor**, **não entre em modo mono-usuário** e **não execute `VACUUM FULL`**. Se o log sugerir single-user mode, essa mensagem é histórica — veja a seção 13, Fase 5.

---

## Sumário

1. [Por que este texto existe](#1-por-que-este-texto-existe)
2. [Fundamentos: MVCC e o número que cada versão de linha carrega](#2-fundamentos-mvcc-e-o-número-que-cada-versão-de-linha-carrega)
3. [O XID: 32 bits e aritmética circular](#3-o-xid-32-bits-e-aritmética-circular)
4. [O problema: quando o passado parece futuro](#4-o-problema-quando-o-passado-parece-futuro)
5. [A solução: congelamento](#5-a-solução-congelamento)
6. [A contabilidade: relfrozenxid, datfrozenxid e age()](#6-a-contabilidade-relfrozenxid-datfrozenxid-e-age)
7. [Visibility Map: onde está a maior parte da eficiência](#7-visibility-map-onde-está-a-maior-parte-da-eficiência)
8. [MultiXact: o contador paralelo](#8-multixact-o-contador-paralelo)
9. Parâmetros e limites efetivos
10. O que limita o VACUUM: horizontes e retenções
11. Monitoramento e planejamento de capacidade
12. Laboratório reproduzível
13. Anatomia de um incidente
14. Anti-padrões
15. Otimizações e melhorias
16. Runbook de emergência
17. O fantasma do PostgreSQL 9.5
18. O que mudou em cada versão
19. Checklist operacional
20. Referências

---

## 1. Por que este texto existe

O *transaction ID wraparound* pertence a uma classe de risco operacional que combina três características desagradáveis:

1. **Pode permanecer discreto por muito tempo.** O banco pode continuar atendendo normalmente enquanto a idade dos XIDs cresce.
2. **É previsível.** O limite decorre de aritmética de 32 bits, e a margem pode ser estimada a partir da idade atual e da taxa de consumo de XIDs.
3. **Fica mais caro quanto mais tarde for tratado.** Quando o servidor começa a emitir os avisos finais, a margem pode representar poucas horas ou poucos dias em ambientes de alta taxa transacional.

O PostgreSQL contém salvaguardas para impedir que versões antigas de linhas se tornem invisíveis depois da volta do contador. Se a manutenção preventiva não consegue avançar os marcos de congelamento, o servidor passa a recusar operações que precisem atribuir novos XIDs antes que ocorra perda lógica de visibilidade.

Nas versões suportadas do PostgreSQL, isso **não significa automaticamente parar o servidor ou entrar em modo mono-usuário**. Mesmo quando novas escritas são recusadas, um `VACUUM` normal ainda pode ser executado. A recuperação correta começa por remover os bloqueadores do horizonte e executar o menor trabalho necessário para avançar os marcos. O procedimento completo está na seção 16.

A boa notícia é que o risco é mensurável e evitável. O problema costuma surgir menos por falta de mecanismo no PostgreSQL e mais por ausência de monitoramento, configuração incompatível com a carga ou retenções que impedem o `VACUUM` de progredir.

Este texto pressupõe experiência geral em infraestrutura, desenvolvimento ou dados, mas não exige conhecimento prévio profundo das estruturas internas do PostgreSQL.

### Como ler este guia

Os blocos marcados como **essenciais para operação** são úteis para qualquer pessoa responsável pelo ambiente. As inspeções com `pageinspect`, manipulação de `pg_resetwal` e leitura de cabeçalhos de tupla são aprofundamentos destinados a laboratório descartável.

> **Regra de segurança:** nenhum comando com `pg_resetwal` deve ser executado em produção. A ferramenta é apresentada somente para criar estados artificiais em um container que será destruído depois.

### Antes de começar: o trauma das versões antigas

Boa parte da reputação do wraparound foi formada em versões nas quais um vacuum agressivo precisava reler muito mais páginas e havia menos ferramentas para acompanhar o progresso. O PostgreSQL 9.6 adicionou o bit `all_frozen` ao Visibility Map e `pg_stat_progress_vacuum`; versões posteriores acrescentaram controle de índices, failsafe, melhorias de freeze e, no PostgreSQL 18, *eager freezing*.

A matemática de 32 bits não desapareceu. O que mudou foi a capacidade de amortizar, observar e controlar o trabalho. A seção 17 mostra essa evolução sem transformar melhorias modernas em promessa de risco zero.

---

## 2. Fundamentos: MVCC e o número que cada versão de linha carrega

### O problema que o MVCC resolve

Imagine dois usuários atuando ao mesmo tempo:

- **Ana** inicia um relatório que percorre milhões de pedidos.
- **Bruno**, durante a execução do relatório, altera o valor de um pedido.

Se a consulta de Ana misturar parte do estado anterior com parte do estado posterior à alteração, o resultado pode ficar internamente inconsistente.

O PostgreSQL usa **MVCC** (*Multi-Version Concurrency Control*). Em vez de sobrescrever uma linha em uso por outros snapshots, ele mantém versões físicas diferentes da mesma linha lógica. Ana pode continuar vendo a versão adequada ao snapshot da consulta, enquanto Bruno cria uma nova versão.

Uma regra prática útil é:

> Em operações DML comuns, leituras simples normalmente não bloqueiam escritas, e escritas normalmente não bloqueiam leituras simples.

Essa regra tem exceções. Um lock `ACCESS EXCLUSIVE`, adquirido por operações como `TRUNCATE`, `VACUUM FULL` e várias formas de `ALTER TABLE`, bloqueia até um `SELECT` comum. Consultas com `FOR UPDATE`, `FOR SHARE` e variantes também participam de conflitos de lock em nível de linha.

### `xmin` e `xmax`

Cada versão física de linha, chamada de **tupla**, possui campos de sistema no cabeçalho:

| Campo | Significado simplificado |
|---|---|
| `xmin` | XID da transação que criou a versão |
| `xmax` | XID ou MultiXact associado à remoção, substituição ou lock da versão |

O modelo introdutório costuma dizer que `xmax = 0` significa “a linha ainda está viva”. Isso é útil no primeiro contato, mas não é uma regra completa: `xmax` também pode representar locks de linha e MultiXacts. A decisão de visibilidade considera `xmin`, `xmax`, o estado das transações e o snapshot.

Outro detalhe importante é o nível de isolamento:

- Em `READ COMMITTED`, padrão do PostgreSQL, cada comando obtém seu próprio snapshot.
- Em `REPEATABLE READ` e `SERIALIZABLE`, o snapshot permanece estável durante a transação, sujeito às regras de cada nível.

Portanto, a pergunta correta não é sempre “isso estava confirmado quando a transação começou?”, mas “isso é visível para o snapshot usado por este comando ou transação?”.

### Observando uma atualização

```sql
CREATE TABLE demo (id int, texto text);

INSERT INTO demo VALUES (1, 'primeira');
INSERT INTO demo VALUES (2, 'segunda');

SELECT xmin, xmax, id, texto
FROM demo
ORDER BY id;
```

Saída ilustrativa:

```text
 xmin  | xmax | id |  texto
-------+------+----+----------
 43120 |    0 |  1 | primeira
 43121 |    0 |  2 | segunda
```

Agora atualize uma linha:

```sql
UPDATE demo
SET texto = 'alterada'
WHERE id = 1;

SELECT xmin, xmax, id, texto
FROM demo
ORDER BY id;
```

```text
 xmin  | xmax | id |  texto
-------+------+----+----------
 43122 |    0 |  1 | alterada
 43121 |    0 |  2 | segunda
```

A versão antiga da linha `id = 1` não foi modificada no lugar. Ela recebeu metadados que indicam sua substituição, e uma nova versão foi criada. A versão antiga só poderá ser removida quando nenhum snapshot relevante ainda puder precisar dela.

> **Ponto-chave:** a visibilidade de versões de linhas depende de interpretar identificadores de transação. O wraparound é perigoso porque, sem congelamento, a ordem relativa desses identificadores deixa de ser inequívoca após aproximadamente metade do espaço circular.

---

## 3. O XID: 32 bits e aritmética circular

### XID virtual, XID interno e `xid8`

Toda transação recebe um identificador virtual. Um XID não virtual, o `TransactionId` interno, é atribuído quando a transação precisa escrever no banco. Funções como `pg_current_xact_id()` também forçam essa atribuição.

Para verificar se a transação já recebeu XID sem consumir um desnecessariamente, use:

```sql
SELECT pg_current_xact_id_if_assigned();
```

Se a transação ainda for somente leitura e nenhuma operação tiver forçado a atribuição, o resultado será `NULL`.

```sql
SELECT pg_current_xact_id();
```

Essa função retorna um `xid8`: uma representação de 64 bits que inclui o epoch e não sofre wraparound durante a vida prática da instalação. Isso **não altera** o fato de que o XID armazenado nos cabeçalhos de tupla e usado internamente na comparação MVCC possui 32 bits.

As funções antigas da família `txid_*`, como `txid_current()`, continuam disponíveis por compatibilidade, mas a documentação atual prefere as variantes `pg_*` baseadas em `xid8`.

### Transações somente leitura e subtransações

Uma transação somente leitura normalmente não recebe XID não virtual. Existem duas ressalvas:

1. funções de inspeção como `pg_current_xact_id()` podem forçar a atribuição;
2. uma operação que inicialmente apenas lia pode receber um XID quando realiza a primeira escrita.

`SAVEPOINT` e blocos `EXCEPTION` de PL/pgSQL criam subtransações. Porém, uma subtransação somente leitura não recebe automaticamente um `subxid`. O `subxid` é atribuído quando ela tenta escrever; nesse momento, seus ancestrais também recebem XIDs não virtuais, se ainda não possuírem.

```sql
DO $$
BEGIN
  BEGIN
    INSERT INTO demo VALUES (99, 'x');
  EXCEPTION WHEN unique_violation THEN
    NULL;
  END;
END $$;
```

Esse padrão pode consumir mais XIDs que o número de transações de aplicação sugere, especialmente em loops que criam muitas subtransações com escrita. Acima de 64 subxids abertos por backend, o PostgreSQL também passa a depender mais de consultas a `pg_subtrans`, aumentando o custo de gerenciamento.

### O tamanho do contador

O XID interno normal é um inteiro sem sinal de 32 bits:

```text
2^32 = 4.294.967.296 valores
```

Os valores 0, 1 e 2 são reservados para significados especiais, incluindo `InvalidTransactionId`, `BootstrapTransactionId` e `FrozenTransactionId`.

Um sistema que consome 1.000 XIDs por segundo gera aproximadamente:

```text
1.000 × 86.400 = 86,4 milhões de XIDs por dia
```

Nesse ritmo, o espaço completo de 32 bits seria percorrido em cerca de 49,7 dias. A janela de comparação útil, porém, é aproximadamente metade disso.

### A aritmética circular

O contador dá a volta. Depois do maior XID normal, os valores retornam ao início do espaço disponível. Por isso, uma comparação numérica simples não basta para decidir se um XID é antigo ou futuro.

O PostgreSQL usa comparação modular. Para um XID de referência, aproximadamente metade do círculo é tratada como passado e metade como futuro:

```text
                         XID atual
                             │
                       ┌─────▼─────┐
                ┌──────┤  círculo  ├──────┐
                │      └───────────┘      │
                │                         │
       ~2^31 XIDs no passado      ~2^31 XIDs no futuro
                │                         │
                └────────────┬────────────┘
                         ponto oposto
```

> **Consequência:** o limite operacional não é quatro bilhões de transações. Uma versão de linha com XID normal precisa ser congelada antes de envelhecer aproximadamente dois bilhões de XIDs.

---

## 4. O problema: quando o passado parece futuro

Considere uma versão de linha criada pelo XID 100. Enquanto o contador avança menos de aproximadamente `2^31` posições, esse XID permanece no lado considerado passado.

Depois que a distância circular ultrapassa esse ponto, o mesmo valor pode ser interpretado como estando no futuro. Uma versão criada por uma transação “futura” não deve ser visível para o snapshot atual.

Sem proteção, a linha poderia parecer desaparecer logicamente, embora os bytes ainda estivessem no disco. O PostgreSQL evita esse cenário por meio de:

1. congelamento de versões antigas;
2. vacuums agressivos quando os marcos envelhecem;
3. autovacuum anti-wraparound, mesmo quando o autovacuum normal foi desabilitado;
4. warnings quando a margem se aproxima de aproximadamente 40 milhões de XIDs;
5. recusa de novas atribuições de XID quando restam menos de aproximadamente 3 milhões até o ponto crítico.

Os valores de 40 milhões e 3 milhões valem para as versões 14 a 18. O PostgreSQL 19, ainda em beta, eleva o limiar de aviso para aproximadamente 100 milhões; veja a seção 18.

Quando a proteção final é acionada, transações já em andamento podem continuar e novas transações somente leitura ainda podem ser iniciadas. Operações que precisam modificar registros ou atribuir XIDs falham. Isso preserva a integridade lógica, mas pode representar indisponibilidade funcional para a aplicação.

---

## 5. A solução: congelamento

### `FrozenTransactionId`

O `FrozenTransactionId`, valor reservado 2, é tratado como mais antigo que qualquer XID normal. Nas versões modernas, congelar uma tupla significa fazer com que o XID de inserção seja tratado como suficientemente antigo para todos os snapshots presentes e futuros.

Isso não torna a linha “eterna” nem impede alterações posteriores. A versão congelada continua válida **até ser atualizada ou removida**. O congelamento retira o `xmin` daquela versão da aritmética de wraparound; as demais regras de MVCC continuam valendo.

### Como o estado é armazenado

Até o PostgreSQL 9.3, o congelamento podia substituir fisicamente o `xmin` pelo valor 2. Desde o PostgreSQL 9.4, o `xmin` original é preservado e o estado congelado é representado por bits em `t_infomask`:

```text
HEAP_XMIN_COMMITTED = 0x0100
HEAP_XMIN_INVALID   = 0x0200
HEAP_XMIN_FROZEN    = 0x0300
```

A combinação dos dois bits é usada como sentinela de congelamento. Outros bits podem estar presentes no mesmo campo; por isso, a inspeção deve usar máscara, não igualdade direta.

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT
    lp,
    t_xmin,
    t_infomask,
    (t_infomask & 768) = 768 AS xmin_congelado
FROM heap_page_items(get_raw_page('demo', 0))
WHERE lp_off > 0;
```

Uma alternativa mais legível nas versões atuais é:

```sql
SELECT
    lp,
    t_xmin,
    heap_tuple_infomask_flags(t_infomask, t_infomask2)
FROM heap_page_items(get_raw_page('demo', 0))
WHERE lp_off > 0;
```

> `pageinspect` lê estruturas internas. Use apenas com privilégios adequados e em um ambiente no qual essa inspeção tenha sido autorizada.

### Quem congela

O `VACUUM` é responsável pelo congelamento. Há três comportamentos relevantes:

- **Vacuum normal:** concentra-se nas páginas que precisam de manutenção e pode pular páginas indicadas pelo Visibility Map.
- **Vacuum agressivo:** visita todas as páginas que ainda possam conter XIDs ou MultiXacts não congelados; páginas `all_frozen` podem ser puladas.
- **Eager freezing, no PostgreSQL 18:** um vacuum normal pode examinar parte das páginas `all_visible` que ainda não são `all_frozen`, amortizando o trabalho que seria concentrado em um vacuum agressivo futuro.

`VACUUM (FREEZE)` força limites de congelamento mais agressivos, equivalentes a definir `vacuum_freeze_min_age` e `vacuum_freeze_table_age` como zero para aquela execução. Ele é útil em cargas iniciais ou campanhas planejadas, mas **não é o comando recomendado para a primeira resposta a uma parada por esgotamento de XIDs**, porque pode fazer mais trabalho que o necessário para restaurar as escritas.

---

## 6. A contabilidade: `relfrozenxid`, `datfrozenxid` e `age()`

O PostgreSQL não percorre todas as tuplas sempre que precisa avaliar risco. Ele mantém marcos conservadores em catálogos.

### Por relação

`pg_class.relfrozenxid` é um limite inferior: XIDs anteriores a esse marco não devem permanecer como XIDs de inserção não congelados na relação.

Tabelas TOAST possuem seu próprio `relfrozenxid`. Para avaliar uma tabela junto com a TOAST associada:

```sql
SELECT
    c.oid::regclass AS relacao,
    greatest(
        age(c.relfrozenxid),
        coalesce(age(t.relfrozenxid), 0)
    ) AS maior_idade,
    age(c.relfrozenxid) AS idade_heap,
    age(t.relfrozenxid) AS idade_toast,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho_total
FROM pg_class c
LEFT JOIN pg_class t
       ON t.oid = c.reltoastrelid
WHERE c.relkind IN ('r', 'm')
ORDER BY maior_idade DESC
LIMIT 30;
```

### Por banco

`pg_database.datfrozenxid` representa o menor marco relevante entre as relações daquele banco. Uma única relação antiga pode impedir o avanço do banco inteiro.

```sql
SELECT
    datname,
    age(datfrozenxid) AS idade_xid,
    mxid_age(datminmxid) AS idade_multixact
FROM pg_database
ORDER BY idade_xid DESC;
```

### Por cluster

A margem do cluster é limitada pelo banco com o `datfrozenxid` mais antigo. Como XIDs são globais ao cluster, carga em um banco também avança o contador observado pelos demais.

Relações frequentemente esquecidas incluem:

- tabelas de catálogos do sistema mantidas por muito tempo; os índices associados podem aumentar o custo do vacuum, mas não possuem `relfrozenxid` próprio;
- tabelas TOAST de relações pouco acessadas;
- materialized views antigas;
- tabelas com parâmetros de autovacuum alterados;
- bancos raramente acessados;
- tabelas de backup, staging ou cópias `_old` mantidas sem necessidade.

> Uma relação pequena pode manter o `datfrozenxid` antigo. O risco não é proporcional apenas ao tamanho da relação; o custo para corrigi-la, sim.

### O que `age()` mede

`age(xid)` retorna a distância, em transações, entre o XID informado e o contador atual, considerando a aritmética interna de XIDs. O valor deve ser monitorado como idade, não como XID absoluto.

---

## 7. Visibility Map: onde está a maior parte da eficiência

Cada relação heap possui um arquivo auxiliar `_vm`. O Visibility Map mantém dois bits por página:

| Bit | Significado operacional |
|---|---|
| `all_visible` | todas as tuplas da página são visíveis para todas as transações atuais e futuras, até a página ser modificada |
| `all_frozen` | todas as tuplas da página estão congeladas; vacuums futuros podem pular a página até ocorrer escrita ou lock que invalide o estado |

`all_frozen` implica `all_visible`.

### O que pode ser inferido — e o que não pode

É útil decompor o heap em três grupos:

| Grupo | Cálculo aproximado | Interpretação |
|---|---:|---|
| páginas `all_frozen` | `all_frozen` | podem ser puladas por vacuum agressivo |
| páginas `all_visible`, mas não `all_frozen` | `all_visible - all_frozen` | candidatas a trabalho de congelamento |
| páginas não `all_visible` | `total - all_visible` | podem conter escrita recente, tuplas mortas ou simplesmente não terem sido marcadas pelo último vacuum |

O terceiro grupo **não é uma estimativa direta de trabalho de índice**. Uma página sem `all_visible` não necessariamente contém tuplas mortas ou `LP_DEAD`. O VM é conservador: bit ausente significa “não há garantia”, não “a condição oposta foi provada”.

### Inspeção

```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT
    c.oid::regclass AS relacao,
    pg_relation_size(c.oid) / current_setting('block_size')::int AS paginas_heap,
    vm.all_visible,
    vm.all_frozen,
    round(
        100.0 * vm.all_frozen /
        nullif(pg_relation_size(c.oid) / current_setting('block_size')::int, 0),
        1
    ) AS pct_all_frozen
FROM pg_class c
CROSS JOIN LATERAL pg_visibility_map_summary(c.oid) AS vm
WHERE c.relkind = 'r'
  AND pg_relation_size(c.oid) > 0
ORDER BY pg_relation_size(c.oid) DESC
LIMIT 30;
```

> **Privilégios.** As funções de inspeção de `pg_visibility` podem ser executadas por superusuários e por membros do papel predefinido `pg_stat_scan_tables`; o papel `pg_monitor` herda esse privilégio. A função `pg_truncate_visibility_map()` é a exceção e permanece restrita a superusuário, porque altera o Visibility Map. Em serviços gerenciados, a disponibilidade da extensão e a possibilidade de conceder esses papéis dependem do provedor. Se essa telemetria não estiver disponível, use a variante sem Visibility Map da consulta de ranking; você perde a estimativa de trabalho pendente, não o acompanhamento de idade.

Uma relação estática com 100% das páginas `all_frozen` tende a impor muito pouco I/O de heap ao próximo vacuum agressivo. Ainda existem custos de abertura da relação, locks, leitura do VM, atualização de metadados e processamento separado da TOAST. Portanto, “quase nenhum I/O de heap” é mais preciso que “custo zero”.

### `INDEX_CLEANUP`

O padrão `AUTO` permite ao PostgreSQL decidir se vale a pena processar os índices. `INDEX_CLEANUP OFF` pode ser útil em uma emergência ou em uma relação comprovadamente imutável, mas não deve ser inferido apenas de `all_visible = total`.

Em tabelas com modificações frequentes, desabilitar continuamente a limpeza de índices pode acumular bloat e ponteiros de linha mortos. Para uso geral, mantenha `AUTO` e altere somente com telemetria e objetivo explícito.

---

## 8. MultiXact: o contador paralelo

Quando várias transações mantêm locks compatíveis sobre a mesma linha, o campo `xmax` não consegue armazenar individualmente todos os participantes. O PostgreSQL cria um **MultiXactId**, que referencia uma lista de membros armazenada em `pg_multixact`.

MultiXactIds também usam contadores finitos e precisam de manutenção. Os marcos paralelos são:

| XID | MultiXact |
|---|---|
| `pg_class.relfrozenxid` | `pg_class.relminmxid` |
| `pg_database.datfrozenxid` | `pg_database.datminmxid` |
| `autovacuum_freeze_max_age` | `autovacuum_multixact_freeze_max_age` |
| `vacuum_freeze_min_age` | `vacuum_multixact_freeze_min_age` |
| `vacuum_freeze_table_age` | `vacuum_multixact_freeze_table_age` |
| `vacuum_failsafe_age` | `vacuum_multixact_failsafe_age` |

No PostgreSQL 18, os defaults principais são:

- `autovacuum_multixact_freeze_max_age = 400000000`;
- `vacuum_multixact_freeze_min_age = 5000000`;
- `vacuum_multixact_freeze_table_age = 150000000`;
- `vacuum_multixact_failsafe_age = 1600000000`.

Monitore os dois contadores:

```sql
SELECT
    datname,
    age(datfrozenxid) AS idade_xid,
    mxid_age(datminmxid) AS idade_multixact
FROM pg_database
ORDER BY greatest(age(datfrozenxid), mxid_age(datminmxid)) DESC;
```

Cargas com muitos locks de linha compartilhados, validações de chaves estrangeiras, filas concorrentes ou sistemas de reserva podem consumir MultiXacts mais rapidamente que cargas OLTP simples. Não assuma que o XID sempre será o primeiro limite.

> **Olhando para frente.** No PostgreSQL 19, o espaço de armazenamento dos membros de MultiXact passa a 64 bits e deixa de esgotar. Isso **não** vale para as versões 14 a 18 descritas nesta seção, e mesmo na 19 o `MultiXactId` continua sendo um contador de 32 bits que exige congelamento. Detalhes na seção 18.

---

