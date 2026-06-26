# sql-integration-analysis

Análise de Divergência em Integração de Sistemas

Contexto

Durante operação assistida em um cliente do setor industrial, foi identificada uma divergência sistemática entre os horários registrados em um sistema externo de controle e os horários exibidos no relatório da plataforma de monitoramento.

A investigação foi conduzida diretamente no banco de dados de produção (PostgreSQL/TimescaleDB) utilizando SQL.


Problema Identificado

A análise revelou dois problemas distintos coexistindo na integração:

Problema 1 — Divergência de Fuso Horário

O sistema externo enviava timestamps em UTC-3 (horário de Brasília). O banco de dados da plataforma operava em UTC-0. A ausência de conversão na importação resultava em todos os registros sendo gravados com 3 horas a mais em relação ao horário real.

Problema 2 — Atraso na Sincronização

Independente do fuso horário, alguns registros chegavam com atraso variável após o encerramento do evento. Após isolar o fuso horário, foi possível medir o atraso real de sincronização de forma isolada.


Metodologia

1. Validação de registro específico

sqlSELECT
    id,
    xid,
    start_date,
    end_date,
    created_at,
    ROUND((EXTRACT(EPOCH FROM (created_at - start_date)) / 3600)::numeric, 1) AS diferenca_horas,
    value
FROM integration.records
WHERE xid = 'ext-000001';

2. Distribuição de diferenças por quantidade de registros

sqlSELECT
    ROUND(EXTRACT(EPOCH FROM (created_at - start_date)) / 3600) AS diferenca_horas,
    COUNT(*) AS quantidade
FROM integration.records
WHERE source = 'external'
GROUP BY diferenca_horas
ORDER BY diferenca_horas;

3. Resumo executivo do impacto

sqlSELECT
    COUNT(*) AS total_registros,
    COUNT(CASE WHEN EXTRACT(EPOCH FROM (created_at - start_date)) / 3600 <= 4 THEN 1 END) AS sincronizacao_normal,
    COUNT(CASE WHEN EXTRACT(EPOCH FROM (created_at - start_date)) / 3600 > 4  THEN 1 END) AS sincronizacao_atrasada,
    ROUND(AVG(EXTRACT(EPOCH FROM (created_at - start_date)) / 3600)::numeric, 1) AS media_horas_atraso,
    ROUND(MAX(EXTRACT(EPOCH FROM (created_at - start_date)) / 3600)::numeric, 1) AS maior_atraso_horas
FROM integration.records
WHERE source = 'external';

4. Isolamento do fuso horário para medir atraso real

sqlSELECT
    r.id,
    r.xid,
    r.start_date,
    r.end_date,
    (r.end_date - r.start_date)                          AS duracao_evento,
    r.created_at,
    (r.created_at - r.start_date)                        AS diferenca_total,
    (r.created_at - INTERVAL '3 hours')                  AS created_at_local,
    ((r.created_at - INTERVAL '3 hours') - r.start_date) AS atraso_sincronizacao,
    ((r.created_at - INTERVAL '3 hours') - r.end_date)   AS atraso_apos_encerramento
FROM integration.records r
WHERE r.source = 'external'
AND r.start_date >= NOW() - INTERVAL '7 days'
ORDER BY r.created_at DESC;


Conclusões


O problema de fuso horário afetava 100% dos registros e tinha causa raiz bem definida
O atraso de sincronização, isolado do fuso, era baixo na maioria dos casos (menos de 20 minutos)
Registros com atraso maior tinham explicação legítima — eventos de longa duração que demoravam mais para encerrar no sistema externo
Os dois problemas estavam mascarados um pelo outro e só foram separados após isolar o fuso horário na query



Aprendizados Técnicos


EXTRACT(EPOCH FROM interval) converte qualquer intervalo de tempo para segundos no PostgreSQL
Subtrair INTERVAL '3 hours' de um timestamp converte UTC-0 para UTC-3 diretamente na query
Comparar created_at - end_date (e não created_at - start_date) isola o atraso real de sincronização, excluindo a duração do evento
Em integrações entre sistemas, sempre verificar o timezone de cada ponta antes de comparar timestamps



Stack


PostgreSQL 14 + TimescaleDB
DBeaver (cliente SQL)
Análise exploratória via SQL puro



Autor

Julio Antunes
