# Köppen–Geiger Brasil: reanálise ERA5 vs. Alvares et al. (2013)

Projeto acadêmico de reprodução e extensão do mapa de classificação climática de
Köppen–Geiger para o Brasil publicado em:

> ALVARES, C.A.; STAPE, J.L.; SENTELHAS, P.C.; GONÇALVES, J.L.M.; SPAROVEK, G.
> **Köppen's climate classification map for Brazil.** *Meteorologische Zeitschrift*,
> v. 22, n. 6, p. 711–728, 2013.

O artigo original classifica o território brasileiro em 1 ha de resolução a partir de
2.950 estações meteorológicas (krigagem de precipitação + regressão de temperatura por
altitude/SRTM), normais climatológicas de 1950–1990. Este projeto testa se é possível
reproduzir — e até atualizar — esse mapa usando **dados abertos modernos** (reanálise
ERA5 e estações do INMET), aplicando exatamente o mesmo algoritmo de decisão descrito
no artigo (idêntico a Kottek et al. 2006 / Peel et al. 2007).

## Pergunta de pesquisa

A divergência entre o mapa resultante e o artigo original é causada por **viés de
fonte de dado** (reanálise de modelo vs. estação observada) ou por **mudança
climática real** entre a linha de base 1950–1990 e o período atual? O projeto isola
essas duas variáveis rodando o mesmo algoritmo sobre quatro combinações diferentes de
fonte/período de dado.

## Estrutura do repositório

```
├── notebook.ipynb                         # 1. ERA5, recorte parcial (GO/MS/MG/SP/PR/BA/TO), 1995–2025
├── notebook_brasil.ipynb                  # 2. ERA5, Brasil inteiro (IBGE), 1996–2025
├── notebook_estacoes.ipynb                # 3. Estações automáticas INMET (BDMEP), regressão+krigagem, ~2000–2025
├── notebook_1950_1990.ipynb               # 4. ERA5, Brasil inteiro, MESMA janela do artigo (1950–1990)
│
├── documentacao_koppen_era5.pdf           # Relatório técnico do notebook 1 (metodologia, tecnologias, resultados)
├── documentacao_koppen_era5_1950_1990.pdf # Relatório técnico do notebook 4 (comparação direta com o artigo)
│
├── data/
│   ├── brasil_ibge.geojson                # Malha oficial do Brasil (API do IBGE) — recorte geográfico
│   ├── brazil_states.geojson              # Contorno dos estados (sobreposição nos mapas)
│   ├── etopo2022_60s_brasil.nc            # Modelo de elevação ETOPO 2022 (NOAA/ERDDAP), 1 arc-min
│   ├── bdmep_estacoes/                    # 653 CSVs — rede automática INMET (BDMEP)
│   └── bdmep_convencional_1950_1990/      # 646 CSVs — rede convencional INMET (BDMEP), 1950–1990
│
├── *.grib                                  # Dados brutos ERA5 (Copernicus CDS)
├── cds_request.py                          # Script de exemplo do request à API do Copernicus CDS
└── requirements.txt
```

## Metodologia (resumo)

1. **Carregamento**: GRIB lido via `earthkit-data` → `xarray`; conversão de unidades
   (Kelvin→°C; taxa de precipitação m/dia→mm/mês).
2. **Recorte geográfico**: máscara ponto-em-polígono (`shapely.contains_xy`) contra a
   malha oficial do IBGE — necessário porque a bbox do ERA5 inclui países vizinhos e
   oceano.
3. **Climatologia mensal**: média por mês-calendário sobre todos os anos disponíveis,
   por célula de grade.
4. **Classificação de Köppen–Geiger**: implementação vetorizada em NumPy da Tabela 1
   do artigo (idêntica a Kottek et al. 2006 / Peel et al. 2007), verificada contra 15
   perfis climáticos sintéticos em todos os notebooks. **A função é copiada sem
   nenhuma alteração entre os quatro notebooks** — apenas o dado de entrada muda.
5. **Correção de viés** (`notebook_1950_1990.ipynb`): regression-kriging do erro
   `ERA5 − estação` usando 405 estações convencionais do INMET, aplicado como
   correção espacial sobre a grade do ERA5.

## Principais achados

- **A classificação por zona (A/B/C) bate de forma consistente com o artigo** em
  todas as quatro combinações testadas (81–82% de zona A em todas).
- **A zona B (árido/semiárido) mostra uma progressão temporal quase monotônica**:
  4,9% (artigo, 1950–1990) → 6,1% (ERA5 na mesma janela) → ~8% (dado moderno,
  ERA5 e estações) — evidência de que parte da divergência reflete mudança real,
  não apenas viés de fonte de dado.
- **A maior divergência específica (tipos Am↔Aw) está concentrada numa faixa
  geográfica estreita** (transição Amazônia–Cerrado), explicada por um vazio de
  cobertura de estação de até ~400 km de raio exatamente ali — não por um viés
  difuso do ERA5.
- Duas estratégias de correção foram testadas e documentadas com resultado honesto
  (uma com ganho líquido pequeno e misto; outra descartada por sinal estatístico
  insuficiente) — ver `notebook_1950_1990.ipynb`, seções 7–9.

Detalhes completos, com gráficos e tabelas, nos dois PDFs de documentação.

## Fontes de dados

| Fonte | Uso | Licença/acesso |
|---|---|---|
| [ERA5 (Copernicus CDS)](https://cds.climate.copernicus.eu/) | Reanálise de temperatura e precipitação | Copernicus License |
| [IBGE — Malhas Geográficas](https://servicodados.ibge.gov.br/api/docs/malhas) | Fronteira oficial do Brasil | Dados abertos |
| [INMET — BDMEP](https://bdmep.inmet.gov.br/) | Estações meteorológicas (automáticas e convencionais) | Dados abertos |
| [ETOPO 2022 (NOAA)](https://www.ncei.noaa.gov/products/etopo-global-relief-model) | Modelo de elevação | Domínio público |
| [click_that_hood](https://github.com/codeforamerica/click_that_hood) | GeoJSON de estados brasileiros | MIT |

## Como reproduzir

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab
```

Os `.grib` e os CSVs de estação já estão incluídos no repositório — não é necessário
refazer os downloads para rodar os notebooks. O script `cds_request.py` documenta os
parâmetros exatos usados para baixar os dados ERA5 via API do Copernicus CDS, caso
queira atualizar o período ou a resolução.

## Limitações conhecidas

- Resolução espacial (0,1°–0,25° conforme o notebook) muito mais grosseira que o 1 ha
  do artigo original — relevo local (picos isolados) não é resolvido.
- ERA5 tem viés seco de precipitação documentado (~12–38% nas localidades de
  referência testadas), mais pronunciado nas décadas de 1950–60 por menor densidade
  de observações assimiladas.
- Correções de viés testadas usam os mesmos pontos de estação para calibração e
  avaliação (sem separação treino/teste) — ver ressalvas detalhadas em cada notebook.

Ressalvas completas, com números, estão na última seção de cada notebook e nos PDFs.

## Autor

Projeto desenvolvido como exploração acadêmica de climatologia computacional e
geoestatística aplicada ao território brasileiro.
