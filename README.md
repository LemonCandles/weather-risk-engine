# 🧭 Bússola Climática

A Bússola Climática é um motor de análise de risco meteorológico. A proposta é coletar dados de uma fonte climática, preservar sua origem, avaliar a qualidade das informações e calcular indicadores de risco por localidade e período. Cada resultado deve mostrar os fatores que o explicam, a fonte, o horário de atualização e o intervalo em que é válido.

Os indicadores são apoio analítico e não substituem alertas de serviços meteorológicos ou de defesa civil. O primeiro método será uma regra transparente e experimental; sua pontuação não deve ser interpretada como probabilidade calibrada de um evento ou de impacto local.

## Arquitetura

O sistema é dividido em dois componentes que trocam arquivos versionados:

```text
Fonte meteorológica
       |
       v
Go: cliente HTTP e coletor concorrente
       |
       v
C1: pacote de coleta em JSON/NDJSON
       |
       v
Python: arquivo bruto, normalização, qualidade e risco
       |                         |
       |                         +--> Parquet e DuckDB para análise e auditoria
       v
C2: snapshot JSON publicado atomicamente
       |
       v
Go: API REST de consulta
```

O **coletor Go** consulta as localidades configuradas com limites de concorrência, timeout, cancelamento e tentativas controladas. Ele converte horários e unidades para o contrato C1 e preserva a resposta original quando os termos da fonte permitem. O pacote contém também os resultados e falhas de cada coleta.

O **pipeline Python** lê C1, guarda dados brutos, identifica registros duplicados ou inválidos, constrói variáveis e calcula uma avaliação de risco explicável. Parquet e DuckDB organizam os dados analíticos internamente. Ao terminar, o pipeline exporta um snapshot C2 com avaliações, fatores, qualidade e atualidade. Uma publicação só substitui a anterior depois de validada; falhas preservam o último snapshot íntegro.

A **API Go** lê somente C2. Cada requisição usa uma versão fixa do snapshot e retorna a avaliação com sua procedência e validade. A coleta e o processamento acontecem fora das requisições HTTP, para que a lentidão da fonte meteorológica não bloqueie as consultas. Dados antigos ou insuficientes são sinalizados explicitamente; ausência de avaliação não equivale a risco baixo.

A fronteira entre os componentes tem apenas dois formatos externos: **C1**, produzido por Go e consumido por Python, e **C2**, produzido por Python e consumido pela API Go. Ambos terão schema, versão e exemplos de teste. O provedor, as localidades, o horizonte e os limiares do primeiro risco serão definidos antes da implementação.

## Tecnologias previstas

| Tecnologia | Papel no projeto |
|---|---|
| **Go** | Coleta HTTP concorrente e API REST. |
| **Python** | Validação, transformação de dados e cálculo do risco. |
| **uv** | Ambiente e dependências reproduzíveis do componente Python. |
| **Polars** | Manipulação tabular durante o processamento. |
| **Parquet** | Armazenamento colunar dos dados brutos e processados. |
| **DuckDB** | Consultas e materialização analítica dentro do componente Python. |
| **JSON/NDJSON** | Contratos C1 e C2 entre os componentes. |

A escolha da fonte meteorológica e de seus campos dependerá da cobertura, das unidades, da frequência, dos limites de uso e da licença para conservar os dados necessários. Métodos estatísticos mais complexos poderão ser comparados à regra inicial quando houver dados adequados, sem substituir automaticamente um resultado explicável por um modelo experimental.
