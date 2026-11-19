# EPT LoL no Power BI

O relatório terá três páginas principais — **Ranking e contribuição de pontos**, **Partidas e desempenho** e **Campeonatos e colocações**. A página **Regras e conferência** documenta a pontuação e permite auditar a cobertura dos dados.

## 1. Conferir as consultas já abertas

Use as tabelas que você já trouxe do dataset `femeeproject.femeeData`. O [notebook de carga](src/load/load_GBQ.ipynb) lê os Excels tratados e publica os DataFrames com os nomes abaixo, sem declarar um esquema explícito. Os campos e as contagens vêm dos arquivos locais. Confirme na prévia do Power Query os nomes, tipos e valores da sua carga.

| Consulta BigQuery | O que cada linha representa | Campos esperados | Referência local |
|---|---|---|---:|
| `Times` | Uma equipe com o ranking atual | `time_id`, `nome_da_equipe`, `ranking_atual_automatico`, `ranking_base`, `penalidade_inatividade` | 224 |
| `Campeonatos` | Um campeonato ou edição cadastrada | `id_camp`, `nome_do_campeonato`, `tier`, `status`, `link`, `dat_inicio`, `dat_termino` | 34 |
| `Partidas` | Um registro identificado por `match_id` | `match_id`, `data`, `tier`, `stage`, `game_time`, `match_format`, `pts_time_a`, `pts_time_b`, `tournament_id`, `team_a_id`, `team_b_id`, `winner_team_id` | 525 |
| `Colocacoes_Campeonatos` | Uma posição preenchida em um campeonato | `id_camp`, `placement_rank`, `team_name` | 201 |
| `Pontuacao_Times` | Pontos de uma equipe por rótulo de evento | `time_id`, `campeonato`, `pontos_colocacao`, `pontos_partidas` | 356 |
| `Tier_Pontuacao` | Uma regra de pontos por tier, formato e resultado | `tier`, `formato` (ex. `bo3_win`), `pontos` | 30 |
| `Tier_Colocacao` | Uma regra de pontos por tier e posição | `tier`, `colocacao` (ex. `1º lugar`), `pontos` | 32 |

- [ ] **T01 — Conferir a navegação e o perfil das sete consultas.** Mantenha os nomes da tabela acima para usar o DAX deste guia. Como a origem já entrega tabelas com cabeçalhos, comece pela etapa de navegação existente. Em **Exibir**, habilite qualidade e perfil de coluna e selecione o perfil do conjunto inteiro. Registre a data da carga, as contagens, os nulos e os erros por consulta. Use **Referência** para criar verificações auxiliares e desabilite a carga delas quando servirem apenas para conferência. [Perfil e tipos no Power Query](https://learn.microsoft.com/power-query/common-issues)
- [ ] **T02 — Ajustar tipos e campos para o relatório.** Use número inteiro para IDs, `placement_rank` e pontos inteiros após conferir se a conversão preserva os valores. `id_camp = 0` é válido. Use decimal para ranking atual, ranking base, penalidade e `game_time`. A penalidade é fração, então 0,71 deve aparecer como 71%. O [tratamento de partidas](src/transform/transform_partidas.ipynb) grava `game_time` em minutos decimais. Só leve duração aos visuais se a cobertura for suficiente e exibida. Preserve nulos, zeros e pontos negativos. Inspecione colunas extras na sua carga e remova apenas as confirmadas como descartáveis.
- [ ] **T03 — Criar campos de filtro.** Em `Partidas`, crie `DataPartida` como **Data**, com o dia do evento e a coluna `data` original preservada. Se a origem vier com fuso horário, confirme a data de negócio antes de extrair o dia. Trate `dat_inicio` e `dat_termino` como datas. Padronize o tier em S, A, B, C e D nas consultas que o contêm. Em `Tier_Pontuacao`, divida `formato` pelo sublinhado em `FormatoRegra` e `ResultadoRegra` (ex. `BO3` e `win`). Em `Tier_Colocacao`, extraia o número antes de `º` como `PosicaoRegra`, inteira. Verifique que `tier + FormatoRegra + ResultadoRegra` e `tier + PosicaoRegra` são chaves únicas.

## 2. Validar e completar os vínculos

- [ ] **T04 — Conferir chaves antes de criar relações.** Use **Agrupar por → Contar linhas** para testar unicidade e filtre os nulos separadamente. Confira `Times[time_id]`, `Campeonatos[id_camp]`, `Partidas[match_id]`, o par `id_camp + placement_rank` das colocações e o par `time_id + campeonato` da pontuação. Use **Mesclar consultas como novas → Anti esquerda** para listar chaves sem cadastro. Verifique `Partidas[tournament_id]` e `Colocacoes_Campeonatos[id_camp]` contra campeonatos, e `team_a_id`, `team_b_id`, `winner_team_id` e `Pontuacao_Times[time_id]` contra times. Nos registros de partida, confira se A e B são equipes distintas e se o vencedor é um dos lados. Investigue duplicatas, órfãos e divergências antes de incluir os registros nos indicadores que dependem dessas chaves. [Tipos de junção no Power Query](https://learn.microsoft.com/en-us/power-query/merge-queries-overview)
- [ ] **T05 — Mapear os rótulos de evento.** Na amostra local, os 29 rótulos distintos de `Pontuacao_Times[campeonato]` têm zero correspondências exatas com `Campeonatos[nome_do_campeonato]`. Crie uma consulta auxiliar com `rotulo_origem`, `edicao_confirmada` e `evidencia`. Valide abreviações como `cmee_18` pela edição e pela data na origem. Resolva a edição cadastrada ao `id_camp` da carga atual e mescle por correspondência exata com **Externa esquerda**. Quando um rótulo for ambíguo, investigue se a origem oferece campos para resolver a edição. Mantenha a chave vazia enquanto a associação estiver pendente. Compare o número de linhas e as somas antes e depois de expandir a chave.
- [ ] **T06 — Mapear as equipes das colocações.** Compare `team_name` com `nome_da_equipe` após normalizar caixa e espaços, preservando os nomes originais. Dos 123 nomes distintos da amostra, 98 coincidem exatamente antes desse tratamento. Revise os demais com uma tabela de aliases e evidências. Se um alias representar equipes diferentes conforme edição ou ano, use esse contexto no mapeamento. Resolva a equipe cadastrada ao `time_id` atual, faça uma mesclagem externa esquerda e expanda a chave. Cada linha deve encontrar no máximo uma equipe. Registre quantas linhas e quantos nomes ficaram pendentes.
- [ ] **T07 — Criar `Partida_Equipe` com Referência e Acrescentar.** A partir de `Partidas`, crie duas consultas de referência, uma para cada lado. Preserve `match_id`, `tournament_id`, `DataPartida`, `match_format`, `stage` e `winner_team_id` em ambas. Renomeie as colunas conforme o quadro e acrescente as duas consultas em uma nova. Desabilite a carga das referências intermediárias. Use somente participações com dois IDs de equipe válidos e distintos. Classifique `ResultadoEquipe` como **Vitória** ou **Derrota** apenas quando o vencedor for um dos participantes. Use **Indefinido** quando não houver vencedor válido e **Empate** somente se houver evidência explícita de empate. Confira duas linhas por partida com participantes válidos e uma linha por par `match_id + time_id`.

| Campo em `Partida_Equipe` | Referência lado A | Referência lado B |
|---|---|---|
| `time_id` | `team_a_id` | `team_b_id` |
| `adversario_id` | `team_b_id` | `team_a_id` |
| `PontosEquipe` | `pts_time_a` | `pts_time_b` |
| `Lado` | A | B |

Para o detalhe da página de partidas, expanda o nome do adversário com uma mesclagem externa esquerda com `Times` pelo ID. Na amostra, o `match_id = 2` tem 0 ponto dos dois lados e vencedor com ID 2. Use esse caso para testar a classificação pelo vencedor.

## 3. Montar o modelo

- [ ] **T08 — Aplicar as consultas e configurar as relações.** Em **Fechar e aplicar**, carregue as tabelas finais. Na vista **Modelo**, confira as relações detectadas automaticamente. Use relações ativas de **um para muitos**, com filtro em uma direção, da primeira para a segunda coluna da matriz abaixo. Os IDs devem ter o mesmo tipo nas duas pontas. [Relações no Power BI Desktop](https://learn.microsoft.com/power-bi/transform-model/desktop-relationships-understand)

| Lado 1 | Lado muitos | Quando ligar |
|---|---|---|
| `Campeonatos[id_camp]` | `Partidas[tournament_id]` | Após T04 |
| `Campeonatos[id_camp]` | `Colocacoes_Campeonatos[id_camp]` | Após T04 |
| `Campeonatos[id_camp]` | `Partida_Equipe[tournament_id]` | Após T07 |
| `Times[time_id]` | `Pontuacao_Times[time_id]` | Após T04 |
| `Times[time_id]` | `Partida_Equipe[time_id]` | Após T07 |
| `Campeonatos[id_camp]` | `Pontuacao_Times[id_camp]` | Após T05, para linhas mapeadas |
| `Times[time_id]` | `Colocacoes_Campeonatos[time_id]` | Após T06, para linhas mapeadas |
| `CalendarioPartidas[Data]` | `Partidas[DataPartida]` e `Partida_Equipe[DataPartida]` | Uma relação por tabela após T09 |

Use `Partida_Equipe` como origem dos visuais que precisam responder à segmentação de equipe, formato, fase e lado. As tabelas de regras podem compartilhar uma dimensão `DimTierRegra` com uma linha para cada tier (S a D) para filtrar as duas matrizes. O tier das páginas de competição vem de `Campeonatos`. Compare com o tier registrado em `Partidas` se houver divergência.

- [ ] **T09 — Criar o calendário das partidas.** Gere uma tabela com datas contínuas entre a menor e a maior `DataPartida` válida. Use `Data` única e sem horário, crie ano, mês e ano-mês e ordene o rótulo pelo valor cronológico. Marque a tabela como calendário e crie as duas relações indicadas. Use filtros de período somente para registros de partidas. Para uma série de campeonatos iniciados, crie um calendário próprio ligado a `dat_inicio`. O ranking tem uma única fotografia da fonte e os pontos por rótulo têm o grão de evento.

## 4. Criar as medidas

- [ ] **T10 — Criar uma medida por definição abaixo.** Os exemplos usam vírgula como separador. Ajuste ao padrão do Desktop se ele estiver configurado para usar ponto e vírgula.

```dax
Equipes cadastradas = DISTINCTCOUNT(Times[time_id])
Campeonatos cadastrados = DISTINCTCOUNT(Campeonatos[id_camp])
Registros na origem = DISTINCTCOUNT(Partidas[match_id])
Registros com equipes válidas = DISTINCTCOUNT(Partida_Equipe[match_id])
Participações em partidas = COUNTROWS(Partida_Equipe)

Vitórias = CALCULATE(
    [Participações em partidas],
    KEEPFILTERS(Partida_Equipe[ResultadoEquipe] = "Vitória")
)
Derrotas = CALCULATE(
    [Participações em partidas],
    KEEPFILTERS(Partida_Equipe[ResultadoEquipe] = "Derrota")
)
Taxa de vitória da equipe = IF(
    HASONEVALUE(Times[time_id]),
    DIVIDE([Vitórias], [Vitórias] + [Derrotas])
)

Pontos de partidas por evento = SUM(Pontuacao_Times[pontos_partidas])
Pontos de colocação por evento = SUM(Pontuacao_Times[pontos_colocacao])
Pontos registrados em partidas = SUM(Partida_Equipe[PontosEquipe])
Colocações registradas = COUNTROWS(Colocacoes_Campeonatos)
Títulos registrados = CALCULATE(
    DISTINCTCOUNT(Colocacoes_Campeonatos[id_camp]),
    KEEPFILTERS(Colocacoes_Campeonatos[placement_rank] = 1)
)
Registros sem data = CALCULATE(
    [Registros na origem],
    KEEPFILTERS(ISBLANK(Partidas[DataPartida]))
)
```

Formate a taxa como percentual. Em cada linha de equipe, use o `ranking_atual_automatico` como **pontuação atual do ranking**, com as casas decimais da fonte. Para Top 10, use a pontuação como valor por equipe e ordene do maior para o menor. Se nomes de exibição se repetirem, use um rótulo que identifique cada equipe pelo ID. Exiba empates de pontuação conforme a fonte e valide o critério antes de publicar uma posição ordinal. Os pontos de partidas por evento e os pontos registrados por lado têm grãos distintos e devem ser lidos separadamente.

## 5. Construir as páginas

- [ ] **T11 — Ranking e contribuição de pontos.** Responde quem lidera a fotografia atual e quais eventos contribuíram para a pontuação de cada equipe.

| Visual | Campos e medidas |
|---|---|
| Cartão de cadastro | `[Equipes cadastradas]`, com indicação do contexto de filtro |
| Barras horizontais Top 10 | Nome da equipe no eixo e `ranking_atual_automatico` como valor por equipe. Filtre ranking vazio |
| Tabela de ranking | Equipe, ranking atual, ranking base e penalidade em percentual. Exiba por equipe e desabilite os totais |
| Matriz de contribuição | Equipe → rótulo de evento nas linhas, com as duas medidas de pontos por evento. Após T05, use o nome cadastrado nos registros mapeados e preserve a identificação dos pendentes |

Use segmentação de equipe na página toda e aplique o filtro de evento somente à matriz. Confira `fear_clan` na amostra local, com ranking 466,03 e penalidade de 71%. Troque o evento e confirme que apenas a contribuição por evento responde.

- [ ] **T12 — Partidas e desempenho.** Responde onde e quando ocorreram os registros e como uma equipe se saiu dos dois lados.

| Visual | Campos e medidas |
|---|---|
| Cartões | `[Registros com equipes válidas]` e, com uma equipe selecionada, `[Vitórias]`, `[Derrotas]` e `[Taxa de vitória da equipe]`. Para os cartões de vitórias e derrotas, use medidas de exibição com a mesma condição `HASONEVALUE(Times[time_id])` da taxa |
| Série mensal | Ano-mês de `CalendarioPartidas` × `[Registros com equipes válidas]`. Exclua o mês em branco somente desse visual |
| Barras ou matriz | Campeonato, `Partida_Equipe[match_format]` e `Partida_Equipe[stage]` × `[Registros com equipes válidas]` |
| Tabela de detalhe | `match_id`, data, campeonato, equipe, adversário, lado, formato, fase, resultado e pontos do lado. O detalhe tem uma linha por participação |

Use segmentações de equipe, campeonato, tier do campeonato, formato e período. Teste uma equipe que apareça como A e B e confirme que o cartão de partidas conta cada `match_id` uma vez. Com duas equipes selecionadas, um confronto entre elas aparece em duas linhas de detalhe e conta como um registro no cartão. Use **registros de partidas** nos títulos até confirmar na fonte se cada linha representa uma série ou outro nível de detalhe.

- [ ] **T13 — Campeonatos e colocações.** Responde quais eventos existem, qual o status e quem ficou em cada posição.

| Visual | Campos e medidas |
|---|---|
| Cartões | `[Campeonatos cadastrados]`, `[Colocações registradas]` e `[Títulos registrados]` somente após confirmar que a primeira posição representa título na origem |
| Barras | Tier e status de `Campeonatos` × `[Campeonatos cadastrados]` |
| Tabela de campeonatos | Nome, tier, status, início, término e link de referência |
| Tabela de classificação | Campeonato, `placement_rank`, `team_name` e, se mapeada, equipe cadastrada. Ordene pela posição numérica |

Use tier, status e campeonato para a página toda. Se houver filtro de equipe, aplique-o à classificação e aos cartões de colocações e títulos. Antes de T06, use o nome de origem somente nesses visuais. Na conferência, selecione um evento com pódio preenchido e valide manualmente as posições 1 a 3.

- [ ] **T14 — Regras e conferência.** Mostre uma matriz de tier × posição e outra de tier × formato/resultado, usando o valor de pontos por combinação única. Use `SELECTEDVALUE` para exibir o valor e desabilite subtotais e totais das matrizes. Deixe as células sem regra em branco, incluindo colocações do tier D. Confira S/1º = 100, S/BO3/win = 15 e S/BO3/loss = −15. Inclua os cartões de registros na origem e sem data, além da cobertura dos dois mapeamentos (linhas vinculadas ÷ linhas totais, por tabela). Aponte a competência da carga ou a data de referência da fonte quando disponível.

## 6. Validar antes de entregar

- [ ] **T15 — Reconciliar amostras e filtros.** Compare cartões, totais e pelo menos uma seleção por página com as tabelas do BigQuery. Para a amostra local, `Partidas` tem 525 registros (389 BO1, 135 BO3, 1 BO5), 2 sem data e 460 sem duração. Use esses valores como referência para a mesma carga. Confira contagens antes e depois dos mapeamentos e investigue divergências sem mascarar nulos. Teste filtros vazios, múltiplas seleções, zeros e negativos.
- [ ] **T16 — Conferir a atualização.** Atualize o relatório e valide tipos, chaves, quantidades e amostras. A carga usa `replace` por tabela e os IDs do pipeline podem mudar conforme a ordem dos dados. Atualize o Power BI depois da conclusão das sete cargas e teste se os mapas continuam apontando para as equipes e edições corretas. Prefira manter o mapa pelo nome/edição de cadastro ou por uma chave persistente da fonte e resolver os IDs na carga corrente.
- [ ] **T17 — Salvar o PBIX e as evidências.** Documente dataset, data da carga, regras dos indicadores e cobertura dos mapeamentos. Revise títulos, unidades e **Editar interações**. Salve uma captura de cada página e do diagrama do modelo para o portfólio.

Comece por T01–T04 e use os vínculos existentes para montar o ranking, a contribuição por rótulo e o catálogo com colocações. T07 libera a análise por equipe em partidas. T05 e T06 liberam os filtros integrados conforme a cobertura obtida.
