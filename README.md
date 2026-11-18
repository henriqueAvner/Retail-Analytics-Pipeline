# EPT Analytics Pipeline

Pipeline de Engenharia de Dados para o **ranking EPT de League of Legends**. O projeto organiza dados de equipes, partidas, campeonatos e pontuação que estavam distribuídos em abas de uma planilha. O resultado são sete tabelas que permitem consultar o ranking atual, relacionar partidas aos cadastros e analisar regras de pontuação e colocações.

## Fluxo implementado

```mermaid
flowchart LR
    A["Google Sheets · EPT LoL"] -->|gspread| B["4 arquivos Parquet brutos"]
    B --> C["4 notebooks de transformação em pandas"]
    C --> D["7 tabelas em Excel"]
    D -->|pandas-gbq| E["BigQuery · femeeData"]
```

A [extração](src/extract/extract.ipynb) lê as abas **PONTUAÇÃO LoL**, **PARTIDAS LOL**, **CAMPEONATOS** e **PONTUAÇÃO TIER** da [planilha EPT LoL](https://docs.google.com/spreadsheets/d/1kMau11VHrNjGijHWZ1ii0c_U2zcuCTUE_T04MReKjJE/edit). Cada aba é gravada em um Parquet comprimido com Snappy. Os notebooks de transformação usam essas cópias locais e exportam os resultados em Excel. O [notebook de carga](src/load/load_GBQ.ipynb) lê os sete arquivos e envia as tabelas ao BigQuery, com substituição individual por tabela.

## Transformações na prática

- **Equipes e pontuação.** Separei o cadastro de equipes, que reúne ranking atual, ranking base e penalidade por inatividade, dos pontos por evento. Converti as colunas de eventos da planilha em linhas por equipe e rótulo de campeonato. [Notebook](src/transform/transform_pontuacao.ipynb)
- **Campeonatos e colocações.** Separei os dados cadastrais de cada torneio das colocações. As posições, originalmente distribuídas em colunas, passaram a ser linhas identificadas por campeonato e posição. [Notebook](src/transform/transform_campeonatos.ipynb)
- **Partidas e vínculos por ID.** Limpei linhas sem equipes, converti datas, duração e pontos, e padronizei nomes antes dos cruzamentos. Usei equivalências explícitas para nomes como `sesc_geek` → `sesc_geek_2024`. Os cruzamentos à esquerda com validação de cardinalidade preservam os registros de partidas e associam os IDs do torneio, das equipes A e B e da vencedora. [Notebook](src/transform/transform_partidas.ipynb)
- **Regras por tier.** Separei as matrizes de pontuação por resultado e por colocação. Usei `melt()` para transformar as colunas de cada matriz em linhas. [Notebook](src/transform/transform_pontuacao_tier.ipynb)

### Exemplo real — pontos para BO3 no tier S

Recorte da matriz antes da transformação:

| tier | bo3_win | bo3_loss |
|---|---:|---:|
| TIER S | 15 | -15 |

As duas regras após a transformação, no formato usado pela tabela de saída:

| tier | formato | pontos |
|---|---|---:|
| S | bo3_win | 15 |
| S | bo3_loss | -15 |

## Tabelas produzidas

As contagens correspondem aos arquivos Excel versionados no repositório.

| Arquivo tratado | Registros | O que cada linha representa |
|---|---:|---|
| [Times.xlsx](src/transform/excel_transform/times/Times.xlsx) | 224 | Uma equipe e seus indicadores de ranking |
| [Pontuacao_Times.xlsx](src/transform/excel_transform/times/Pontuacao_Times.xlsx) | 356 | Pontos de uma equipe por rótulo de campeonato, separados entre partidas e colocação |
| [Campeonatos.xlsx](src/transform/excel_transform/campeonatos/Campeonatos.xlsx) | 34 | Um campeonato |
| [Colocacoes_Campeonatos.xlsx](src/transform/excel_transform/campeonatos/Colocacoes_Campeonatos.xlsx) | 201 | Uma posição preenchida em um campeonato |
| [Resultado_Partidas.xlsx](src/transform/excel_transform/partidas/Resultado_Partidas.xlsx) | 525 | Um registro de partida com equipes, campeonato, vencedor e pontos |
| [Pontuacao_tier.xlsx](src/transform/excel_transform/tier/Pontuacao_tier.xlsx) | 30 | Uma regra de pontos por tier e combinação de formato e resultado |
| [Tier_Colocacao.xlsx](src/transform/excel_transform/tier/Tier_Colocacao.xlsx) | 32 | Uma regra de pontos por tier e posição final |

Nos arquivos atuais, os 525 registros de partida apontam para campeonatos e equipes presentes nos cadastros. O vencedor é uma das duas equipes em todos os registros. Há 2 partidas sem data e 460 sem duração. Na tabela de pontuação de equipes, o campeonato segue como rótulo textual. Nas colocações, a equipe é identificada pelo nome. O ranking representa a posição atual da equipe na fonte.

## Como percorrer o projeto

1. Em `src/extract/`, rode [extract.ipynb](src/extract/extract.ipynb) para atualizar os quatro Parquets em `arquivos_brutos/`. Os Parquets já estão no repositório para reproduzir as transformações com a amostra salva. Para atualizar a extração, configure o caminho do JSON na célula de autenticação e use uma conta de serviço com acesso à planilha.
2. Em `src/transform/`, rode [transform_pontuacao.ipynb](src/transform/transform_pontuacao.ipynb), [transform_campeonatos.ipynb](src/transform/transform_campeonatos.ipynb) e [transform_pontuacao_tier.ipynb](src/transform/transform_pontuacao_tier.ipynb). Em seguida, rode [transform_partidas.ipynb](src/transform/transform_partidas.ipynb), que lê os cadastros de equipes e campeonatos exportados pelos dois primeiros.
3. Para a carga, crie um arquivo `.env` na raiz e defina o caminho do JSON da conta de serviço. Um caminho relativo é resolvido a partir da pasta do `.env`.

   ```dotenv
   GOOGLE_APPLICATION_CREDENTIALS=cred/sua-conta-de-servico.json
   ```

   Em `src/load/`, abra [load_GBQ.ipynb](src/load/load_GBQ.ipynb) e confira a configuração de projeto e dataset. A última célula grava no BigQuery com `replace` e substitui as tabelas de destino.

Os notebooks usam Python e pandas. A extração usa `gspread`, `oauth2client` e `pyarrow`. A leitura e escrita dos Excels usa `openpyxl`. A carga usa `pandas-gbq`, `google-cloud-bigquery` e `python-dotenv` para ler o `.env`.
