# sql-integration-analysis

🔍 Você já viu um relatório com horários errados e pensou "deve ser bug do sistema"?

Recentemente me deparei exatamente com isso durante uma análise de integração entre dois sistemas.

Os horários de um evento no sistema A estavam sempre 3 horas adiantados no sistema B. Parecia simples — e era. Mas tinha um segundo problema escondido atrás do primeiro.

𝗢 𝗾𝘂𝗲 𝗲𝗻𝗰𝗼𝗻𝘁𝗿𝗲𝗶:

→ Problema 1: o sistema externo enviava timestamps em UTC-3 (Brasília), mas o banco de dados receptor operava em UTC-0. Sem conversão de fuso na importação, todos os registros chegavam com 3 horas a mais.

→ Problema 2: alguns registros chegavam com atraso variável — 30 minutos, 1 hora, às vezes mais. Parecia falha de sincronização. Mas quando isolei o fuso horário na query, o atraso real era baixo na maioria dos casos. Os outliers tinham explicação: eventos que demoravam mais para encerrar no sistema de origem.

𝗗𝗼𝗶𝘀 𝗽𝗿𝗼𝗯𝗹𝗲𝗺𝗮𝘀 𝗱𝗶𝘀𝘁𝗶𝗻𝘁𝗼𝘀 𝗺𝗮𝘀𝗰𝗮𝗿𝗮𝗱𝗼𝘀 𝘂𝗺 𝗽𝗲𝗹𝗼 𝗼𝘂𝘁𝗿𝗼.

𝗔 𝗾𝘂𝗲𝗿𝘆 𝗾𝘂𝗲 𝘀𝗲𝗽𝗮𝗿𝗼𝘂 𝗼𝘀 𝗱𝗼𝗶𝘀:

SELECT
  (created_at - INTERVAL '3 hours') AS timestamp_local,
  ((created_at - INTERVAL '3 hours') - end_date) AS atraso_real
FROM integration.records
WHERE source = 'external'
ORDER BY created_at DESC;

Subtrair o fuso direto na query revelou que o atraso de sincronização era mínimo — o problema real era só a conversão de timezone.

𝗔𝗽𝗿𝗲𝗻𝗱𝗶𝘇𝗮𝗱𝗼:

Em integrações entre sistemas, sempre verifique o timezone de cada ponta antes de comparar timestamps. Um dado "errado" muitas vezes é só um dado mal interpretado.

O repositório com as queries completas está no meu GitHub 👇
[link do repositório]

#SQL #AnáliseDeDados #PostgreSQL #DataAnalysis #Integração #BI
