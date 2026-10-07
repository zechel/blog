---
title: "Transaction ID Wraparound no PostgreSQL (3/4): laboratório, anatomia de incidentes e anti-padrões"
date: 2026-10-07T09:00:00-03:00
description: "Um laboratório reproduzível com pg_resetwal em container descartável, as fases de um incidente real de wraparound e os anti-padrões mais comuns."
tags: ["postgresql", "dba", "vacuum", "wraparound"]
categories: ["Infraestrutura"]
toc: true
math: false
comments: true
draft: false
---

> **Laboratório, anatomia de incidentes e anti-padrões**
>
> **Navegação da série:** [Parte 1](/posts/postgresql-wraparound-parte-1-fundamentos/) · [Parte 2](/posts/postgresql-wraparound-parte-2-parametros-monitoramento/) · **Parte 3** · Parte 4 (em breve)

---

## 12. Laboratório reproduzível

Este laboratório usa PostgreSQL 18 em um volume Docker descartável. Os comandos de manipulação de `pg_resetwal` são deliberadamente destrutivos e não representam um procedimento de produção.

> **Nunca execute `pg_resetwal` em um cluster que contenha dados importantes.** Mesmo em laboratório, o servidor deve ser encerrado de forma limpa antes da alteração.

### Lab 0 — Criando o ambiente

> **Atenção: a imagem oficial mudou na versão 18.** Até a 17, o `PGDATA` padrão era `/var/lib/postgresql/data` e o `VOLUME` declarado apontava para esse mesmo caminho. A partir da 18, o `PGDATA` passou a ser versionado — `/var/lib/postgresql/18/docker` — e o `VOLUME` foi movido para o diretório pai `/var/lib/postgresql`. A mudança existe para permitir `pg_upgrade --link` entre majors dentro do mesmo volume.
>
> O efeito colateral é silencioso e desagradável: se você montar um volume nomeado em `/var/lib/postgresql/data` com a imagem 18, o container sobe normalmente, mas os arquivos do cluster vão para um volume anônimo em `/var/lib/postgresql/18/docker`. O volume que você criou fica vazio. Nos Labs 5 e 6, que montam esse volume em um container auxiliar, o `pg_resetwal` não encontraria o cluster.
>
> O laboratório abaixo resolve o problema fixando `PGDATA` explicitamente e montando o volume no diretório pai. Assim os caminhos usados no restante do texto permanecem válidos.

```bash
mkdir -p ~/lab-wraparound
cd ~/lab-wraparound

docker volume create pglab-data

docker run -d \
  --name pglab \
  -e POSTGRES_PASSWORD=lab \
  -e POSTGRES_DB=lab \
  -e PGDATA=/var/lib/postgresql/data \
  -p 55432:5432 \
  -v pglab-data:/var/lib/postgresql \
  postgres:18

for TENTATIVA in $(seq 1 60); do
  if docker exec pglab pg_isready -U postgres -d lab >/dev/null 2>&1; then
    break
  fi

  if [ "${TENTATIVA}" -eq 60 ]; then
    echo "O PostgreSQL não ficou pronto em 60 segundos."
    exit 1
  fi

  sleep 1
done

docker exec -it pglab \
  psql -U postgres -d lab -c "SELECT version();"
```

Confirme que o cluster está mesmo dentro do volume nomeado antes de seguir:

```bash
docker exec pglab psql -U postgres -d lab -tAc "SHOW data_directory;"

docker run --rm \
  -v pglab-data:/var/lib/postgresql \
  --entrypoint sh \
  postgres:18 \
  -c 'ls /var/lib/postgresql/data/pg_xact && head -1 /var/lib/postgresql/data/PG_VERSION'
```

A primeira consulta deve retornar `/var/lib/postgresql/data`. A segunda deve listar pelo menos um segmento (`0000`) e imprimir `18`. Se qualquer uma das duas falhar, o cluster não está no volume nomeado e os Labs 5 e 6 não vão funcionar.

> O laboratório não depende da presença de um link simbólico específico dentro da imagem. Ao fixar `PGDATA=/var/lib/postgresql/data`, o entrypoint cria ou reutiliza esse subdiretório dentro do volume montado no diretório pai. Em um volume nomeado novo, o Docker normalmente pré-popula o ponto de montagem com o conteúdo inicial da imagem; esse comportamento não deve ser presumido para *bind mounts* ou volumes criados com a opção equivalente a `volume-nocopy`. O requisito determinante é que os containers auxiliares dos Labs 5 e 6 montem o mesmo volume no mesmo diretório pai usado no Lab 0.

Crie um atalho para os próximos comandos:

```bash
psqll() {
  docker exec -i pglab \
    psql -U postgres -d lab -v ON_ERROR_STOP=1 "$@"
}
```

Extensões usadas no laboratório:

```bash
psqll <<'SQL'
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pg_visibility;
SQL
```

### Lab 1 — Quando um XID é atribuído

```bash
psqll <<'SQL'
BEGIN;

SELECT pg_current_xact_id_if_assigned() AS antes;
SELECT 1;
SELECT pg_current_xact_id_if_assigned() AS depois_do_select;

CREATE TEMP TABLE xid_demo (id int);
INSERT INTO xid_demo VALUES (1);

SELECT pg_current_xact_id_if_assigned() AS depois_da_escrita;
ROLLBACK;
SQL
```

O resultado esperado é:

- `antes`: `NULL`;
- `depois_do_select`: `NULL`;
- `depois_da_escrita`: um `xid8`.

Agora observe que a função sem o sufixo `_if_assigned` força a atribuição:

```bash
psqll <<'SQL'
BEGIN;
SELECT pg_current_xact_id_if_assigned() AS antes;
SELECT pg_current_xact_id() AS xid_forcado;
SELECT pg_current_xact_id_if_assigned() AS depois;
ROLLBACK;
SQL
```

### Lab 2 — Observando o congelamento

```bash
psqll <<'SQL'
DROP TABLE IF EXISTS congelamento;

CREATE TABLE congelamento AS
SELECT
    g AS id,
    repeat('x', 200) AS payload
FROM generate_series(1, 50000) AS g;

SELECT
    c.relname,
    age(c.relfrozenxid) AS idade,
    s.all_visible,
    s.all_frozen,
    pg_relation_size(c.oid)
        / current_setting('block_size')::int AS paginas
FROM pg_class AS c
CROSS JOIN LATERAL pg_visibility_map_summary(c.oid) AS s
WHERE c.oid = 'congelamento'::regclass;
SQL
```

Inspecione algumas tuplas da primeira página:

```bash
psqll <<'SQL'
SELECT
    lp,
    t_xmin,
    t_infomask,
    (t_infomask & 768) = 768 AS xmin_congelado
FROM heap_page_items(get_raw_page('congelamento', 0))
WHERE lp_off > 0
LIMIT 5;
SQL
```

Congele a tabela:

```bash
psqll <<'SQL'
VACUUM (FREEZE, VERBOSE) congelamento;

SELECT
    lp,
    t_xmin,
    t_infomask,
    (t_infomask & 768) = 768 AS xmin_congelado
FROM heap_page_items(get_raw_page('congelamento', 0))
WHERE lp_off > 0
LIMIT 5;

SELECT *
FROM pg_visibility_map_summary('congelamento');

SELECT age(relfrozenxid)
FROM pg_class
WHERE oid = 'congelamento'::regclass;
SQL
```

O `t_xmin` original permanece. O que muda são os bits de `t_infomask` que fazem o XID de inserção ser tratado como congelado.

### Lab 3 — Medindo WAL em um cenário controlado

O experimento a seguir força um checkpoint imediatamente antes do freeze. Isso maximiza a probabilidade de que a primeira modificação de cada página gere uma imagem completa de página (*full-page image*, FPI) quando `full_page_writes` está ativo.

```bash
psqll <<'SQL'
DROP TABLE IF EXISTS custo_wal;

CREATE TABLE custo_wal AS
SELECT
    g AS id,
    repeat('z', 500) AS payload
FROM generate_series(1, 500000) AS g;

CHECKPOINT;

SELECT pg_current_wal_lsn() AS lsn_antes \gset

VACUUM (FREEZE) custo_wal;

SELECT
    pg_size_pretty(pg_relation_size('custo_wal')) AS tamanho_heap,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), :'lsn_antes')
    ) AS wal_gerado,
    round(
        pg_wal_lsn_diff(pg_current_wal_lsn(), :'lsn_antes')::numeric
        / nullif(pg_relation_size('custo_wal'), 0),
        2
    ) AS fator_neste_teste;
SQL
```

O fator observado vale apenas para este teste. Ele depende de checkpoint, páginas já alteradas, compressão de WAL, versão, concorrência e quantidade de páginas realmente modificadas.

Veja as configurações relevantes:

```bash
psqll <<'SQL'
SHOW full_page_writes;
SHOW wal_compression;
SHOW checkpoint_timeout;
SHOW max_wal_size;
SQL
```

Para comparar compressão, recrie uma tabela equivalente depois de habilitar uma opção suportada pelo servidor:

```bash
psqll <<'SQL'
ALTER SYSTEM SET wal_compression = 'lz4';
SELECT pg_reload_conf();
SQL
```

Confirme o valor aplicado antes de repetir o teste:

```bash
psqll -c "SHOW wal_compression;"
```

A disponibilidade de `lz4` ou `zstd` depende de como o PostgreSQL foi compilado. Não presuma que toda distribuição oferece os mesmos algoritmos.

### Lab 4 — O segundo freeze tende a ser barato

```bash
psqll <<'SQL'
\timing on

VACUUM (FREEZE, VERBOSE) custo_wal;
VACUUM (FREEZE, VERBOSE) custo_wal;
SQL
```

Compare:

- páginas escaneadas;
- páginas congeladas;
- avanço de `relfrozenxid`;
- duração;
- WAL gerado.

Se ninguém modificou a tabela entre as execuções e todas as páginas já estão `all_frozen`, o segundo comando deve ter muito menos trabalho de heap. Em PostgreSQL 18, os logs também podem mostrar páginas percorridas por eager scanning em vacuums normais.

### Lab 5 — Envelhecendo o contador artificialmente

Crie uma função shell para alterar o próximo XID em um cluster parado. O volume é montado em um container auxiliar usando a mesma imagem e o mesmo usuário `postgres`.

```bash
avancar_xid() {
  ALVO="$1"
  SEGMENTO=$(printf '%04X' $(( ALVO / 1048576 )))

  echo "Próximo XID: ${ALVO}"
  echo "Segmento pg_xact: ${SEGMENTO}"

  INICIO_STOP=$(date -u +%Y-%m-%dT%H:%M:%SZ)
  docker stop --time 60 pglab >/dev/null

  EXIT_CODE=$(docker inspect --format '{{.State.ExitCode}}' pglab)
  if [ "${EXIT_CODE}" -ne 0 ]; then
    echo "O container terminou com código ${EXIT_CODE}; pg_resetwal não será executado."
    return 1 2>/dev/null || exit 1
  fi

  if ! docker logs --since "${INICIO_STOP}" pglab 2>&1 \
       | grep -q "database system is shut down"; then
    echo "O PostgreSQL não confirmou um shutdown limpo; pg_resetwal não será executado."
    return 1 2>/dev/null || exit 1
  fi

  docker run --rm \
    --user postgres \
    -v pglab-data:/var/lib/postgresql \
    postgres:18 \
    bash -ceu "
      ARQUIVO=/var/lib/postgresql/data/pg_xact/${SEGMENTO}
      if [ ! -f \"\${ARQUIVO}\" ]; then
        dd if=/dev/zero of=\"\${ARQUIVO}\" bs=256k count=1 status=none
      fi
      pg_resetwal -n -D /var/lib/postgresql/data
      pg_resetwal -x ${ALVO} -D /var/lib/postgresql/data
    "

  docker start pglab >/dev/null

  for TENTATIVA in $(seq 1 60); do
    if docker exec pglab pg_isready -U postgres -d lab >/dev/null 2>&1; then
      break
    fi

    if [ "${TENTATIVA}" -eq 60 ]; then
      echo "O PostgreSQL não ficou pronto em 60 segundos."
      return 1 2>/dev/null || exit 1
    fi

    sleep 1
  done
}
```

Repare que o container auxiliar monta o volume em `/var/lib/postgresql`, e não em `/var/lib/postgresql/data`. O ponto de montagem precisa ser o mesmo do Lab 0; o `PGDATA` continua sendo o subdiretório `data` dentro do volume.

O arquivo adicional em `pg_xact` existe apenas para permitir que o servidor encontre o segmento de status correspondente ao XID artificial. Zerar esse arquivo seria inaceitável em um cluster real.

A imagem oficial usa `STOPSIGNAL SIGINT`, portanto `docker stop` solicita um *fast shutdown*. O Docker, porém, força `SIGKILL` se o processo não terminar dentro do timeout. A função usa `--time 60`, exige código de saída zero do container e só prossegue quando encontra, nos logs produzidos depois do pedido de parada, a confirmação `database system is shut down`. Se qualquer uma dessas verificações falhar, o laboratório aborta antes de executar `pg_resetwal`. As esperas por `pg_isready` também possuem limite de 60 segundos para evitar loops indefinidos quando o servidor não inicia.

> O `grep` pressupõe mensagens em inglês, que é o padrão da imagem oficial. Se o container tiver sido criado com `lc_messages` em outro idioma, o teste falha mesmo com um shutdown correto. Nesse caso, ajuste o texto procurado ou rode o laboratório com o locale padrão.

Avance para aproximadamente 1,5 bilhão:

```bash
avancar_xid 1500000000
```

Confira o estado:

```bash
psqll <<'SQL'
SELECT
    datname,
    age(datfrozenxid) AS idade,
    round(
        100 * age(datfrozenxid)::numeric / 2147483648,
        1
    ) AS pct_da_meia_janela
FROM pg_database
ORDER BY idade DESC;
SQL
```

Agora observe a reação do autovacuum. Para registrar todas as execuções no laboratório:

```bash
psqll <<'SQL'
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
SQL

docker restart pglab >/dev/null
sleep 10

docker logs pglab 2>&1 \
  | grep -Ei 'wraparound|aggressive vacuum|automatic vacuum' \
  | tail -30
```

A tabela congelada no Lab 2 pode exigir pouco ou nenhum acesso às páginas de heap se continuar integralmente `all_frozen`. Outras relações precisarão ser processadas de acordo com seus marcos e Visibility Maps.

### Lab 6 — Salvaguarda final e recuperação atual

Este laboratório usa uma transação preparada (*prepared transaction*) como retenção deliberada. O alvo será calculado a partir do `datfrozenxid` mais antigo do cluster, evitando avançar o contador além da fronteira que queremos apenas observar.

Habilite 2PC somente neste container e aumente temporariamente o gatilho de anti-wraparound para que o cenário possa ser preparado de forma controlada:

```bash
psqll <<'SQL'
ALTER SYSTEM SET max_prepared_transactions = 10;
ALTER SYSTEM SET autovacuum_freeze_max_age = 2000000000;
SQL

docker restart pglab >/dev/null

for TENTATIVA in $(seq 1 60); do
  if docker exec pglab pg_isready -U postgres -d lab >/dev/null 2>&1; then
    break
  fi

  if [ "${TENTATIVA}" -eq 60 ]; then
    echo "O PostgreSQL não ficou pronto em 60 segundos."
    exit 1
  fi

  sleep 1
done
```

Crie uma transação preparada que permanecerá antiga durante o teste:

```bash
psqll <<'SQL'
DROP TABLE IF EXISTS bloqueio_wraparound;
CREATE TABLE bloqueio_wraparound (id int PRIMARY KEY);

BEGIN;
INSERT INTO bloqueio_wraparound VALUES (1);
SELECT pg_current_xact_id() AS xid_da_prepared;
PREPARE TRANSACTION 'lab_wraparound';

SELECT
    gid,
    transaction,
    age(transaction) AS idade
FROM pg_prepared_xacts;
SQL
```

Para tornar o ponto de partida previsível, congele os quatro bancos do container enquanto a transação preparada mantém o horizonte. `template0` será temporariamente aberto **somente neste laboratório descartável** e permanecerá acessível até a recuperação terminar:

```bash
psqll -d postgres \
  -c "ALTER DATABASE template0 ALLOW_CONNECTIONS true;"

for DB in lab postgres template1 template0; do
  docker exec -i pglab \
    psql -U postgres -d "${DB}" -v ON_ERROR_STOP=1 \
    -c "VACUUM (FREEZE, VERBOSE);"
done
```

Confira o banco com o marco mais antigo e capture seu XID absoluto:

```bash
BASE=$(
  psqll -At -F '|' -c \
    "SELECT datname, datfrozenxid::text::bigint
       FROM pg_database
      ORDER BY age(datfrozenxid) DESC
      LIMIT 1"
)

IFS='|' read -r BANCO_BASE XID_BASE <<< "${BASE}"

echo "Banco limitante: ${BANCO_BASE}"
echo "datfrozenxid base: ${XID_BASE}"
```

Calcule um próximo XID que deixe aproximadamente dois milhões de posições até a fronteira circular em relação ao marco global mais antigo. Como o laboratório já foi avançado para cerca de 1,5 bilhão no Lab 5, o resultado esperado permanece abaixo de `2^32`. O teste aborta em vez de tentar atravessar o wrap físico:

```bash
ALVO=$(( XID_BASE + 2147483648 - 2000000 ))

if (( ALVO >= 4294967296 )); then
  echo "O alvo cruzaria 2^32. Recrie o laboratório e repita a sequência."
  return 1 2>/dev/null || exit 1
fi

echo "Próximo XID artificial: ${ALVO}"

avancar_xid "${ALVO}"
```

Registre imediatamente as idades e tente uma operação que precise de novo XID:

```bash
psqll <<'SQL'
SELECT
    datname,
    datfrozenxid,
    age(datfrozenxid) AS idade_xid
FROM pg_database
ORDER BY idade_xid DESC;
SQL

psqll -c "CREATE TABLE deve_ser_recusada (id int);"
```

O erro esperado informa que o banco não aceita comandos que atribuam novos transaction IDs para evitar perda por wraparound. Processos internos podem alterar alguns XIDs entre o start e a consulta, mas a transação preparada impede que o horizonte avance livremente.

A recuperação moderna não exige modo mono-usuário. Primeiro, resolva a retenção criada pelo laboratório:

```bash
psqll -c "ROLLBACK PREPARED 'lab_wraparound';"
```

Depois execute `VACUUM` normal como superusuário em todos os bancos do container:

```bash
for DB in lab postgres template1 template0; do
  docker exec -i pglab \
    psql -U postgres -d "${DB}" -v ON_ERROR_STOP=1 \
    -c "VACUUM (VERBOSE);"
done
```

Confirme o avanço e teste uma nova escrita:

```bash
psqll <<'SQL'
SELECT
    datname,
    age(datfrozenxid) AS idade_xid,
    mxid_age(datminmxid) AS idade_multixact
FROM pg_database
ORDER BY idade_xid DESC;

CREATE TABLE escrita_restaurada (id int);
DROP TABLE escrita_restaurada;
SQL
```

Restaure a proteção padrão de `template0`:

```bash
psqll -d postgres \
  -c "ALTER DATABASE template0 ALLOW_CONNECTIONS false;"
```

Não use `VACUUM FULL` nessa situação. Ele precisa de XIDs e locks mais fortes. Também não escolha `VACUUM FREEZE` como primeira resposta operacional: perto da salvaguarda, o objetivo é avançar os marcos com o menor trabalho necessário. O `VACUUM FREEZE` acima foi usado apenas para preparar artificialmente um estado controlado de laboratório.

### Limpeza

```bash
docker rm -f pglab
docker volume rm pglab-data
```

---
## 13. Anatomia de um incidente

### Fase 1 — Crescimento discreto

A aplicação pode continuar operando sem sintomas evidentes enquanto `age(datfrozenxid)` cresce. A idade, isoladamente, não causa lentidão. O risco aparece quando o PostgreSQL precisa processar muito trabalho em pouco tempo ou quando algum horizonte impede o avanço.

Sinais antecipados:

- idade crescente sem quedas depois de vacuums agressivos;
- relações grandes com `relfrozenxid` antigo;
- baixa porcentagem de páginas `all_frozen` em dados estáticos;
- slots de replicação com `xmin` ou `catalog_xmin` antigos;
- transações preparadas esquecidas;
- transações longas recorrentes;
- taxa de XIDs maior que a prevista.

### Fase 2 — Avisos próximos ao limite

Quando o banco mais antigo chega a aproximadamente quarenta milhões de XIDs da fronteira, o PostgreSQL começa a emitir avisos semelhantes a:

```text
WARNING:  database "producao" must be vacuumed within 39985967 transactions
HINT:  To avoid XID assignment failures, execute a database-wide VACUUM in that database.
        You might also need to commit or roll back old prepared transactions,
        or drop stale replication slots.
```

> **A redação depende da versão.** O texto acima é o do PostgreSQL 17 e 18. A reformulação dessas mensagens entrou no ciclo da 17 e **não foi retroportada**. Em PostgreSQL 14, 15 e 16 o mesmo evento produz:
>
> ```text
> WARNING:  database "producao" must be vacuumed within 39985967 transactions
> HINT:  To avoid a database shutdown, execute a database-wide VACUUM in that database.
>         You might also need to commit or roll back old prepared transactions,
>         or drop stale replication slots.
> ```
>
> O número e o significado são os mesmos. Muda apenas a descrição da consequência: o servidor não vai "desligar", ele vai parar de atribuir novos XIDs.

Quarenta milhões podem representar meses ou horas. Em um cluster que consome 40 milhões de XIDs por dia, a margem é aproximadamente um dia.

Esse aviso deve abrir incidente imediatamente. Não é um alerta para “avaliar na próxima manutenção”.

### Fase 3 — Vacuums agressivos concorrendo por recursos

Quando várias relações cruzam limiares próximos, o autovacuum pode ter mais trabalho elegível que workers disponíveis. Isso é comum após:

- restauração ou migração que criou muitas relações na mesma janela;
- carga inicial de um data warehouse;
- criação em massa de partições;
- aumento tardio da taxa transacional;
- configuração que adiou manutenção por muito tempo.

Possíveis efeitos:

- aumento de leituras e escritas;
- mais WAL;
- fila de arquivamento;
- lag em réplicas;
- disputa por buffers e armazenamento;
- degradação de latência na aplicação.

Um vacuum identificado em `pg_stat_activity` com o sufixo `(to prevent wraparound)` recebe tratamento especial: ao contrário de autovacuums comuns, ele não é automaticamente interrompido por todo lock conflitante. Cancelá-lo sem diagnóstico apenas mantém a relação elegível e reduz a margem restante.

> **No PostgreSQL 19**, essa identificação por texto deixa de ser necessária: `pg_stat_progress_vacuum` passa a expor as colunas `started_by` e `mode`. Nas versões 14 a 18, o sufixo na coluna `query` é o sinal disponível.

### Fase 4 — Failsafe

Ao atingir o failsafe efetivo, o `VACUUM` passa a priorizar o avanço contra wraparound. Entre outras medidas, ele pode:

- ignorar o cost delay;
- pular limpeza de índices;
- reduzir operações secundárias que atrasariam a conclusão.

Nesse estágio, alterar `autovacuum_vacuum_cost_delay` pode não produzir o efeito esperado, porque o failsafe foi projetado justamente para remover o freio.

### Fase 5 — Bloqueio de novas atribuições

Quando restam menos de aproximadamente três milhões de XIDs, o PostgreSQL recusa comandos que precisem atribuir novos XIDs:

```text
ERROR:  database is not accepting commands that assign new transaction IDs
        to avoid wraparound data loss in database "producao"
HINT:  Execute a database-wide VACUUM in that database.
        You might also need to commit or roll back old prepared transactions,
        or drop stale replication slots.
```

> **Se você opera PostgreSQL 14, 15 ou 16, a mensagem que vai aparecer é outra — e ela contradiz este guia.** A redação anterior é:
>
> ```text
> ERROR:  database is not accepting commands to avoid wraparound data loss
>         in database "producao"
> HINT:  Stop the postmaster and vacuum that database in single-user mode.
>         You might also need to commit or roll back old prepared transactions,
>         or drop stale replication slots.
> ```
>
> **Esse hint é histórico e não deve ser seguido.** Ele data de uma época em que o `VACUUM` consumia um XID e realmente não podia ser executado nessa condição. Isso deixou de ser verdade com o mecanismo de *Lazy XID* introduzido no PostgreSQL 8.3 — implementado em 2007 e lançado em 4 de fevereiro de 2008 —, mas a mensagem só foi corrigida no ciclo da versão 17, sem retroporte para as versões anteriores.
>
> Em qualquer versão suportada hoje, o `VACUUM` roda normalmente com o servidor no ar. Parar o postmaster para entrar em modo mono-usuário desliga salvaguardas importantes e deixa a aplicação totalmente fora do ar em vez de somente leitura. Isso não é necessário para que o `VACUUM` execute e não oferece benefício operacional que justifique o risco e a indisponibilidade no procedimento padrão. A seção 16 descreve o procedimento correto.

Nessa condição:

- transações já em andamento podem continuar, sujeitas ao estado do sistema;
- novas transações somente leitura podem ser iniciadas;
- operações que modifiquem registros ou façam `TRUNCATE` falham;
- `VACUUM` normal continua disponível.

Do ponto de vista do negócio, uma aplicação OLTP pode estar efetivamente indisponível, mesmo que consultas simples ainda respondam.

### Fase 6 — Recuperação

A recuperação atual é feita com o servidor em execução:

1. resolver transações preparadas antigas;
2. encerrar ou concluir transações que prendem o horizonte;
3. remover slots realmente obsoletos;
4. executar `VACUUM` normal como superusuário no banco afetado;
5. repetir nos demais bancos que apresentem idade crítica;
6. confirmar o avanço de `datfrozenxid` e corrigir a causa.

Modo mono-usuário é uma exceção, não o procedimento padrão. Ele só faz sentido em cenários específicos, como quando a estratégia escolhida é `DROP` ou `TRUNCATE` de relações dispensáveis para evitar processá-las. O modo mono-usuário desabilita salvaguardas importantes e aumenta o risco operacional.

### A assimetria entre prevenção e resposta

| Aspecto | Prevenção planejada | Incidente próximo ao limite |
|---|---|---|
| Momento | janela escolhida | execução imediata |
| Estratégia | lotes, teste e telemetria | menor trabalho capaz de restaurar margem |
| Throttle | calibrado | pode ser reduzido ou ignorado pelo failsafe |
| Mudanças | podem passar por homologação | decisões sob pressão |
| Risco de negócio | baixo e controlável | indisponibilidade ou forte degradação |

O trabalho físico pode ser semelhante. O custo operacional é radicalmente diferente porque a margem de decisão desaparece.

---

## 14. Anti-padrões

### Aumentar `autovacuum_freeze_max_age` apenas para parar vacuums

O parâmetro pode ser maior em arquiteturas específicas, mas não deve ser usado como botão de silêncio. Antes de qualquer aumento, documente:

- motivação;
- taxa atual e de pico de XIDs;
- tempo de vacuum das maiores relações;
- failsafe efetivo;
- margem em dias;
- crescimento de `pg_xact` e `pg_commit_ts`;
- plano de reversão.

Sem isso, o ajuste apenas troca manutenção frequente por risco concentrado.

### Desabilitar autovacuum em tabelas grandes

```sql
ALTER TABLE grande SET (autovacuum_enabled = false);
```

Isso não bloqueia o anti-wraparound. Apenas remove manutenção preventiva normal e pode deixar mais tuplas mortas, bloat e estatísticas antigas para o momento de emergência.

### Desabilitar autovacuum globalmente

```text
autovacuum = off
```

Mesmo assim, processos anti-wraparound podem ser iniciados. O resultado é perder a maior parte da manutenção preventiva sem remover a proteção de última instância.

### Cancelar todo vacuum anti-wraparound

Antes de cancelar, compare:

- XIDs restantes;
- taxa de consumo;
- progresso atual;
- estado do WAL;
- impacto no armazenamento;
- possibilidade de outro worker assumir a relação.

Uma relação que ainda excede o gatilho continuará elegível. Cancelamentos repetidos podem transformar uma degradação controlável em bloqueio de escrita.

### Usar `VACUUM FREEZE` como primeira resposta no limite final

`VACUUM FREEZE` equivale a usar idades mínimas de freeze iguais a zero. Perto da salvaguarda, ele pode fazer mais trabalho que o necessário.

A orientação atual é executar `VACUUM` normal depois de remover os bloqueadores. O objetivo imediato é avançar os marcos o suficiente para restaurar a capacidade de atribuir XIDs.

### Usar `VACUUM FULL` durante a salvaguarda

`VACUUM FULL` reescreve a relação, exige `ACCESS EXCLUSIVE` e precisa de recursos transacionais adicionais. Ele não é uma ferramenta de recuperação de wraparound. Reserve-o para casos planejados de compactação física.

### Rodar freeze sem verificar retenções

Se uma transação, uma transação preparada ou um slot mantém um horizonte antigo, uma campanha pode gerar muito I/O e pouco avanço. Sempre compare o estado dos bloqueadores antes e depois.

### Aplicar `INDEX_CLEANUP OFF` indiscriminadamente

A opção pode reduzir a duração em emergência ou em relações realmente imutáveis. Em tabelas modificadas, uso contínuo acumula entradas mortas nos índices e ponteiros de linha que só poderão ser removidos por uma futura limpeza.

Para uso normal, mantenha `AUTO`.

### Inferir trabalho de índices apenas pelo Visibility Map

`total − all_visible` não significa “páginas que exigem index cleanup”. O conjunto inclui páginas recentes, páginas ainda não visitadas, páginas com tuplas mortas e outros estados. O VM é uma excelente telemetria de freeze, mas não substitui estatísticas do `VACUUM`, bloat e comportamento dos índices.

### Converter `vacuum_cost_limit` diretamente em MB/s

Unidades de custo representam hits, misses e páginas sujas com pesos diferentes. A vazão resultante depende de cache, concorrência e armazenamento.

```sql
SET vacuum_cost_delay = '2ms';
SET vacuum_cost_limit = 600;
```

Esse par é apenas um ponto de teste. Meça `iostat`, métricas do volume, latência da aplicação e progresso do `VACUUM` para calibrar.

### Fazer o `archive_command` sempre retornar sucesso

Exemplo incorreto:

```bash
archive_command = 'ferramenta %p; gzip < %p > /destino/%f.gz; exit 0'
```

O PostgreSQL considera código de saída zero como arquivamento concluído. Se um comando anterior falhar e a cadeia terminar com `exit 0`, o segmento pode ser reciclado sem ter sido armazenado corretamente.

Use uma única ferramenta confiável ou uma cadeia que preserve a falha:

```bash
archive_command = 'ferramenta %p'
```

Teste restauração, não apenas a existência de arquivos no destino.

### Tratar métricas do exporter como nomes universais

Coletores, nomes e tipos de métricas mudam entre exporters e versões. Habilite explicitamente o coletor de wraparound, consulte `/metrics` e escreva alertas contra a saída real do ambiente.

---

