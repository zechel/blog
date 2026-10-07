---
title: "Transaction ID Wraparound no PostgreSQL (4/4): otimizações, runbook, evolução por versão e checklist"
date: 2026-10-07T09:01:00-03:00
description: "Otimizações, runbook de emergência, a evolução do PostgreSQL 9.5 ao 19 e um checklist operacional para não chegar perto do limite."
tags: ["postgresql", "dba", "vacuum", "wraparound"]
categories: ["Infraestrutura"]
toc: true
math: false
comments: true
draft: false
---

> **Otimizações, runbook, evolução por versão e checklist**
>
> **Navegação da série:** [Parte 1](/posts/postgresql-wraparound-parte-1-fundamentos/) · [Parte 2](/posts/postgresql-wraparound-parte-2-parametros-monitoramento/) · [Parte 3](/posts/postgresql-wraparound-parte-3-laboratorio-incidentes/) · **Parte 4**

---

## 15. Otimizações e melhorias

### Separe relações quentes e frias

| Característica | Relações frias | Relações quentes |
|---|---|---|
| Exemplos | partições fechadas, histórico imutável | pedidos, filas, sessões, estoque |
| Objetivo | atingir alto `all_frozen` e permanecer estável | manter tuplas mortas, VM e idade sob controle continuamente |
| Ferramenta principal | freeze planejado ou carga já congelada | autovacuum bem dimensionado |
| `INDEX_CLEANUP` | pode ser dispensável após validação | normalmente `AUTO` |
| Frequência | pontual ou rara | contínua |

Uma página deixa de ser `all_frozen` quando recebe inserção, atualização, remoção ou lock de linha. Por isso, a classificação deve ser baseada em telemetria, não apenas no nome da tabela.

### Identifique frio por contadores

Colete snapshots de `pg_stat_all_tables`:

```sql
SELECT
    schemaname,
    relname,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_tup_hot_upd,
    last_vacuum,
    last_autovacuum,
    clock_timestamp() AS coletado_em
FROM pg_stat_all_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema');
```

Os contadores podem ser reiniciados por `pg_stat_reset`, reinicialização, promoção ou recriação da relação. Guarde também `stats_reset` e identidade do cluster.

### Use `wal_compression` com medição

Freeze pode modificar muitas páginas e gerar full-page images. `wal_compression` pode reduzir esse volume, principalmente em páginas compressíveis.

```sql
SHOW wal_compression;
```

Algoritmos disponíveis dependem da compilação e da versão. O custo é CPU; o benefício precisa ser medido no primário, no arquivamento e nas réplicas.

### Trabalhe com orçamento por lote

Uma campanha manual deve definir:

- bytes máximos por janela;
- horário limite para iniciar nova relação;
- número máximo de vacuums simultâneos;
- teto de lag das réplicas;
- teto da fila de arquivamento;
- latência máxima da aplicação;
- critério de pausa;
- verificação de avanço após cada lote.

Começar pelas relações menores pode reduzir rapidamente o número de objetos antigos. Começar pela maior relação pode ser melhor quando ela é o verdadeiro gargalo de tempo. A ordem depende da margem e do objetivo, não de uma regra única.

### Calibre o throttle de `VACUUM` manual

`VACUUM` manual usa os parâmetros de custo da sessão. Um exemplo inicial:

```sql
SET vacuum_cost_delay = '2ms';
SET vacuum_cost_limit = 600;
SET maintenance_work_mem = '2GB';
SET lock_timeout = '2min';

VACUUM (VERBOSE, TRUNCATE OFF) esquema.relacao;
```

Não associe `600` a uma vazão fixa. Ajuste com base em observação.

`TRUNCATE OFF` evita a tentativa de truncar páginas vazias no fim da relação, operação que pode precisar de `ACCESS EXCLUSIVE` por um intervalo curto e gerar conflitos em réplicas.

No PostgreSQL 16 ou posterior:

```sql
VACUUM (
    VERBOSE,
    TRUNCATE OFF,
    BUFFER_USAGE_LIMIT '256MB'
) esquema.relacao;
```

Um buffer maior pode acelerar a operação, mas também aumentar a expulsão de páginas úteis de `shared_buffers`. O valor global `vacuum_buffer_usage_limit` e o limite de um oitavo de `shared_buffers` devem ser considerados.

### Particionamento declarativo

Particionamento não elimina wraparound. Cada partição física tem seus próprios marcos e precisa de vacuum. O benefício é operacional:

- granularidade de tratamento;
- partições antigas naturalmente frias;
- remoção rápida de dados vencidos com `DROP` ou `DETACH`;
- trabalho distribuído ao longo do ciclo de vida.

`DROP TABLE` de uma partição pode remover milhões de linhas sem processá-las individualmente, mas exige lock `ACCESS EXCLUSIVE` na partição e na tabela particionada. Planeje o lock e as dependências.

Desde o PostgreSQL 14 existe uma alternativa com lock mais leve:

```sql
ALTER TABLE eventos
DETACH PARTITION eventos_2024_01 CONCURRENTLY;
```

O comando desanexa a partição sem manter `ACCESS EXCLUSIVE` na tabela pai durante toda a operação. Em troca, ele não pode ser executado dentro de um bloco de transação, exige uma segunda fase que pode ficar pendente se a sessão for interrompida — resolvida com `FINALIZE` — e espera transações concorrentes terminarem. Depois de desanexada, a relação continua existindo com seus próprios marcos e ainda precisa ser vacuumada ou removida.

### `COPY ... FREEZE`

Em uma carga inicial, o PostgreSQL pode inserir linhas já congeladas:

```sql
BEGIN;

CREATE TABLE carga_inicial (
    id bigint,
    payload text
);

COPY carga_inicial
FROM '/dados/carga.csv'
WITH (FORMAT csv, FREEZE);

COMMIT;
```

Restrições importantes:

- a tabela precisa ter sido criada ou truncada na subtransação atual;
- não pode haver cursores abertos;
- a transação não pode manter snapshots mais antigos;
- não funciona em tabela particionada;
- no PostgreSQL 18, é rejeitado para foreign tables;
- somente `COPY FROM` aceita a opção;
- depois do commit, as linhas ficam visíveis até para snapshots obtidos antes da carga, como os de transações `REPEATABLE READ` já em andamento, o que viola a regra usual de visibilidade MVCC.

Use apenas quando essa visibilidade antecipada for aceitável e o processo de carga estiver desenhado para a restrição.

### Vacuums disparados por inserção

Desde o PostgreSQL 13, tabelas append-only podem disparar vacuum com:

```text
autovacuum_vacuum_insert_threshold
autovacuum_vacuum_insert_scale_factor
```

Exemplo por tabela:

```sql
ALTER TABLE eventos SET (
    autovacuum_vacuum_insert_threshold = 10000,
    autovacuum_vacuum_insert_scale_factor = 0.02
);
```

Valores menores aumentam frequência e custo recorrente, mas ajudam a atualizar o VM e distribuir o freeze. Meça o efeito antes de aplicar em milhares de partições.

### Dimensione workers e memória em conjunto

Mais workers aumentam paralelismo entre relações, mas também podem multiplicar:

- memória por worker;
- requisições de I/O;
- WAL;
- competição por buffers;
- conexões internas.

No PostgreSQL 18, `autovacuum_worker_slots` reserva o limite de processos no início do servidor, enquanto `autovacuum_max_workers` determina quantos podem trabalhar simultaneamente dentro dessa reserva. Confirme os contextos e limites diretamente em `pg_settings` da versão instalada:

```sql
SELECT
    name,
    setting,
    unit,
    context,
    pending_restart
FROM pg_settings
WHERE name IN (
    'autovacuum_worker_slots',
    'autovacuum_max_workers',
    'autovacuum_work_mem',
    'maintenance_work_mem'
)
ORDER BY name;
```

### Memória para dead tuples

`maintenance_work_mem` e `autovacuum_work_mem` influenciam a quantidade de identificadores de tuplas mortas que o vacuum consegue manter antes de processar índices.

No PostgreSQL 17, a estrutura de memória do vacuum foi reimplementada e deixou de ter o antigo teto efetivo de 1 GB para esse conjunto. Isso reduz a necessidade de vários ciclos de limpeza de índices em tabelas grandes, mas não torna memória ilimitada segura.

Monitore:

```sql
SELECT
    pid,
    relid::regclass,
    phase,
    index_vacuum_count
FROM pg_stat_progress_vacuum;
```

Mais de um ciclo de índice pode ser legítimo em tabelas muito alteradas. Use o dado junto com logs `VERBOSE`, número de tuplas mortas e memória configurada.

### Eager freezing no PostgreSQL 18

O PostgreSQL 18 permite que vacuums normais inspecionem algumas páginas `all_visible` ainda não `all_frozen` e tentem congelá-las antecipadamente. Isso distribui parte do trabalho que, de outra forma, ficaria concentrado no próximo vacuum agressivo.

```sql
SHOW vacuum_max_eager_freeze_failure_rate;
```

Por relação:

```sql
ALTER TABLE historico
SET (vacuum_max_eager_freeze_failure_rate = 0.10);
```

Aumentar o valor não significa “congelar tudo”. Apenas falhas de congelamento contam contra o limite configurado, enquanto congelamentos bem-sucedidos possuem um teto interno de 20% das páginas `all_visible`, mas ainda não `all_frozen`, da relação. O objetivo é amortizar o trabalho entre vacuums normais. Antes de alterar, observe logs, páginas congeladas e custo adicional.

---

## 16. Runbook de emergência

Este runbook pressupõe PostgreSQL 14 a 18. Confirme a versão e o minor release antes de agir.

### Situação A — Idade alta, mas escritas ainda funcionam

#### 1. Registre o estado

```sql
SELECT version();

SELECT
    datname,
    age(datfrozenxid) AS idade_xid,
    mxid_age(datminmxid) AS idade_multixact
FROM pg_database
ORDER BY idade_xid DESC;
```

Registre também:

- taxa de XIDs por dia e por segundo;
- horário e tamanho dos últimos checkpoints;
- estado do arquivamento;
- lag das réplicas;
- espaço em `pg_wal` e no destino de archive;
- vacuums ativos e progresso;
- maiores relações antigas.

#### 2. Investigue retenções

```sql
SELECT
    pid,
    usename,
    state,
    age(backend_xid) AS idade_xid,
    age(backend_xmin) AS idade_xmin,
    clock_timestamp() - xact_start AS duracao,
    left(query, 120) AS query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL
   OR backend_xmin IS NOT NULL
ORDER BY greatest(
             coalesce(age(backend_xid), 0),
             coalesce(age(backend_xmin), 0)
         ) DESC;

SELECT
    gid,
    prepared,
    owner,
    database,
    age(transaction) AS idade
FROM pg_prepared_xacts
ORDER BY idade DESC;

SELECT
    slot_name,
    slot_type,
    active,
    age(xmin) AS idade_xmin,
    age(catalog_xmin) AS idade_catalog_xmin,
    restart_lsn,
    wal_status,
    safe_wal_size
FROM pg_replication_slots
ORDER BY greatest(
             coalesce(age(xmin), 0),
             coalesce(age(catalog_xmin), 0)
         ) DESC;
```

Resolva somente o que estiver confirmado como bloqueador e seguro para encerrar.

#### 3. Identifique as relações mais antigas

```sql
SELECT
    n.nspname AS esquema,
    c.relname AS relacao,
    c.relkind,
    age(c.relfrozenxid) AS idade_xid,
    mxid_age(c.relminmxid) AS idade_multixact,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS tamanho_total
FROM pg_class AS c
JOIN pg_namespace AS n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 50;
```

#### 4. Execute o menor trabalho suficiente

Quando existe margem, vacuume relações individualmente para controlar impacto:

```sql
SET lock_timeout = '2min';
SET vacuum_cost_delay = '2ms';
SET vacuum_cost_limit = 600;

VACUUM (VERBOSE, TRUNCATE OFF) esquema.relacao;
```

Em urgência maior, ajuste ou remova a limitação por custo somente depois de verificar capacidade do armazenamento, WAL e réplicas.

#### 5. Confirme o resultado

```sql
SELECT
    datname,
    age(datfrozenxid) AS idade_xid,
    mxid_age(datminmxid) AS idade_multixact
FROM pg_database
ORDER BY idade_xid DESC;
```

Um vacuum concluído em uma relação não garante que `datfrozenxid` cairá: outra relação, inclusive TOAST ou catálogo, pode continuar sendo o mínimo.

### Situação B — O banco já recusa operações com novos XIDs

> **Antes de qualquer coisa: ignore o hint do log se ele mandar usar modo mono-usuário.** Em PostgreSQL 14, 15 e 16 a mensagem ainda diz `Stop the postmaster and vacuum that database in single-user mode`. A recomendação tornou-se obsoleta com o *Lazy XID* do PostgreSQL 8.3, lançado em fevereiro de 2008, e a mensagem só foi corrigida a partir da versão 17. Detalhes na [seção 13, Fase 5](/posts/postgresql-wraparound-parte-3-laboratorio-incidentes/#fase-5--bloqueio-de-novas-atribuições).

#### O que não fazer

- não reinicie o servidor apenas para “limpar” o estado;
- não pare o postmaster como procedimento padrão;
- não entre em modo mono-usuário sem necessidade específica;
- não use `VACUUM FULL`;
- não comece com `VACUUM FREEZE`;
- não remova slots ou transações preparadas sem validar o impacto.

#### Procedimento

1. **Comunique a indisponibilidade de escrita.** Consultas simples ainda responderem não significa que a aplicação esteja operacional.
2. **Resolva transações preparadas antigas.** Use o coordenador distribuído ou uma decisão formal de recuperação.
3. **Finalize transações longas.** Commit, rollback ou `pg_terminate_backend` conforme impacto conhecido.
4. **Remova somente slots de replicação comprovadamente obsoletos.** Uma réplica associada pode precisar ser reconstruída.
5. **Execute `VACUUM` normal como superusuário no banco afetado.**

```sql
VACUUM (VERBOSE);
```

O comando sem lista de tabelas processa todas as relações para as quais o papel tem permissão. O uso de superusuário é importante para incluir catálogos compartilhados que podem impedir o avanço de `datfrozenxid`.

Se uma relação muito grande atrasar a recuperação e outras relações menores forem mais antigas, é possível trabalhar por tabela:

```sql
VACUUM (VERBOSE, TRUNCATE OFF) esquema.relacao;
```

6. **Conecte-se aos demais bancos críticos e repita a verificação.** O limite do cluster depende do banco com `datfrozenxid` mais antigo.
7. **Confirme que novas escritas voltaram e que a idade realmente caiu.**
8. **Corrija autovacuum, capacidade e retenções antes de encerrar o incidente.**

`template0` normalmente não aceita conexões, e isso costuma gerar a suposição errada de que ele nunca é mantido. O launcher do autovacuum percorre todas as entradas de `pg_database` e os workers conseguem se conectar independentemente de `datallowconn`. Na prática, `template0` é congelado pelo autovacuum como qualquer outro banco.

Portanto, se `template0` aparecer como o banco limitante, a pergunta certa é por que o autovacuum não o processou — worker indisponível, `autovacuum = off`, retenção global ou parâmetro alterado —, e não como abrir conexões nele. Se ele aparecer como o banco limitante e não for corrigido pelo autovacuum, não altere `datallowconn` de forma improvisada durante o incidente. Use um procedimento específico para a versão, com backup válido, janela controlada e revisão das causas que deixaram o template envelhecer.

### Situação C — Anti-wraparound saturando I/O

Primeiro determine se o processo está em failsafe. Nesse modo, o cost delay pode ser ignorado.

```sql
SELECT
    a.pid,
    a.datname,
    p.relid::regclass AS relacao,
    p.phase,
    p.heap_blks_scanned,
    p.heap_blks_total,
    a.wait_event_type,
    a.wait_event,
    a.query
FROM pg_stat_activity AS a
JOIN pg_stat_progress_vacuum AS p USING (pid)
WHERE a.query LIKE '%to prevent wraparound%';
```

> **No PostgreSQL 19**, o filtro por texto pode ser substituído pelas colunas `started_by` e `mode` de `pg_stat_progress_vacuum`, que informam diretamente quem iniciou o vacuum e qual sua agressividade. A consulta acima continua sendo o caminho nas versões 14 a 18.

Se ainda houver margem suficiente e o vacuum responder ao cost delay:

```sql
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '10ms';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 200;
SELECT pg_reload_conf();
```

Esses valores são exemplos de contenção, não recomendação universal. Recalcule o tempo de conclusão com a nova velocidade. Desacelerar tanto que o processo não conclua antes da salvaguarda é pior que a saturação temporária.

### Situação D — MultiXact próximo do limite

Repita a lógica usando:

- `mxid_age(datminmxid)`;
- `mxid_age(relminmxid)`;
- `autovacuum_multixact_freeze_max_age`;
- `vacuum_multixact_failsafe_age`.

Identifique workloads com muitos locks de linha e confirme se vacuums agressivos estão avançando `relminmxid`. Não trate um incidente de MultiXact apenas olhando `datfrozenxid`.

> **No PostgreSQL 19**, `pg_get_multixact_stats()` complementa — e não substitui — `mxid_age()`. A função retorna `num_mxids`, `num_members`, `members_size` e `oldest_multixact`, sendo especialmente útil para observar padrões de alocação de membros e o espaço consumido em `pg_multixact/members`. O alargamento do `MultiXactOffset` para 64 bits elimina o wraparound do contador de membros, mas o `MultiXactId` continua com 32 bits; por isso, `mxid_age(datminmxid)` e `mxid_age(relminmxid)` continuam sendo as métricas de envelhecimento e wraparound. A função fornece um retrato instantâneo, portanto taxas exigem amostras periódicas. Na documentação das versões beta, todas as colunas retornam `NULL` por padrão sem os privilégios de `pg_read_all_stats`; confirme esse comportamento novamente na versão final. Nas versões 14 a 18, a função não existe e a exaustão do espaço de membros continua sendo um risco real.

### Quando o modo mono-usuário pode aparecer

A documentação atual reserva o modo mono-usuário para exceções, especialmente quando se pretende executar `DROP` ou `TRUNCATE` de relações dispensáveis em vez de vacuumá-las. Esse modo remove salvaguardas de wraparound e exige parada completa. Use somente com:

- decisão formal de incidente;
- backup e recuperação verificados;
- lista exata de comandos;
- validação por profissional experiente na versão utilizada;
- compreensão de que novos XIDs podem piorar o estado.

### Serviços gerenciados: o que muda

Boa parte dos ambientes brasileiros de porte médio roda em RDS, Aurora, Cloud SQL ou Azure Database for PostgreSQL. O diagnóstico descrito neste guia continua válido; a caixa de ferramentas, não.

O que **não** está disponível para o administrador do cliente:

- superusuário real — você recebe um papel administrativo (`rds_superuser`, `cloudsqlsuperuser`, `azure_pg_admin`) com privilégios elevados, mas não irrestritos;
- acesso ao sistema de arquivos, e portanto `pg_resetwal`, inspeção direta de `pg_xact` e `du` nos diretórios de controle;
- modo mono-usuário, que exige controle do processo do servidor.

A disponibilidade de extensões de inspeção varia por provedor, versão e política da instância. Azure Database for PostgreSQL Flexible Server e Cloud SQL documentam suporte a `pageinspect` e `pg_visibility`; RDS e Aurora mantêm listas específicas por versão e podem restringir a instalação por parâmetros do serviço. A presença da extensão também não garante privilégios idênticos aos do PostgreSQL comunitário. Consulte a lista oficial do provedor e teste o papel administrativo ou de monitoramento antes de depender dessas funções em um runbook.

O que continua funcionando e deve ser o eixo do trabalho:

- `age(datfrozenxid)`, `mxid_age(datminmxid)` e o ranking por relação;
- `VACUUM` manual e por tabela, incluindo `VERBOSE`, `TRUNCATE OFF` e `BUFFER_USAGE_LIMIT`;
- `pg_stat_progress_vacuum`, `pg_stat_activity`, `pg_replication_slots`, `pg_prepared_xacts`;
- amostragem de `pg_current_xact_id()` para medir taxa;
- parâmetros por relação via `ALTER TABLE ... SET (...)`.

Pontos de atenção específicos:

- **Parâmetros globais passam pelo provedor.** `ALTER SYSTEM` costuma estar bloqueado; use grupos de parâmetros, flags ou o painel equivalente. Alguns parâmetros exigem reinicialização agendada, o que é relevante quando a margem é curta.
- **O provedor pode ter defaults próprios.** Não presuma que `autovacuum_freeze_max_age` está em 200 milhões. Confirme com `SHOW`, e confirme também se a plataforma aplica ajustes automáticos em função do tamanho da instância.
- **Bancos e extensões internos do provedor** aparecem em `pg_database` e no ranking de relações. Eles são mantidos pelo provedor, mas podem sim ser o banco limitante do cluster. Se isso acontecer, abra chamado em vez de improvisar.
- **Slots de replicação órfãos** são uma causa recorrente em ambientes com CDC gerenciado, réplicas de leitura e ferramentas de migração como DMS ou Datastream. Uma tarefa abandonada pode deixar um slot retendo `xmin`, `catalog_xmin` ou WAL muito além do esperado.
- **Aurora PostgreSQL** tem características próprias de armazenamento e de manutenção. Trate a documentação específica do serviço como normativa, não a documentação do PostgreSQL comunitário.
- **`hot_standby_feedback`** pode estar habilitado ou desabilitado conforme o provedor, a versão e o grupo de parâmetros. Confirme diretamente na réplica com `SHOW hot_standby_feedback`; quando habilitado, ele pode produzir o efeito de retenção descrito na [seção 10](/posts/postgresql-wraparound-parte-2-parametros-monitoramento/#10-o-que-limita-o-vacuum-horizontes-e-retenções).

A consequência prática é que, em serviços gerenciados, prevenção vale ainda mais: as opções de resposta em incidente são estruturalmente menores.

---
## 17. O fantasma do PostgreSQL 9.5

Profissionais que operaram versões antigas costumam associar wraparound a scans intermináveis e baixa previsibilidade. Essa memória tem fundamento, mas precisa ser separada do comportamento das versões atuais.

### Antes do PostgreSQL 9.6

O Visibility Map possuía o bit `all_visible`, mas não o bit `all_frozen`. Um vacuum agressivo não tinha uma marca persistente que lhe permitisse saber que determinada página já estava completamente congelada.

A consequência era que páginas estáticas precisavam ser revisitadas por vacuums agressivos futuros. O trabalho anterior não produzia a mesma redução permanente de scans que passou a existir com `all_frozen`.

### O divisor de águas do PostgreSQL 9.6

O PostgreSQL 9.6 adicionou:

- o bit `all_frozen` no Visibility Map;
- `pg_stat_progress_vacuum`;
- capacidade de um vacuum agressivo pular páginas já completamente congeladas.

Para relações frias, isso alterou a natureza do custo. Depois que as páginas fossem verificadas e marcadas como `all_frozen`, vacuums agressivos posteriores poderiam ignorá-las enquanto continuassem sem modificações.

Após uma atualização de versão, o servidor não pode inferir retroativamente todos os bits `all_frozen` sem inspecionar as páginas. Por isso, o primeiro ciclo capaz de preencher o mapa ainda pode ter custo alto. O benefício aparece nos ciclos seguintes.

### O throttle antigo

Até o PostgreSQL 11, o default de `autovacuum_vacuum_cost_delay` era `20ms`. No PostgreSQL 12, ele mudou para `2ms`.

Isso representa dez vezes menos tempo de espera para o mesmo ciclo de custo, mas não equivale automaticamente a dez vezes mais MB/s. O resultado real depende de cache, pesos de custo, armazenamento e concorrência.

Uma tabela de 1 TB lida a 10 MB/s levaria aproximadamente 29 horas, mas esse é um exemplo aritmético baseado em vazão observada ou imposta — não uma velocidade universal do PostgreSQL 9.5.

### Menos observabilidade e menos opções

Versões antigas também não possuíam vários recursos usados hoje:

- `pg_stat_progress_vacuum` antes do 9.6;
- `INDEX_CLEANUP`, `SKIP_LOCKED` e `TRUNCATE` antes do 12;
- `PARALLEL` no `VACUUM` antes do 13;
- `vacuum_failsafe_age` antes do 14;
- `BUFFER_USAGE_LIMIT` antes do 16;
- nova gestão de memória do vacuum antes do 17;
- eager freezing antes do 18.

A ausência desses recursos não significava que todo vacuum falharia, mas deixava menos alternativas para observar, limitar e distribuir o custo.

### Tabelas append-only

Antes do PostgreSQL 13, inserções isoladamente não participavam do gatilho moderno de vacuum por inserção. Uma tabela que recebia apenas `INSERT` podia passar longos períodos sem vacuum normal disparado por tuplas mortas, até que a idade exigisse uma execução agressiva.

`autovacuum_vacuum_insert_threshold` e `autovacuum_vacuum_insert_scale_factor` reduziram esse problema ao permitir que inserções acionem vacuum.

### Freeze antes do PostgreSQL 9.4

Até o PostgreSQL 9.3, congelar uma tupla podia substituir fisicamente seu `xmin` por `FrozenTransactionId`. Desde o 9.4, o XID original é preservado e o estado congelado é representado por bits no cabeçalho.

Essa mudança melhorou a capacidade de diagnóstico e preservou informação histórica da tupla, sem alterar o objetivo de proteção contra wraparound.

### Como interpretar o trauma

A conclusão útil não é “as versões atuais resolveram tudo”. É:

- a janela de 32 bits continua existindo;
- retenções ainda podem impedir progresso;
- configuração e capacidade ainda importam;
- o PostgreSQL moderno consegue evitar muito trabalho repetido que versões antigas precisavam refazer;
- procedimentos antigos, especialmente o uso automático de modo mono-usuário, não devem ser copiados sem conferir a documentação da versão atual.

No momento desta publicação, as versões oficialmente suportadas são 14, 15, 16, 17 e 18. PostgreSQL 14 chega ao fim de suporte em 12 de novembro de 2026. Versões anteriores à 14 não recebem correções da comunidade e devem ter migração planejada independentemente de wraparound.

---

## 18. O que mudou em cada versão

| Versão | Mudança relevante | Efeito operacional |
|---|---|---|
| **até 9.3** | freeze podia sobrescrever `xmin` com `FrozenTransactionId` | informação original do XID de inserção não era preservada da mesma forma |
| **9.4** | representação `HEAP_XMIN_FROZEN` em bits do cabeçalho | preserva o `xmin` original |
| **9.6** | bit `all_frozen` e `pg_stat_progress_vacuum` | páginas congeladas passam a ser puláveis por vacuum agressivo; progresso observável |
| **12** | `INDEX_CLEANUP`, `SKIP_LOCKED`, `TRUNCATE`; delay default de 20 ms para 2 ms | mais controle e menor throttle padrão |
| **13** | gatilhos de vacuum baseados em inserções; vacuum paralelo de índices | melhora tabelas append-only e processamento de índices |
| **14** | `vacuum_failsafe_age` e `vacuum_multixact_failsafe_age` | estratégia explícita de último recurso |
| **15** | logging padrão de autovacuum lento em 10 minutos; mais detalhes em `VACUUM VERBOSE`; compressão WAL ampliada | melhor observabilidade e opções de redução de WAL |
| **16** | *page-level freezing*: o vacuum passa a decidir por página inteira quando congelar compensa; `BUFFER_USAGE_LIMIT` | congela mais cedo e de forma mais barata em páginas que já seriam modificadas, e permite controlar uso de buffers |
| **17** | nova gestão de memória do vacuum; remoção do teto silencioso de 1 GB; WAL de vacuum mais compacto | menos ciclos de índice e menor consumo de memória em muitos workloads |
| **18** | eager freezing, métricas adicionais de tempo e WAL em vacuum e introdução de `autovacuum_worker_slots` | distribui freeze ao longo do tempo, amplia a observabilidade e separa a reserva de processos do número de workers ativos |

### PostgreSQL 18 e eager freezing

Um vacuum normal pode visitar páginas que seriam puladas por estarem `all_visible`, mas ainda não `all_frozen`. Quando o congelamento é bem-sucedido, o custo de um vacuum agressivo futuro diminui.

O parâmetro principal é:

```sql
SHOW vacuum_max_eager_freeze_failure_rate;
```

O default do PostgreSQL 18 é `0.03`, ou 3%. O algoritmo acompanha tentativas malsucedidas e limita quanto trabalho antecipado será feito. Valor zero desabilita eager scanning.

Eager freezing reduz concentração de trabalho, mas não elimina:

- vacuums agressivos periódicos;
- necessidade de monitorar idade;
- retenções por snapshots e slots;
- risco de uma configuração incompatível com a carga.

### O que vem no PostgreSQL 19

> **Conteúdo de versão beta.** O PostgreSQL 19 chegou ao beta 4 em 24 de setembro de 2026, com disponibilidade geral prevista para outubro de 2026. O *feature freeze* já ocorreu, mas defaults e detalhes ainda podem mudar até a GA. Nada desta subseção deve ser usado para decisões de produção; ela existe para que o planejamento de médio prazo não seja feito com um mapa desatualizado. Confirme tudo nas notas de versão finais.

Tudo o que segue descreve funcionalidades novas ou mudanças de comportamento exclusivas do PostgreSQL 19. Alguns objetos citados — como `VACUUM`, `ANALYZE`, `EXPLAIN`, `pg_stat_wal` e `log_autovacuum_min_duration` — já existem nas versões 14 a 18, mas sem as capacidades adicionais descritas nesta subseção.

**O contador de posições dos membros de MultiXact passa a 64 bits.** Até a versão 18, o `MultiXactOffset` que endereça as entradas em `pg_multixact/members` era de 32 bits, criando um limite de aproximadamente quatro bilhões de membros acumulados. Cargas com muitos locks compartilhados de linha e verificações de chave estrangeira podiam consumir membros muito mais rapidamente que novos MultiXactIds. O PostgreSQL 19 elimina o wraparound desse contador e o teto de `2^32` membros. Isso **não** transforma o `MultiXactId` em 64 bits: ele continua sendo um contador de 32 bits, e `relminmxid`, `datminmxid`, vacuum e envelhecimento de MultiXacts continuam necessários. A [seção 8](/posts/postgresql-wraparound-parte-1-fundamentos/#8-multixact-o-contador-paralelo) permanece válida; o que desaparece é a classe específica de exaustão do espaço de membros.

**O aviso antecipado muda de 40 para 100 milhões.** O limiar em que o servidor começa a emitir avisos de proximidade de wraparound para XIDs e MultiXacts sobe de aproximadamente 40 milhões para aproximadamente 100 milhões restantes. Na prática, isso amplia a janela entre o primeiro aviso e a salvaguarda final. Se você usa esse warning como gatilho de alerta ou como referência na sua política de SLO, revise os números descritos na [seção 11](/posts/postgresql-wraparound-parte-2-parametros-monitoramento/#11-monitoramento-e-planejamento-de-capacidade) ao migrar.

**Autovacuum ganha priorização e paralelismo limitado.** O novo sistema de pontuação do PostgreSQL 19 ordena as relações elegíveis a partir de cinco pesos configuráveis: `autovacuum_freeze_score_weight`, `autovacuum_multixact_freeze_score_weight`, `autovacuum_vacuum_score_weight`, `autovacuum_vacuum_insert_score_weight` e `autovacuum_analyze_score_weight`. A visão `pg_stat_autovacuum_scores` permite observar os componentes da pontuação por tabela, embora represente o estado no momento da consulta e não seja uma garantia exata da próxima decisão do launcher. Esse mecanismo atua diretamente no cenário da [Fase 3](/posts/postgresql-wraparound-parte-3-laboratorio-incidentes/#fase-3--vacuums-agressivos-concorrendo-por-recursos), no qual existem mais relações elegíveis que workers disponíveis.

Separadamente, o PostgreSQL 19 introduz `autovacuum_max_parallel_workers`, desabilitado por padrão, e o parâmetro por tabela `autovacuum_parallel_workers`. Eles permitem que **um único autovacuum** use workers paralelos nas fases de vacuum e cleanup dos índices. Não paralelizam o percurso do heap e não aumentam, por si só, o número de relações processadas simultaneamente. O ganho depende de haver pelo menos dois índices elegíveis, do tamanho desses índices e dos limites globais de workers paralelos.

**Scans comuns podem marcar páginas como `all_visible`.** Até a 18, apenas `VACUUM` e `COPY ... FREEZE` atualizavam esse bit do Visibility Map. Na 19, varreduras de tabela também podem marcá-lo, o que tende a reduzir o trabalho pendente encontrado por vacuums futuros em tabelas lidas com frequência.

**Novas colunas de diagnóstico em `pg_stat_progress_vacuum`.** O PostgreSQL 19 acrescenta `started_by`, que informa quem iniciou o vacuum, e `mode`, que informa sua agressividade. Isso substitui, na 19, a heurística de identificar vacuums anti-wraparound pelo texto `(to prevent wraparound)` em `pg_stat_activity`, disponível na consulta da [seção 11](/posts/postgresql-wraparound-parte-2-parametros-monitoramento/#11-monitoramento-e-planejamento-de-capacidade) e usada como filtro na [Situação C](#situação-c--anti-wraparound-saturando-io). Nas versões 14 a 18 essas colunas não existem e a heurística continua sendo o caminho disponível.

**`pg_get_multixact_stats()`.** O PostgreSQL 19 adiciona a função `pg_get_multixact_stats()`, que retorna `num_mxids`, `num_members`, `members_size` e `oldest_multixact`. Ela não é sucessora de `mxid_age()`: como o `MultiXactId` permanece com 32 bits, `mxid_age(datminmxid)` e `mxid_age(relminmxid)` continuam sendo as referências para envelhecimento e wraparound. Com o `MultiXactOffset` de 64 bits, a motivação original de monitorar wraparound do contador de membros deixou de existir; o valor principal da função está em observar padrões de alocação e o espaço consumido pelos membros. A chamada fornece um retrato instantâneo, então taxas exigem amostras periódicas. Na documentação das versões beta, todas as colunas retornam `NULL` por padrão sem os privilégios de `pg_read_all_stats`; confirme esse comportamento novamente na versão final. Até a 18, a função não existe, como descrito na [seção 8](/posts/postgresql-wraparound-parte-1-fundamentos/#8-multixact-o-contador-paralelo) e na [Situação D](#situação-d--multixact-próximo-do-limite).

**Logging de autovacuum se divide em dois parâmetros.** Na 19, `log_autovacuum_min_duration` passa a controlar apenas o logging das operações de vacuum, e o novo `log_autoanalyze_min_duration` cuida das operações de analyze. Quem depende hoje de um único parâmetro para os dois — inclusive no [checklist](#19-checklist-operacional) deste guia — precisa revisar a configuração ao migrar.

**Mais instrumentação de custo.** O PostgreSQL 19 acrescenta reporte de uso de memória e de paralelismo aos logs de `VACUUM (VERBOSE)` e do autovacuum, e reporte de bytes de *full-page write* em `VACUUM`, `ANALYZE`, `EXPLAIN (ANALYZE, WAL)` e `pg_stat_wal`. Isso permite separar a parcela de *full-page images* dentro do WAL total, enquanto o [Lab 3](/posts/postgresql-wraparound-parte-3-laboratorio-incidentes/#lab-3--medindo-wal-em-um-cenário-controlado) mede o WAL agregado por diferença de LSN em um cenário controlado.

**Comando `REPACK`.** O PostgreSQL 19 introduz um comando nativo que unifica no novo nome as funções históricas de `VACUUM FULL` e `CLUSTER`, ambos mantidos por compatibilidade. A opção `CONCURRENTLY` permite reescrever a tabela sem manter `ACCESS EXCLUSIVE` durante toda a operação, e um novo parâmetro `max_repack_replication_slots` limita os slots usados para isso. Apesar do nome e do objetivo semelhante, o comando nativo não deve ser tratado como implementação compatível ou intercambiável com a extensão `pg_repack` sem comparar sintaxe, locks, espaço temporário, replicação e limitações. `REPACK` não é uma ferramenta de wraparound — vale a mesma ressalva feita sobre `VACUUM FULL` na [seção 14](/posts/postgresql-wraparound-parte-3-laboratorio-incidentes/#14-anti-padrões) —, mas muda o repertório disponível para bloat.

### Minor releases também importam

Recursos de uma major version permanecem estáveis, mas correções de bugs, segurança e corrupção chegam por minor releases. A comunidade recomenda sempre a minor atual da major em uso.

No momento desta publicação, a página oficial de versionamento lista:

| Major | Minor atual | Suporte final |
|---:|---:|---|
| 18 | 18.6 | 14 de novembro de 2030 |
| 17 | 17.11 | 8 de novembro de 2029 |
| 16 | 16.15 | 9 de novembro de 2028 |
| 15 | 15.19 | 11 de novembro de 2027 |
| 14 | 14.24 | 12 de novembro de 2026 |

Confirme novamente essas versões antes de publicar ou executar o guia, pois minor releases mudam ao longo do tempo. O próximo ciclo trimestral está programado para 12 de novembro de 2026, mesma data do fim de suporte do PostgreSQL 14; a tabela acima deverá ser revista depois dessa data.

### E XIDs de 64 bits?

O `xid8` fornece uma representação com epoch para funções de usuário e monitoramento, mas o cabeçalho das tuplas continua armazenando XIDs truncados de 32 bits. Migrar toda a representação física para 64 bits teria impacto em formato de página, espaço por tupla, compatibilidade e atualização de clusters.

Até o PostgreSQL 18, o core continua dependendo de freeze e da manutenção do espaço circular de 32 bits.

> **Não confunda com a mudança do 19.** O alargamento para 64 bits anunciado no PostgreSQL 19 é do `MultiXactOffset`, o contador de posições das entradas de **membros** de MultiXact. Nem o `MultiXactId` nem o `TransactionId` passam a 64 bits. O `xmin` das tuplas continua sendo um XID de 32 bits, o congelamento continua obrigatório e a aritmética circular descrita na [seção 3](/posts/postgresql-wraparound-parte-1-fundamentos/#3-o-xid-32-bits-e-aritmética-circular) permanece igual. O que desaparece é uma classe específica de exaustão do armazenamento de membros, não o wraparound de XID nem toda a manutenção de MultiXacts.

---

## 19. Checklist operacional

### Diário ou por coleta automatizada

- [ ] coletar `max(age(datfrozenxid))` de todos os bancos;
- [ ] coletar `max(mxid_age(datminmxid))`;
- [ ] registrar taxa de consumo de `xid8`;
- [ ] calcular dias restantes com uma taxa conservadora;
- [ ] alertar para slots de replicação inativos e idades de `xmin`/`catalog_xmin`;
- [ ] alertar para prepared transactions;
- [ ] alertar para `idle in transaction` acima da política;
- [ ] acompanhar fila de arquivamento e lag das réplicas;
- [ ] detectar vacuums `(to prevent wraparound)` — no PostgreSQL 19, usar `started_by` e `mode` de `pg_stat_progress_vacuum`.

### Semanal

- [ ] revisar relações mais antigas por `relfrozenxid` e `relminmxid`;
- [ ] incluir TOAST e materialized views no ranking;
- [ ] revisar relações grandes com baixo `pct_frozen`;
- [ ] conferir logs de autovacuum lento;
- [ ] analisar vacuums com múltiplos ciclos de índice;
- [ ] comparar taxa de XIDs com a semana anterior;
- [ ] validar se a idade cai depois de vacuums agressivos.

### Mensal

- [ ] recalcular projeção de dias restantes;
- [ ] revisar `reloptions` fora do padrão;
- [ ] validar exceções de `autovacuum_enabled = false`;
- [ ] revisar `autovacuum_freeze_max_age` global e por tabela;
- [ ] identificar partições frias candidatas a freeze planejado;
- [ ] eliminar tabelas temporárias, cópias `_old` e staging sem finalidade;
- [ ] testar restauração do archive/PITR;
- [ ] revisar capacidade de `pg_wal`, archive e réplicas para campanhas futuras.

### Na criação de um cluster

- [ ] manter `autovacuum = on`;
- [ ] definir um SLO de idade e dias restantes;
- [ ] configurar `log_autovacuum_min_duration` de acordo com volume de logs (no PostgreSQL 19 esse parâmetro cobre apenas vacuum; o analyze passa a usar `log_autoanalyze_min_duration`);
- [ ] dimensionar workers, memória e custo com testes;
- [ ] configurar `idle_in_transaction_session_timeout` quando compatível;
- [ ] manter `max_prepared_transactions = 0` se 2PC não for usado;
- [ ] configurar limites e alertas para slots de replicação;
- [ ] validar que `archive_command` falha quando deve falhar;
- [ ] avaliar `wal_compression`;
- [ ] habilitar métricas de wraparound no exporter utilizado;
- [ ] documentar toda alteração de `autovacuum_freeze_max_age`;
- [ ] em serviço gerenciado, confirmar os defaults efetivos do provedor com `SHOW`, sem presumir os do PostgreSQL comunitário;
- [ ] executar a minor release atual da major escolhida.

### Antes de uma campanha manual

- [ ] confirmar versão e minor release;
- [ ] medir idade XID e MultiXact;
- [ ] medir taxa normal e de pico;
- [ ] verificar transações longas;
- [ ] verificar transações preparadas;
- [ ] verificar `xmin`, `catalog_xmin` e `restart_lsn` dos slots;
- [ ] confirmar saúde e espaço do arquivamento;
- [ ] confirmar lag e capacidade das réplicas;
- [ ] estimar páginas não `all_frozen` sem tratar o valor como previsão exata;
- [ ] definir lote, janela, concorrência e critérios de pausa;
- [ ] validar lock timeout e `TRUNCATE OFF`;
- [ ] testar em uma relação representativa;
- [ ] registrar estado antes e depois;
- [ ] confirmar avanço de `relfrozenxid` e `datfrozenxid`.

### Durante um incidente

- [ ] comunicar impacto de escrita de forma explícita;
- [ ] conferir a major version antes de interpretar o `HINT` do log;
- [ ] não reiniciar sem causa técnica;
- [ ] não usar modo mono-usuário como padrão, mesmo que a mensagem do servidor sugira isso;
- [ ] não usar `VACUUM FULL`;
- [ ] não começar com `VACUUM FREEZE` na salvaguarda final;
- [ ] resolver retenções antes do trabalho pesado;
- [ ] executar `VACUUM` normal como superusuário;
- [ ] monitorar progresso, WAL, armazenamento e réplicas;
- [ ] repetir por banco quando necessário;
- [ ] confirmar restauração das escritas;
- [ ] abrir ação corretiva para autovacuum, capacidade ou aplicação.

---

## 20. Referências

As referências oficiais abaixo são a fonte normativa para comportamento, parâmetros e procedimentos operacionais. O InterDB é usado como apoio conceitual e histórico.

### Documentação oficial — versão atual

- [Routine Vacuuming — Preventing Transaction ID Wraparound Failures](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [Vacuuming — autovacuum, custo, freeze e failsafe](https://www.postgresql.org/docs/current/runtime-config-vacuum.html)
- [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html)
- [COPY — opção FREEZE](https://www.postgresql.org/docs/current/sql-copy.html)
- [Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Subtransactions](https://www.postgresql.org/docs/current/subxacts.html)
- [Database Page Layout](https://www.postgresql.org/docs/current/storage-page-layout.html)
- [`pg_class`](https://www.postgresql.org/docs/current/catalog-pg-class.html)
- [`pg_database`](https://www.postgresql.org/docs/current/catalog-pg-database.html)
- [`pg_replication_slots`](https://www.postgresql.org/docs/current/view-pg-replication-slots.html)
- [`pg_prepared_xacts`](https://www.postgresql.org/docs/current/view-pg-prepared-xacts.html)
- [`pg_stat_progress_vacuum`](https://www.postgresql.org/docs/current/progress-reporting.html#VACUUM-PROGRESS-REPORTING)
- [WAL — full page writes e compressão](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- [`pg_resetwal`](https://www.postgresql.org/docs/current/app-pgresetwal.html)
- [Single-user mode](https://www.postgresql.org/docs/current/app-postgres.html)

### Extensões

- [`pageinspect`](https://www.postgresql.org/docs/current/pageinspect.html)
- [`pg_visibility`](https://www.postgresql.org/docs/current/pgvisibility.html)

### Notas de versão

- [PostgreSQL 8.3 — introdução do *Lazy XID*](https://www.postgresql.org/docs/release/8.3.0/)
- [PostgreSQL 9.6](https://www.postgresql.org/docs/release/9.6.0/)
- [PostgreSQL 12](https://www.postgresql.org/docs/release/12.0/)
- [PostgreSQL 13](https://www.postgresql.org/docs/release/13.0/)
- [PostgreSQL 14](https://www.postgresql.org/docs/release/14.0/)
- [PostgreSQL 15](https://www.postgresql.org/docs/release/15.0/)
- [PostgreSQL 16](https://www.postgresql.org/docs/release/16.0/)
- [PostgreSQL 17](https://www.postgresql.org/docs/release/17.0/)
- [PostgreSQL 18](https://www.postgresql.org/docs/release/18.html)
- [PostgreSQL 19 — notas de versão em desenvolvimento](https://www.postgresql.org/docs/19/release-19.html)
- [PostgreSQL 19 — configuração de autovacuum](https://www.postgresql.org/docs/19/runtime-config-vacuum.html)
- [PostgreSQL 19 — pontuação do autovacuum](https://www.postgresql.org/docs/19/monitoring-stats.html#PG-STAT-AUTOVACUUM-SCORES-VIEW)
- [PostgreSQL 19 — comando REPACK](https://www.postgresql.org/docs/19/sql-repack.html)
- [PostgreSQL 19 — `pg_stat_progress_vacuum` com `started_by` e `mode`](https://www.postgresql.org/docs/19/progress-reporting.html#PG-STAT-PROGRESS-VACUUM-VIEW)
- [PostgreSQL 19 — `pg_get_multixact_stats()`](https://www.postgresql.org/docs/19/functions-info.html#FUNCTIONS-INFO-SNAPSHOT)
- [Roadmap oficial — próximas minors e PostgreSQL 19](https://www.postgresql.org/developer/roadmap/)
- [Política de versões e suporte](https://www.postgresql.org/support/versioning/)

### Ambiente do laboratório

- [Imagem oficial `postgres` no Docker Hub — mudança de `PGDATA` e `VOLUME` na versão 18](https://hub.docker.com/_/postgres)

### Serviços gerenciados

- [Amazon RDS for PostgreSQL — extensões suportadas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL.Concepts.General.FeatureSupport.Extensions.html)
- [Amazon Aurora PostgreSQL — extensões por versão](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraPostgreSQLReleaseNotes/AuroraPostgreSQL.Extensions.html)
- [Cloud SQL for PostgreSQL — extensões suportadas](https://cloud.google.com/sql/docs/postgres/extensions)
- [Azure Database for PostgreSQL Flexible Server — extensões por versão](https://learn.microsoft.com/azure/postgresql/extensions/concepts-extensions-versions)

### Apoio conceitual

- [The Internals of PostgreSQL — índice](https://www.interdb.jp/pg/index.html)
- [The Internals of PostgreSQL — Transaction ID Wraparound](https://www.interdb.jp/pg/pgsql05/10.html)
- [The Internals of PostgreSQL — Visibility Map](https://www.interdb.jp/pg/pgsql06/02.html)
- [The Internals of PostgreSQL — Freeze Processing](https://www.interdb.jp/pg/pgsql06/03.html)

### Observabilidade

- [prometheus-community/postgres_exporter](https://github.com/prometheus-community/postgres_exporter)

> Ao usar documentação com URL `/current/`, registre a major version consultada. O conteúdo apontará para a versão corrente no futuro e pode mudar depois da publicação deste artigo.

---

## Fechamento

Wraparound não é um evento imprevisível. Ele é o resultado de um contador finito combinado com manutenção insuficiente, capacidade inadequada ou retenções que impedem o avanço.

O indicador mínimo é simples:

```sql
SELECT max(age(datfrozenxid))
FROM pg_database;
```

Mas operar com segurança exige mais quatro perguntas:

1. Qual é a taxa real de consumo de XIDs?
2. Quantos dias restam com uma margem conservadora?
3. Qual relação ou retenção impede o avanço?
4. Quanto tempo, I/O e WAL serão necessários para recuperar margem?

Um dashboard que responde essas perguntas transforma wraparound de crise inesperada em manutenção planejável.

---

*Procedimentos operacionais validados contra a documentação oficial do PostgreSQL 18 e comparados com as versões suportadas 14 a 18. As referências ao PostgreSQL 19 descrevem uma versão em beta e estão marcadas como tal. Os comandos com `pg_resetwal` destinam-se exclusivamente a containers descartáveis.*
