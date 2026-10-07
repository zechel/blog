---
title: "Transaction ID Wraparound no PostgreSQL (2/4): parâmetros, horizontes, retenções e monitoramento"
date: 2026-10-07T08:59:00-03:00
description: "Os parâmetros de freeze e seus limites efetivos, o que impede o VACUUM de avançar e como monitorar e projetar a margem de XIDs."
tags: ["postgresql", "dba", "vacuum", "wraparound"]
categories: ["Infraestrutura"]
toc: true
math: false
comments: true
draft: false
---

> **Parâmetros, horizontes, retenções e monitoramento**
>
> **Navegação da série:** [Parte 1](/posts/postgresql-wraparound-parte-1-fundamentos/) · **Parte 2** · [Parte 3](/posts/postgresql-wraparound-parte-3-laboratorio-incidentes/) · [Parte 4](/posts/postgresql-wraparound-parte-4-runbook-evolucao/)

---

## 9. Parâmetros e limites efetivos

Os parâmetros de freeze não atuam de forma independente. Alguns valores são limitados internamente em função de `autovacuum_freeze_max_age`, e o comportamento efetivo pode ser diferente do valor exibido por `SHOW` ou `pg_settings`.

### Os quatro parâmetros principais

Nas versões PostgreSQL 14 a 18, os valores padrão relacionados a XID são:

| Parâmetro | Default | Contexto | Função |
|---|---:|---|---|
| `vacuum_freeze_min_age` | 50.000.000 | sessão/usuário | idade mínima para que um XID elegível seja congelado |
| `vacuum_freeze_table_age` | 150.000.000 | sessão/usuário | idade que favorece uma estratégia agressiva no próximo `VACUUM` |
| `autovacuum_freeze_max_age` | 200.000.000 | início do servidor | idade que força autovacuum anti-wraparound, mesmo se o autovacuum normal estiver desabilitado |
| `vacuum_failsafe_age` | 1.600.000.000 | sessão/usuário | idade a partir da qual o `VACUUM` prioriza evitar wraparound e reduz trabalhos secundários |

Uma linha do tempo simplificada, usando os defaults, é:

```text
0 ───────► 50M ───────► 150M ───────► 200M ─────────────────► 1,6B ─────► limite
           │             │             │                       │
     XIDs tornam-se   próximo scan   autovacuum             failsafe
     elegíveis para   completo pode  anti-wraparound        reduz trabalho
     congelamento     ser antecipado é forçado              secundário
```

`vacuum_freeze_table_age` não é uma agenda independente. A documentação descreve a decisão do scan agressivo em relação à diferença entre `vacuum_freeze_table_age` e `vacuum_freeze_min_age`, além de outras condições que podem levar o `VACUUM` a percorrer todas as páginas ainda não `all_frozen`.

### Os limites internos

O PostgreSQL aplica limites efetivos:

| Parâmetro | Limite efetivo em relação a `autovacuum_freeze_max_age` |
|---|---|
| `vacuum_freeze_min_age` | no máximo 50% |
| `vacuum_freeze_table_age` | no máximo 95% |
| `vacuum_failsafe_age` | no mínimo 105% |

Esses ajustes são aplicados internamente durante o `VACUUM`. O valor configurado continua aparecendo em `pg_settings`, por isso a diferença pode passar despercebida.

Considere esta configuração:

```text
autovacuum_freeze_max_age = 2.000.000.000
vacuum_failsafe_age        = 1.600.000.000
```

O failsafe efetivo passa a ser, no mínimo:

```text
2.000.000.000 × 1,05 = 2.100.000.000
```

Isso deixa cerca de 47,5 milhões de XIDs até a fronteira matemática de `2^31`, e aproximadamente 44,5 milhões até a salvaguarda que bloqueia novas atribuições quando restam menos de três milhões. Em um ambiente que consome três milhões de XIDs por dia, essa margem representa aproximadamente quinze dias.

O risco não está apenas no número absoluto. Ele depende de:

- taxa de consumo de XIDs;
- duração provável do `VACUUM` das maiores relações;
- número de workers disponíveis;
- vazão do armazenamento;
- geração e arquivamento de WAL;
- existência de horizontes presos;
- porcentagem de páginas já `all_frozen`.

### Aumentar `autovacuum_freeze_max_age`: quando é inadequado e quando pode ser legítimo

Aumentar o parâmetro apenas para silenciar execuções incômodas do autovacuum é um anti-padrão. O trabalho é adiado, a margem do failsafe pode ser comprimida e mais tabelas podem cruzar o limiar em uma janela curta.

Entretanto, a documentação oficial não proíbe aumentos em todos os cenários. Em relações muito grandes e predominantemente estáticas, uma configuração maior pode reduzir a frequência de scans agressivos. O custo adicional inclui mais espaço para `pg_xact` e, quando habilitado, `pg_commit_ts`, além de uma margem operacional menor se a taxa de XIDs crescer ou a manutenção falhar.

Vale ter os números concretos em mente. `pg_xact` armazena dois bits de status por transação:

```text
200.000.000 XIDs  →  ~48 MB
2.000.000.000 XIDs →  ~477 MB
```

Ou seja, subir `autovacuum_freeze_max_age` do default para o teto multiplica o `pg_xact` retido por dez. Isso ainda é pequeno em termos absolutos, e por si só raramente é o argumento decisivo. `pg_commit_ts`, quando `track_commit_timestamp` está ativo, é bem mais pesado: cerca de 10 bytes por transação, o que na mesma comparação vai de aproximadamente 2 GB para 20 GB. Confirme o consumo real no seu ambiente antes de tratar esses valores como orçamento:

```sql
SELECT
    d.diretorio,
    pg_size_pretty(
        sum((pg_stat_file(d.diretorio || '/' || f.name)).size)
    ) AS tamanho
FROM (VALUES
        ('pg_xact'),
        ('pg_commit_ts'),
        ('pg_multixact/offsets'),
        ('pg_multixact/members')
     ) AS d(diretorio)
CROSS JOIN LATERAL pg_ls_dir(d.diretorio) AS f(name)
GROUP BY d.diretorio
ORDER BY d.diretorio;
```

`pg_ls_dir` e `pg_stat_file` são restritas a superusuário por padrão; outros papéis precisam receber `GRANT EXECUTE` nessas funções. Ser membro de `pg_read_server_files` não basta: esse papel só amplia o acesso a arquivos fora do diretório de dados. Com acesso ao sistema de arquivos, `du -sh` nos diretórios correspondentes dentro do `PGDATA` resolve o mesmo. Em serviços gerenciados, use as métricas de armazenamento do provedor.

Portanto, a regra correta é:

> Não aumente `autovacuum_freeze_max_age` como reação automática. Faça isso somente depois de medir taxa de XIDs, duração de vacuum, estado do Visibility Map, retenções, espaço de controle transacional e margem até o failsafe.

### O efeito de `vacuum_freeze_min_age`

Um valor alto evita congelar versões que provavelmente serão modificadas logo, reduzindo escrita e WAL desnecessários. Em contrapartida, depois de um scan que avance `relfrozenxid`, a idade final tende a ficar próxima do `vacuum_freeze_min_age` usado, acrescida dos XIDs consumidos durante a operação.

Isso não significa que “nenhum progresso foi acumulado”. Páginas efetivamente congeladas podem permanecer marcadas como `all_frozen` e ser puladas no futuro. O que fica menor é a distância entre o novo `relfrozenxid` e o XID atual.

### Parâmetros por relação

Alguns parâmetros podem ser definidos por tabela:

```sql
ALTER TABLE historico_2025 SET (
    autovacuum_freeze_min_age             = 0,
    autovacuum_freeze_table_age           = 0,
    autovacuum_multixact_freeze_min_age   = 0,
    autovacuum_multixact_freeze_table_age = 0
);
```

Para uma tabela grande e ativa, também é possível antecipar o autovacuum anti-wraparound:

```sql
ALTER TABLE eventos
SET (autovacuum_freeze_max_age = 100000000);
```

O valor por tabela de `autovacuum_freeze_max_age` pode reduzir o limite global, mas não ampliá-lo.

Liste todas as exceções configuradas:

```sql
SELECT
    n.nspname AS esquema,
    c.relname AS relacao,
    c.reloptions
FROM pg_class AS c
JOIN pg_namespace AS n ON n.oid = c.relnamespace
WHERE c.reloptions IS NOT NULL
ORDER BY 1, 2;
```

### `autovacuum_enabled = false` não desliga a proteção

```sql
ALTER TABLE grande
SET (autovacuum_enabled = false);
```

Esse comando desabilita a manutenção automática normal da relação, mas não impede o autovacuum anti-wraparound. O PostgreSQL ainda pode iniciar um vacuum para proteger XIDs ou MultiXacts antigos.

Na prática, desabilitar autovacuum pode deixar a tabela acumular tuplas mortas, estatísticas defasadas e páginas não `all_visible`. Quando a proteção finalmente for acionada, ela poderá encontrar uma relação maior e mais cara de processar.

---

## 10. O que limita o VACUUM: horizontes e retenções

Executar `VACUUM FREEZE` não garante, por si só, que `relfrozenxid` ou `datfrozenxid` avançarão tanto quanto o operador espera. O `VACUUM` precisa respeitar snapshots e retenções que ainda podem necessitar de versões antigas.

A expressão “horizonte do vacuum” resume o limite mais antigo que ainda precisa ser preservado. Na implementação e nas estatísticas, aparecem conceitos como `OldestXmin`, `backend_xmin`, `xmin` de slots de replicação e `catalog_xmin`.

### 1. Transações longas

Uma transação antiga pode manter um snapshot que exige versões anteriores das linhas. O caso operacional mais comum é `idle in transaction`.

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    age(backend_xid)  AS idade_backend_xid,
    age(backend_xmin) AS idade_backend_xmin,
    now() - xact_start AS duracao_transacao,
    now() - state_change AS tempo_no_estado,
    left(query, 120) AS query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL
   OR backend_xmin IS NOT NULL
ORDER BY greatest(
             coalesce(age(backend_xid), 0),
             coalesce(age(backend_xmin), 0)
         ) DESC;
```

Não termine uma sessão apenas porque ela aparece no topo. Confirme o papel da transação, o impacto de rollback e a política da aplicação.

Uma proteção preventiva comum é:

```sql
ALTER SYSTEM
SET idle_in_transaction_session_timeout = '10min';

SELECT pg_reload_conf();
```

O valor precisa ser compatível com transações legítimas do ambiente.

### 2. Prepared transactions

Uma transação preparada por confirmação em duas fases (2PC) permanece no servidor até receber `COMMIT PREPARED` ou `ROLLBACK PREPARED`. Ela sobrevive a uma reinicialização e não aparece como uma sessão ativa comum.

```sql
SELECT
    gid,
    prepared,
    owner,
    database,
    age(transaction) AS idade_xid
FROM pg_prepared_xacts
ORDER BY prepared;
```

A decisão entre commit e rollback pertence ao protocolo distribuído que criou a transação. Não resolva uma transação preparada órfã sem investigar o coordenador e o estado dos demais participantes.

Quando a plataforma não usa 2PC, manter `max_prepared_transactions = 0` elimina essa classe de retenção.

### 3. Slots de replicação

Slots de replicação possuem retenções diferentes:

- `xmin`: pode preservar versões de linhas;
- `catalog_xmin`: pode preservar linhas de catálogos para decodificação lógica;
- `restart_lsn`: pode preservar arquivos de WAL.

```sql
SELECT
    slot_name,
    slot_type,
    database,
    active,
    active_pid,
    age(xmin) AS idade_xmin,
    age(catalog_xmin) AS idade_catalog_xmin,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,
    safe_wal_size,
    CASE
        WHEN restart_lsn IS NULL THEN NULL
        ELSE pg_size_pretty(
                 pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
             )
    END AS wal_retido
FROM pg_replication_slots
ORDER BY greatest(
             coalesce(age(xmin), 0),
             coalesce(age(catalog_xmin), 0)
         ) DESC;
```

Um slot inativo não é automaticamente abandonado. Pode representar um consumidor temporariamente indisponível. Antes de removê-lo, confirme o contrato de replicação ou CDC e o procedimento de reconstrução do consumidor.

```sql
-- Execute somente depois de confirmar que o slot não será mais usado.
SELECT pg_drop_replication_slot('slot_confirmadamente_obsoleto');
```

`max_slot_wal_keep_size` limita a quantidade de WAL que um slot pode reter em checkpoints. Ele ajuda a proteger o volume `pg_wal`, mas **não libera automaticamente** um `xmin` ou `catalog_xmin` antigo. São problemas relacionados, porém diferentes.

### 4. Feedback de hot standby

Com `hot_standby_feedback = on`, o standby informa ao primário quais versões ainda são necessárias para suas consultas. Isso reduz cancelamentos por conflito de recuperação, mas pode manter versões mortas por mais tempo no primário.

A configuração não é intrinsecamente errada. Ela representa uma troca entre:

- cancelar consultas longas no standby;
- permitir mais bloat e retenção no primário.

Monitore consultas longas nos standbys e o efeito no horizonte do primário.

No primário, o `xmin` informado pelo standby aparece em lugares diferentes conforme a réplica use ou não um slot. Com slot físico, ele fica em `pg_replication_slots.xmin`. Sem slot, ele fica no walsender, em `pg_stat_replication.backend_xmin`, e não aparece em nenhuma consulta de slots:

```sql
SELECT
    pid,
    application_name,
    client_addr,
    state,
    age(backend_xmin) AS idade_xmin_standby
FROM pg_stat_replication
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;
```

### 5. Cursores `WITH HOLD`

Um cursor `WITH HOLD` continua disponível depois do commit, mas seus resultados são materializados em memória ou arquivo temporário no momento da confirmação. Ele não deve ser tratado como um snapshot indefinidamente aberto que, por si só, prende o horizonte do vacuum após o commit.

Isso não elimina outros custos de cursores retidos, como espaço temporário e conexões de aplicação, mas evita um diagnóstico incorreto de wraparound.

### Consulta de pré-voo

Antes de uma campanha manual de freeze, obtenha uma visão consolidada:

```sql
SELECT
    (
        SELECT max(age(backend_xid))
        FROM pg_stat_activity
        WHERE backend_xid IS NOT NULL
    ) AS backend_xid_mais_antigo,
    (
        SELECT max(age(backend_xmin))
        FROM pg_stat_activity
        WHERE backend_xmin IS NOT NULL
    ) AS backend_xmin_mais_antigo,
    (
        SELECT max(age(backend_xmin))
        FROM pg_stat_replication
        WHERE backend_xmin IS NOT NULL
    ) AS standby_xmin_mais_antigo,
    (
        SELECT max(age(xmin))
        FROM pg_replication_slots
        WHERE xmin IS NOT NULL
    ) AS slot_xmin_mais_antigo,
    (
        SELECT max(age(catalog_xmin))
        FROM pg_replication_slots
        WHERE catalog_xmin IS NOT NULL
    ) AS slot_catalog_xmin_mais_antigo,
    (
        SELECT count(*)
        FROM pg_replication_slots
        WHERE NOT active
    ) AS slots_inativos,
    (
        SELECT count(*)
        FROM pg_prepared_xacts
    ) AS prepared_xacts;
```

A consulta não decide o que deve ser encerrado. Ela indica onde investigar antes de gerar I/O e WAL em larga escala.

---

## 11. Monitoramento e planejamento de capacidade

### A consulta essencial por banco

O limite matemático da comparação circular é `2^31 = 2.147.483.648`. O PostgreSQL interrompe novas atribuições antes disso, quando restam menos de aproximadamente três milhões de XIDs.

```sql
WITH limites AS (
    SELECT
        2147483648::numeric AS meia_janela,
        3000000::numeric    AS guarda_seguranca
)
SELECT
    d.datname AS banco,
    age(d.datfrozenxid) AS idade_xid,
    round(
        100 * age(d.datfrozenxid)::numeric / l.meia_janela,
        2
    ) AS pct_da_meia_janela,
    greatest(
        0,
        l.meia_janela - l.guarda_seguranca - age(d.datfrozenxid)
    )::bigint AS xids_ate_bloqueio_estimado,
    mxid_age(d.datminmxid) AS idade_multixact
FROM pg_database AS d
CROSS JOIN limites AS l
ORDER BY age(d.datfrozenxid) DESC;
```

O valor de três milhões é uma salvaguarda documentada, não um SLO de operação. Esperar chegar perto dela é uma falha de planejamento.

### Percentual não basta

Uma idade de 50% pode representar situações muito diferentes:

- consumo de 100 mil XIDs por dia: muitos anos de margem;
- consumo de 100 milhões por dia: poucas semanas;
- consumo crescente por subtransações ou carga nova: projeção anterior inválida.

A métrica operacional mais útil é **tempo restante à taxa observada**, acompanhada do volume de trabalho pendente.

### Medindo a taxa real de consumo

Crie uma tabela de amostragem em um schema administrativo:

```sql
CREATE SCHEMA IF NOT EXISTS observabilidade;

CREATE TABLE IF NOT EXISTS observabilidade.amostra_xid (
    momento timestamptz NOT NULL DEFAULT clock_timestamp(),
    xid8_atual bigint NOT NULL
);
```

Amostre em intervalos regulares:

```sql
INSERT INTO observabilidade.amostra_xid (xid8_atual)
SELECT pg_current_xact_id()::text::bigint;
```

`pg_current_xact_id()` força a atribuição de um XID para a transação da própria coleta. Uma amostra a cada quinze minutos adiciona menos de cem XIDs por dia, custo normalmente irrelevante. O cast intermediário para texto permite armazenar o valor `xid8` como `bigint` para cálculos.

> **Por que não usar `xact_commit + xact_rollback`?** É comum ver essa soma, extraída de `pg_stat_database`, apresentada como medida da taxa de XIDs. Ela mede transações concluídas, não XIDs atribuídos, e pode divergir nas duas direções. Pode superestimar o consumo em cargas com muitas transações somente leitura, que normalmente não recebem XID real, e pode subestimar quando existem muitas subtransações com escrita ou outras alocações de XID que não correspondem diretamente aos contadores de transações da aplicação. Além disso, os valores são separados por banco e podem ser reiniciados, criando descontinuidades na série. Em aplicações PL/pgSQL com tratamento de exceção dentro de loops, a diferença entre as duas medições pode ser significativa.
>
> A série baseada em `pg_current_xact_id()` mede exatamente o que importa: o avanço do contador global do cluster.

Taxa diária:

```sql
WITH amostras AS (
    SELECT
        momento,
        xid8_atual,
        lag(momento) OVER (ORDER BY momento) AS momento_anterior,
        lag(xid8_atual) OVER (ORDER BY momento) AS xid_anterior
    FROM observabilidade.amostra_xid
), intervalos AS (
    SELECT
        momento,
        xid8_atual - xid_anterior AS xids_consumidos,
        extract(epoch FROM momento - momento_anterior) AS segundos
    FROM amostras
    WHERE xid_anterior IS NOT NULL
      AND xid8_atual >= xid_anterior
)
SELECT
    date_trunc('day', momento) AS dia,
    sum(xids_consumidos) AS xids_amostrados,
    round(
        sum(xids_consumidos)::numeric
        / nullif(sum(segundos), 0)
        * 86400
    ) AS xids_por_dia_projetados,
    round(
        max(xids_consumidos::numeric / nullif(segundos, 0))
    ) AS pico_xids_por_segundo
FROM intervalos
GROUP BY 1
ORDER BY 1 DESC;
```

A condição `xid8_atual >= xid_anterior` descarta intervalos que cruzem restauração de backup, troca de cluster ou outra descontinuidade operacional. Como `xid8` inclui epoch, o wraparound normal do XID de 32 bits não reinicia essa série.

### Projetando dias restantes

Substitua a taxa pelo percentil ou pelo pior dia representativo do ambiente, não apenas pela média histórica:

```sql
WITH estado AS (
    SELECT max(age(datfrozenxid))::numeric AS idade
    FROM pg_database
), taxa AS (
    SELECT 3000000::numeric AS xids_por_dia -- substitua pela medição
), limites AS (
    SELECT
        2147483648::numeric AS meia_janela,
        3000000::numeric AS guarda
)
SELECT
    e.idade AS idade_atual,
    round(100 * e.idade / l.meia_janela, 2) AS pct_da_meia_janela,
    floor(
        (l.meia_janela - l.guarda - e.idade)
        / nullif(t.xids_por_dia, 0)
    ) AS dias_ate_bloqueio_estimado,
    current_date
      + floor(
            (l.meia_janela - l.guarda - e.idade)
            / nullif(t.xids_por_dia, 0)
        )::int AS data_estimada
FROM estado AS e
CROSS JOIN taxa AS t
CROSS JOIN limites AS l;
```

Trate a data como projeção, não promessa. Recalcule quando a carga, a arquitetura ou o padrão de subtransações mudar.

### Ranking de relações

```sql
SELECT
    n.nspname AS esquema,
    c.relname AS relacao,
    c.relkind,
    age(c.relfrozenxid) AS idade_xid,
    mxid_age(c.relminmxid) AS idade_multixact,
    pg_size_pretty(pg_relation_size(c.oid)) AS heap,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total,
    s.all_visible,
    s.all_frozen,
    (pg_relation_size(c.oid) / current_setting('block_size')::int)::bigint
        AS paginas_heap,
    round(
        100 * s.all_frozen::numeric
        / nullif(
            pg_relation_size(c.oid)
            / current_setting('block_size')::int,
            0
        ),
        1
    ) AS pct_frozen,
    c.reloptions
FROM pg_class AS c
JOIN pg_namespace AS n ON n.oid = c.relnamespace
LEFT JOIN LATERAL pg_visibility_map_summary(c.oid) AS s ON true
WHERE c.relkind IN ('r', 'm', 't')
  AND pg_relation_size(c.oid) > 0
ORDER BY age(c.relfrozenxid) DESC
LIMIT 50;
```

A extensão `pg_visibility` precisa estar instalada no banco consultado; sem ela, a consulta falha. A função pode gerar leitura adicional; avalie a frequência em clusters com muitas relações.

Sem `pg_visibility`, esta variante mantém o ranking por idade, sem as colunas do Visibility Map:

```sql
SELECT
    n.nspname AS esquema,
    c.relname AS relacao,
    c.relkind,
    age(c.relfrozenxid) AS idade_xid,
    mxid_age(c.relminmxid) AS idade_multixact,
    pg_size_pretty(pg_relation_size(c.oid)) AS heap,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total,
    c.reloptions
FROM pg_class AS c
JOIN pg_namespace AS n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm', 't')
  AND pg_relation_size(c.oid) > 0
ORDER BY age(c.relfrozenxid) DESC
LIMIT 50;
```

### Estimando páginas ainda não `all_frozen`

```sql
SELECT
    pg_size_pretty(
        sum(
            greatest(
                0,
                pg_relation_size(c.oid)
                    / current_setting('block_size')::int
                    - s.all_frozen
            )
            * current_setting('block_size')::int
        )::bigint
    ) AS heap_nao_all_frozen
FROM pg_class AS c
CROSS JOIN LATERAL pg_visibility_map_summary(c.oid) AS s
WHERE c.relkind IN ('r', 'm', 't')
  AND pg_relation_size(c.oid) > 0;
```

Esse número é uma aproximação de páginas que ainda podem exigir inspeção. Ele não é uma previsão exata de tempo, de WAL ou de trabalho de índices.

### Vacuums em andamento

```sql
SELECT
    p.pid,
    p.datname,
    p.relid::regclass AS relacao,
    p.phase,
    p.heap_blks_total,
    p.heap_blks_scanned,
    p.heap_blks_vacuumed,
    round(
        100 * p.heap_blks_scanned::numeric
        / nullif(p.heap_blks_total, 0),
        1
    ) AS pct_percurso_heap,
    p.index_vacuum_count,
    a.wait_event_type,
    a.wait_event,
    clock_timestamp() - a.xact_start AS duracao,
    a.query
FROM pg_stat_progress_vacuum AS p
JOIN pg_stat_activity AS a USING (pid)
ORDER BY duracao DESC;
```

`heap_blks_scanned` representa blocos avançados pelo processo e inclui páginas puladas com auxílio do Visibility Map. Portanto, o percentual indica progresso no percurso lógico da relação, não bytes fisicamente lidos do armazenamento.

### Arquivamento e WAL

```sql
SELECT
    archived_count,
    failed_count,
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;
```

Quando o arquivamento está habilitado, a fila local pode ser observada com:

```sql
SELECT
    count(*) FILTER (WHERE name LIKE '%.ready') AS aguardando_arquivo,
    count(*) FILTER (WHERE name LIKE '%.done')  AS concluidos_visiveis
FROM pg_ls_archive_statusdir();
```

Permissões e disponibilidade da função dependem da versão e do papel utilizado.

### Prometheus e postgres_exporter

O `prometheus-community/postgres_exporter` possui um coletor `database_wraparound`, mas ele é desabilitado por padrão nas versões atuais. Habilite-o explicitamente e verifique os nomes das métricas da versão instalada:

```bash
postgres_exporter --collector.database_wraparound
```

Para estimar taxa e dias restantes, normalmente é necessário expor também um contador monotônico baseado em `pg_current_xact_id()` por meio de uma consulta customizada ou de um exporter próprio. Não copie expressões PromQL sem confirmar:

- nome e tipo da métrica;
- frequência de scrape;
- comportamento após reinicialização ou failover;
- labels por cluster e banco;
- permissões do usuário de monitoramento.

### SLOs e alertas

Não existe um único percentual universal. Uma política deve relacionar idade, taxa e tempo de execução esperado. Um exemplo conservador:

| Condição | Ação sugerida |
|---|---|
| mais de 180 dias projetados e autovacuum saudável | acompanhamento normal |
| 90–180 dias | revisar configuração, VM e taxa |
| 30–90 dias | abrir plano de redução de idade e validar capacidade |
| 7–30 dias | tratar como risco operacional alto, com execução acompanhada |
| menos de 7 dias ou failsafe próximo | tratar como incidente |

Esses intervalos são uma política inicial, não defaults do PostgreSQL. Ambientes com tabelas de dezenas de terabytes podem precisar de margens maiores porque o tempo físico para completar o trabalho é maior.

---
