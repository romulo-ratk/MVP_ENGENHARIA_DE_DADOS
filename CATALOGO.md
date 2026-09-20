# Catálogo de Dados

Transcrição do catálogo de dados do MVP de Engenharia de Dados — Mercado de Banda
Larga Fixa nos Municípios do RS.

O catálogo é implementado diretamente no **Unity Catalog** do Databricks, via
`COMMENT ON TABLE` e `COMMENT ON COLUMN` aplicados programaticamente nos notebooks
das camadas Silver e Gold. Este documento reproduz o conteúdo desses comentários.

Para cada campo estão documentados os quatro itens exigidos pelo enunciado:
**descrição**, **tipo de dado**, **domínio de valores** e **linhagem**.

---

## Convenção de nomes

A unidade de medida faz parte do nome de cada coluna:

| Sufixo | Unidade |
|---|---|
| `_qtd` | contagem de unidades (acessos, domicílios, prestadoras, municípios) |
| `_hab` | habitantes |
| `_pct` | percentual, escala 0 a 100 |
| `_brl` | reais |
| `_mil_brl` | mil reais — unidade original das séries do IBGE |
| `_km2` | quilômetros quadrados |
| `sk_` | prefixo de chave surrogate |
| `e_` | prefixo de flag booleana |
| `_` | prefixo de coluna de controle do pipeline |

---

## Índice

**Camada Bronze**
- [`bronze.telecom.anatel_acessos_raw`](#bronzetelecomanatel_acessos_raw)
- [`bronze.telecom.anatel_total_raw`](#bronzetelecomanatel_total_raw)
- [`bronze.telecom.anatel_densidade_raw`](#bronzetelecomanatel_densidade_raw)
- [`bronze.telecom.ibge_municipios_raw`](#bronzetelecomibge_municipios_raw)
- [`bronze.telecom.ibge_populacao_raw`](#bronzetelecomibge_populacao_raw)
- [`bronze.telecom.ibge_domicilios_raw`](#bronzetelecomibge_domicilios_raw)
- [`bronze.telecom.ibge_domicilios_especie_raw`](#bronzetelecomibge_domicilios_especie_raw)
- [`bronze.telecom.ibge_pib_raw`](#bronzetelecomibge_pib_raw)

**Camada Silver**
- [`silver.telecom.acessos`](#silvertelecomacessos)
- [`silver.telecom.empresas`](#silvertelecomempresas)
- [`silver.telecom.tecnologias`](#silvertelecomtecnologias)
- [`silver.telecom.municipios`](#silvertelecommunicipios)
- [`silver.telecom.qualidade_dados`](#silvertelecomqualidade_dados)

**Camada Gold — Dimensões**
- [`gold.telecom.dim_municipio`](#goldtelecomdim_municipio)
- [`gold.telecom.dim_empresa`](#goldtelecomdim_empresa)
- [`gold.telecom.dim_tecnologia`](#goldtelecomdim_tecnologia)
- [`gold.telecom.dim_faixa_velocidade`](#goldtelecomdim_faixa_velocidade)
- [`gold.telecom.dim_segmento`](#goldtelecomdim_segmento)

**Camada Gold — Fato e tabelas analíticas**
- [`gold.telecom.fato_acessos`](#goldtelecomfato_acessos)
- [`gold.telecom.mercado_municipio`](#goldtelecommercado_municipio)
- [`gold.telecom.prestadora_atuacao`](#goldtelecomprestadora_atuacao)
- [`gold.telecom.indice_oportunidade`](#goldtelecomindice_oportunidade)

---
---

# Camada Bronze

**Princípio da camada:** preservar o dado exatamente como recebido da fonte, sem
filtro de conteúdo nem conversão de tipos. **Todas as colunas são do tipo `STRING`**,
inclusive as numéricas — a tipagem é responsabilidade da camada Silver.

A única alteração técnica é a normalização dos nomes de coluna para snake_case,
exigida pelo formato Delta, que rejeita espaços e acentos em nomes de coluna sem
habilitar column mapping.

## Colunas de controle

Presentes em todas as tabelas da camada:

| Coluna | Tipo | Descrição |
|---|---|---|
| `_arquivo_origem` | STRING | Nome do arquivo CSV ou URL da requisição de origem |
| `_fonte` | STRING | Identificação da fonte e da tabela de origem |
| `_data_ingestao` | STRING | Timestamp UTC da ingestão, formato ISO 8601 |

---

## `bronze.telecom.anatel_acessos_raw`

Acessos de banda larga fixa declarados pelas prestadoras de SCM à Anatel.
Abrangência nacional, ano de 2026, granularidade mensal.

**Grão:** ano × mês × grupo econômico × empresa × CNPJ × porte × UF × município ×
faixa de velocidade × velocidade × tecnologia × meio de acesso × tipo de pessoa ×
tipo de produto

**Fonte:** `https://www.anatel.gov.br/dadosabertos/paineis_de_dados/acessos/acessos_banda_larga_fixa.zip`

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `ano` | STRING | Ano de referência. Domínio: `2026` |
| `mes` | STRING | Mês de referência, sem zero à esquerda. Domínio: `1` a `12` |
| `grupo_economico` | STRING | Grupo econômico declarado. Domínio: 16 categorias no RS. `OUTROS` agrega todos os provedores regionais |
| `empresa` | STRING | Nome comercial da prestadora |
| `cnpj` | STRING | CNPJ da prestadora, 14 dígitos sem formatação |
| `porte_da_prestadora` | STRING | Domínio: `Pequeno Porte`, `Grande Porte` |
| `uf` | STRING | Sigla da unidade federativa. Domínio: 27 UFs |
| `municipio` | STRING | Nome do município |
| `codigo_ibge_municipio` | STRING | Código IBGE do município, 7 dígitos |
| `faixa_de_velocidade` | STRING | Domínio: `0Kbps a 512Kbps`, `512kbps a 2Mbps`, `2Mbps a 12Mbps`, `12Mbps a 34Mbps`, `> 34Mbps` |
| `velocidade` | STRING | Velocidade exata contratada, em Mbps, separador decimal vírgula. 1.135 valores distintos no RS |
| `tecnologia` | STRING | Domínio: 24 categorias — `FTTH`, `ETHERNET`, `Wi-Fi`, `VSAT`, `HFC`, `FTTB`, `ADSL2`, `ADSL1`, `HDSL`, `DWDM`, `OFDMA/TDD`, `TDMA`, `NR`, `FR`, `WIMAX`, `FWA`, `VDSL`, `ATM`, `DTH`, `LTE`, `SDH`, `Cable Modem`, `MMDS`, `PDH` |
| `meio_de_acesso` | STRING | Domínio: `Fibra`, `Rádio`, `Satélite`, `Cabo Metálico`, `Cabo Coaxial` |
| `tipo_de_pessoa` | STRING | Domínio: `Pessoa Física`, `Pessoa Jurídica` |
| `tipo_de_produto` | STRING | Domínio: `INTERNET`, `LINHA_DEDICADA`, `M2M`, `OUTROS` |
| `acessos` | STRING | Quantidade de acessos ativos. Domínio: inteiro positivo |

---

## `bronze.telecom.anatel_total_raw`

Total nacional de acessos de banda larga fixa por mês, publicado pela Anatel.
Referência independente para validar a integridade da ingestão de
`anatel_acessos_raw`.

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `ano` | STRING | Ano de referência |
| `mes` | STRING | Mês de referência, sem zero à esquerda |
| `acessos` | STRING | Total nacional de acessos no mês. Ordem de grandeza: dezenas de milhões |

---

## `bronze.telecom.anatel_densidade_raw`

Densidade de acessos de banda larga fixa calculada pela Anatel, por nível
geográfico. Referência para validar a metodologia de penetração da camada Gold.

**Atenção:** o separador decimal é vírgula.

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `ano` | STRING | Ano de referência |
| `mes` | STRING | Mês de referência |
| `uf` | STRING | Sigla da UF, ou `Brasil` no nível nacional |
| `municipio` | STRING | Nome do município, ou sigla da UF, ou `Brasil`, conforme o nível |
| `codigo_ibge` | STRING | Código IBGE de 7 dígitos, ou `0000000` no nível nacional |
| `densidade` | STRING | Acessos por 100 habitantes. Separador decimal vírgula |
| `nivel_geografico_densidade` | STRING | Domínio: `Brasil`, `UF`, `Municipio` |

---

## `bronze.telecom.ibge_municipios_raw`

Municípios do RS com código IBGE de 7 dígitos e hierarquia geográfica completa.

**Fonte:** `https://servicodados.ibge.gov.br/api/v1/localidades/estados/RS/municipios`

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `codigo_ibge` | STRING | Código IBGE do município, 7 dígitos. Domínio: `4300034` a `4323804` |
| `nome_municipio` | STRING | Nome oficial do município |
| `microrregiao_id` | STRING | Código da microrregião geográfica |
| `microrregiao_nome` | STRING | Nome da microrregião geográfica |
| `mesorregiao_id` | STRING | Código da mesorregião geográfica |
| `mesorregiao_nome` | STRING | Nome da mesorregião. Domínio: 7 categorias no RS |
| `regiao_imediata_id` | STRING | Código da região geográfica imediata, divisão vigente |
| `regiao_imediata_nome` | STRING | Nome da região geográfica imediata |
| `regiao_intermediaria_id` | STRING | Código da região geográfica intermediária |
| `regiao_intermediaria_nome` | STRING | Nome da região geográfica intermediária |
| `uf_sigla` | STRING | Sigla da UF. Domínio: `RS` |
| `uf_nome` | STRING | Nome da UF. Domínio: `Rio Grande do Sul` |

---

## `bronze.telecom.ibge_populacao_raw`

População residente, área territorial e densidade demográfica por município do RS.
Censo Demográfico 2022.

**Fonte:** API SIDRA, tabela 4714, variáveis 93, 6318 e 614
**Formato:** longo — uma linha por município por variável (497 × 3 = 1.491 linhas)

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `nivel_territorial_codigo` | STRING | Código do nível territorial. Domínio: `6` (município) |
| `nivel_territorial` | STRING | Domínio: `Município` |
| `unidade_de_medida_codigo` | STRING | Código da unidade de medida da variável |
| `unidade_de_medida` | STRING | Domínio: `Pessoas`, `Quilômetros quadrados`, `Habitante por quilômetro quadrado` |
| `valor` | STRING | Valor da variável. Pode conter os códigos especiais `-`, `..`, `...`, `X` |
| `municipio_codigo` | STRING | Código IBGE do município, 7 dígitos |
| `municipio` | STRING | Nome do município seguido da UF |
| `variavel_codigo` | STRING | Domínio: `93` (população), `6318` (área), `614` (densidade) |
| `variavel` | STRING | Nome descritivo da variável |
| `ano_codigo` | STRING | Domínio: `2022` |
| `ano` | STRING | Domínio: `2022` |

---

## `bronze.telecom.ibge_domicilios_raw`

Domicílios particulares permanentes ocupados, moradores e média de moradores por
município do RS. Censo Demográfico 2022.

**Fonte:** API SIDRA, tabela 4712, variáveis 381, 382 e 5930
**Formato:** longo — 497 × 3 = 1.491 linhas

Mesma estrutura de colunas de `ibge_populacao_raw`, com:

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `variavel_codigo` | STRING | Domínio: `381` (domicílios ocupados), `382` (moradores), `5930` (média de moradores) |
| `unidade_de_medida` | STRING | Domínio: `Domicílios`, `Pessoas`, `Pessoas` |

---

## `bronze.telecom.ibge_domicilios_especie_raw`

Domicílios recenseados por espécie, por município do RS. Censo Demográfico 2022.
É a fonte do denominador de mercado endereçável.

**Fonte:** API SIDRA, tabela 4711, variável 617, classificação 3 (Espécie)
**Formato:** longo — 497 × 5 = 2.485 linhas

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `nivel_territorial_codigo` | STRING | Domínio: `6` |
| `nivel_territorial` | STRING | Domínio: `Município` |
| `unidade_de_medida_codigo` | STRING | Código da unidade de medida |
| `unidade_de_medida` | STRING | Domínio: `Domicílios` |
| `valor` | STRING | Quantidade de domicílios. Pode conter códigos especiais |
| `municipio_codigo` | STRING | Código IBGE do município, 7 dígitos |
| `municipio` | STRING | Nome do município seguido da UF |
| `variavel_codigo` | STRING | Domínio: `617` |
| `variavel` | STRING | Domínio: `Domicílios recenseados` |
| `ano_codigo` | STRING | Domínio: `2022` |
| `ano` | STRING | Domínio: `2022` |
| `especie_codigo` | STRING | Domínio: `59993` (total), `59998` (particular permanente ocupado), `60001` (não ocupado), `60002` (não ocupado vago), `60003` (não ocupado de uso ocasional) |
| `especie` | STRING | Nome descritivo da espécie de domicílio |

---

## `bronze.telecom.ibge_pib_raw`

PIB, impostos e Valor Adicionado Bruto por setor, por município do RS.

**Fonte:** API SIDRA, tabela 5938, variáveis 37, 543, 498, 513, 517, 6575 e 525
**Ano de referência:** 2021 — o mais recente com desdobramento setorial completo
**Formato:** longo — 497 × 7 = 3.479 linhas

Mesma estrutura de colunas de `ibge_populacao_raw`, com:

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `variavel_codigo` | STRING | Domínio: `37` (PIB total), `543` (impostos líquidos), `498` (VAB total), `513` (VAB agropecuária), `517` (VAB indústria), `6575` (VAB serviços), `525` (VAB administração pública) |
| `unidade_de_medida` | STRING | Domínio: `Mil Reais` |
| `ano` | STRING | Domínio: `2021` |

> **Observação de qualidade de dados.** Em 2022 e 2023 a SIDRA publica apenas a
> variável 37; as seis restantes retornam `...` (dado não disponível) em todos os
> municípios. Por isso o ano de referência foi fixado em 2021.

---
---

# Camada Silver

**Princípio da camada:** dado limpo, tipado, deduplicado e no recorte do MVP —
Rio Grande do Sul, julho de 2026.

## Coluna de controle

Presente em todas as tabelas da camada:

| Coluna | Tipo | Descrição |
|---|---|---|
| `_data_processamento` | STRING | Timestamp UTC do processamento, formato ISO 8601 |

---

## `silver.telecom.acessos`

Acessos de banda larga fixa no RS, período de referência único.

A coluna `velocidade` exata foi descartada por cardinalidade (1.135 valores
distintos) sem ganho analítico. O `meio_acesso` permanece no grão porque não é
determinado funcionalmente pela tecnologia. Nenhum tipo de acesso é descartado.

**Grão:** `periodo` × `codigo_ibge` × `cnpj` × `tecnologia` × `meio_acesso` ×
`faixa_velocidade` × `tipo_pessoa` × `tipo_produto`

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `periodo` | STRING | Período de referência, formato AAAA-MM. Domínio: `2026-07`. Linhagem: colunas `ano` e `mes` de `bronze.anatel_acessos_raw` |
| `codigo_ibge` | STRING | Código IBGE do município, 7 dígitos. Domínio: `4300034` a `4323804`. Linhagem: campo `Código IBGE Município` da Anatel |
| `cnpj` | STRING | CNPJ da prestadora, 14 dígitos. Chave de identidade competitiva. Linhagem: campo `CNPJ` da Anatel |
| `tecnologia` | STRING | Tecnologia de acesso declarada. Domínio: 24 categorias. Linhagem: campo `Tecnologia` da Anatel |
| `meio_acesso` | STRING | Meio físico de transmissão. Domínio: `Fibra`, `Rádio`, `Satélite`, `Cabo Metálico`, `Cabo Coaxial`. Não é determinado pela tecnologia. Linhagem: campo `Meio de Acesso` da Anatel |
| `faixa_velocidade` | STRING | Faixa de velocidade contratada. Domínio: 5 categorias. Linhagem: campo `Faixa de Velocidade` da Anatel |
| `tipo_pessoa` | STRING | Natureza do assinante. Domínio: `Pessoa Física`, `Pessoa Jurídica`. Linhagem: campo `Tipo de Pessoa` da Anatel |
| `tipo_produto` | STRING | Tipo de produto contratado. Domínio: `INTERNET`, `LINHA_DEDICADA`, `M2M`, `OUTROS`. Linhagem: campo `Tipo de Produto` da Anatel |
| `acessos_qtd` | INT | Quantidade de acessos ativos. Unidade: acessos. Medida aditiva. Domínio: inteiro ≥ 1. Linhagem: soma do campo `Acessos` da Anatel, agregado pelo grão desta tabela |

---

## `silver.telecom.empresas`

Prestadoras de SCM com atuação no RS no período de referência.

O campo `grupo_economico` traz `OUTROS` para todos os provedores regionais, o que o
inviabiliza como chave de análise competitiva.

**Chave primária:** `cnpj`

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `cnpj` | STRING | CNPJ da prestadora, 14 dígitos sem formatação. Chave primária. Linhagem: campo `CNPJ` da Anatel |
| `empresa` | STRING | Nome comercial da prestadora. Determinado funcionalmente pelo CNPJ |
| `grupo_economico` | STRING | Grupo econômico declarado. Domínio: 16 categorias no RS. `OUTROS` agrega todos os provedores regionais, 55% das linhas do estado |
| `porte` | STRING | Porte segundo a Anatel. Domínio: `Pequeno Porte`, `Grande Porte` |
| `grupo_identificado` | BOOLEAN | Indica grupo diferente de `OUTROS`. Domínio: `true`, `false`. Linhagem: derivado de `grupo_economico` |

---

## `silver.telecom.tecnologias`

Combinações de tecnologia e meio físico presentes no RS.

A tecnologia **não** determina funcionalmente o meio de acesso: `ETHERNET`, por
exemplo, aparece sobre fibra e sobre cabo metálico.

**Chave primária composta:** `tecnologia` + `meio_acesso`

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `tecnologia` | STRING | Tecnologia de acesso declarada. Parte da chave composta. Domínio: 24 categorias. Linhagem: campo `Tecnologia` da Anatel |
| `meio_acesso` | STRING | Meio físico de transmissão. Parte da chave composta. Domínio: `Fibra`, `Rádio`, `Satélite`, `Cabo Metálico`, `Cabo Coaxial` |
| `e_fibra` | BOOLEAN | Indica meio de acesso igual a `Fibra`. Domínio: `true`, `false`. Linhagem: derivado de `meio_acesso` |

---

## `silver.telecom.municipios`

Municípios do RS consolidando cinco fontes do IBGE. Traz **dois denominadores de
mercado**: `domicilios_ocupados_qtd` (padrão IBGE) e `domicilios_total_qtd`
(universo recenseado, inclui uso ocasional e vagos).

**Chave primária:** `codigo_ibge`

### Identificação e hierarquia geográfica

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `codigo_ibge` | STRING | Código IBGE do município, 7 dígitos. Chave primária. Domínio: `4300034` a `4323804`. Linhagem: IBGE API Localidades |
| `nome_municipio` | STRING | Nome oficial do município. Linhagem: IBGE API Localidades |
| `microrregiao_nome` | STRING | Microrregião geográfica do IBGE. Linhagem: API Localidades |
| `mesorregiao_nome` | STRING | Mesorregião geográfica do IBGE. Domínio: 7 categorias no RS. Linhagem: API Localidades |
| `regiao_imediata_nome` | STRING | Região geográfica imediata, divisão vigente. Linhagem: API Localidades |
| `regiao_intermediaria_nome` | STRING | Região geográfica intermediária do IBGE. Linhagem: API Localidades |
| `uf_sigla` | STRING | Sigla da unidade federativa. Domínio: `RS` |

### População e território

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `populacao_residente_hab` | DOUBLE | População residente. Unidade: habitantes. Linhagem: SIDRA 4714 v/93, Censo 2022 |
| `area_km2` | DOUBLE | Área da unidade territorial. Unidade: km². Linhagem: SIDRA 4714 v/6318 |
| `densidade_demografica_hab_km2` | DOUBLE | Densidade demográfica. Unidade: habitantes por km². Linhagem: SIDRA 4714 v/614 |
| `porte_populacional` | STRING | Faixa de porte populacional. Domínio: `Ate 5 mil`, `5 a 20 mil`, `20 a 100 mil`, `Acima de 100 mil`. Linhagem: derivado de `populacao_residente_hab` |

### Domicílios

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `domicilios_ocupados_qtd` | DOUBLE | Domicílios particulares permanentes ocupados. Unidade: domicílios. Denominador conservador. Não inclui imóveis de uso ocasional nem vagos. Linhagem: SIDRA 4712 v/381 |
| `moradores_em_domicilios_qtd` | DOUBLE | Moradores em domicílios particulares permanentes ocupados. Unidade: pessoas. Linhagem: SIDRA 4712 v/382 |
| `media_moradores_domicilio` | DOUBLE | Média de moradores por domicílio. Unidade: moradores por domicílio. Linhagem: SIDRA 4712 v/5930 |
| `domicilios_total_qtd` | DOUBLE | Total de domicílios recenseados, todas as espécies. Unidade: domicílios. Denominador de mercado endereçável: qualquer imóvel é potencial contratante. Linhagem: SIDRA 4711 v/617 c3/59993 |
| `domicilios_ocupados_esp_qtd` | DOUBLE | Domicílios particulares permanentes ocupados segundo a tabela 4711. Unidade: domicílios. Usado para validação cruzada com `domicilios_ocupados_qtd`. Linhagem: SIDRA 4711 c3/59998 |
| `domicilios_nao_ocupados_qtd` | DOUBLE | Domicílios permanentes não ocupados, soma de vagos e uso ocasional. Unidade: domicílios. Linhagem: SIDRA 4711 c3/60001 |
| `domicilios_vagos_qtd` | DOUBLE | Domicílios não ocupados classificados como vagos. Unidade: domicílios. Baixa probabilidade de contrato ativo. Linhagem: SIDRA 4711 c3/60002 |
| `domicilios_uso_ocasional_qtd` | DOUBLE | Domicílios não ocupados de uso ocasional, típicos de veraneio e turismo. Unidade: domicílios. Alta probabilidade de contrato ativo mesmo sem morador permanente. Linhagem: SIDRA 4711 c3/60003 |
| `uso_ocasional_pct` | DOUBLE | Participação de imóveis de uso ocasional no total recenseado. Unidade: percentual. Domínio: 0 a 100. Linhagem: `domicilios_uso_ocasional_qtd` ÷ `domicilios_total_qtd` × 100 |
| `perfil_ocupacao` | STRING | Classificação pelo peso do uso ocasional. Domínio: `Ocupacao permanente` (abaixo de 10%), `Ocupacao mista` (10 a 29%), `Veraneio ou turismo` (30% ou mais) |

### Economia

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `pib_total_mil_brl` | DOUBLE | PIB a preços correntes. Unidade: mil reais. Linhagem: SIDRA 5938 v/37, ano 2021 |
| `impostos_liquidos_mil_brl` | DOUBLE | Impostos líquidos de subsídios sobre produtos. Unidade: mil reais. Linhagem: SIDRA 5938 v/543 |
| `vab_total_mil_brl` | DOUBLE | Valor adicionado bruto total. Unidade: mil reais. Identidade contábil: `pib_total = vab_total + impostos_liquidos`. Linhagem: SIDRA 5938 v/498 |
| `vab_agropecuaria_mil_brl` | DOUBLE | VAB da agropecuária. Unidade: mil reais. Linhagem: SIDRA 5938 v/513 |
| `vab_industria_mil_brl` | DOUBLE | VAB da indústria. Unidade: mil reais. Linhagem: SIDRA 5938 v/517 |
| `vab_servicos_mil_brl` | DOUBLE | VAB dos serviços, exclusive administração pública. Unidade: mil reais. Linhagem: SIDRA 5938 v/6575 |
| `vab_administracao_mil_brl` | DOUBLE | VAB da administração, defesa, educação e saúde públicas. Unidade: mil reais. Linhagem: SIDRA 5938 v/525 |
| `setor_dominante` | STRING | Setor econômico com maior VAB no município. Domínio: `Agropecuaria`, `Industria`, `Servicos`, `Administracao publica`. Linhagem: maior entre os quatro VAB setoriais, com VAB nulo tratado como zero. Nulo quando o ano de referência do PIB não publica desdobramento setorial |
| `pib_per_capita_brl` | DOUBLE | PIB por habitante. Unidade: reais. Linhagem: `pib_total_mil_brl` × 1000 ÷ `populacao_residente_hab` |

---

## `silver.telecom.qualidade_dados`

Registro consolidado das verificações de qualidade executadas sobre as tabelas
Silver, nas cinco dimensões exigidas pelo enunciado.

| Coluna | Tipo | Descrição e domínio |
|---|---|---|
| `dimensao` | STRING | Dimensão de qualidade avaliada. Domínio: `Completude`, `Consistencia`, `Unicidade`, `Acuracia`, `Outliers` |
| `verificacao` | STRING | Descrição da verificação executada |
| `resultado` | STRING | Resultado observado, como texto |
| `unidade` | STRING | Unidade do resultado. Domínio: `linhas`, `municipios`, `booleano`, `pontos de densidade` |

---
---

# Camada Gold

**Princípio da camada:** dado modelado em esquema estrela e agregado para responder
às perguntas de negócio.

## Coluna de controle

Presente em todas as tabelas da camada:

| Coluna | Tipo | Descrição |
|---|---|---|
| `_data_processamento` | STRING | Timestamp UTC do processamento, formato ISO 8601 |

---

## `gold.telecom.dim_municipio`

Dimensão de municípios do RS. Hierarquia geográfica do IBGE e atributos
socioeconômicos do Censo 2022 e do PIB Municipal.

**Chave surrogate:** `sk_municipio` · **Chave natural:** `codigo_ibge`
**Cardinalidade:** 497 linhas

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_municipio` | INT | Chave surrogate. Inteiro sequencial determinístico gerado pela ordenação do código IBGE. Domínio: 1 a 497 |
| `codigo_ibge` | STRING | Código IBGE do município, 7 dígitos. Chave natural. Domínio: `4300034` a `4323804`. Linhagem: IBGE API Localidades |
| `nome_municipio` | STRING | Nome oficial do município |
| `microrregiao_nome` | STRING | Microrregião geográfica do IBGE |
| `mesorregiao_nome` | STRING | Mesorregião geográfica. Domínio: 7 categorias no RS |
| `regiao_imediata_nome` | STRING | Região geográfica imediata, divisão vigente |
| `regiao_intermediaria_nome` | STRING | Região geográfica intermediária do IBGE |
| `uf_sigla` | STRING | Sigla da unidade federativa. Domínio: `RS` |
| `porte_populacional` | STRING | Faixa de porte. Domínio: `Ate 5 mil`, `5 a 20 mil`, `20 a 100 mil`, `Acima de 100 mil` |
| `populacao_residente_hab` | DOUBLE | População residente. Unidade: habitantes. Linhagem: SIDRA 4714 v/93, Censo 2022 |
| `area_km2` | DOUBLE | Área territorial. Unidade: km². Linhagem: SIDRA 4714 v/6318 |
| `densidade_demografica_hab_km2` | DOUBLE | Densidade demográfica. Unidade: habitantes por km². Linhagem: SIDRA 4714 v/614 |
| `domicilios_total_qtd` | DOUBLE | Total de domicílios recenseados, todas as espécies. Unidade: domicílios. Denominador de mercado endereçável. Linhagem: SIDRA 4711 v/617 c3/59993 |
| `domicilios_ocupados_qtd` | DOUBLE | Domicílios particulares permanentes ocupados. Unidade: domicílios. Denominador conservador. Linhagem: SIDRA 4712 v/381 |
| `domicilios_nao_ocupados_qtd` | DOUBLE | Domicílios permanentes não ocupados. Unidade: domicílios. Linhagem: SIDRA 4711 c3/60001 |
| `domicilios_vagos_qtd` | DOUBLE | Domicílios não ocupados vagos. Unidade: domicílios. Baixa probabilidade de contrato ativo. Linhagem: SIDRA 4711 c3/60002 |
| `domicilios_uso_ocasional_qtd` | DOUBLE | Domicílios de uso ocasional, típicos de veraneio. Unidade: domicílios. Alta probabilidade de contrato ativo. Linhagem: SIDRA 4711 c3/60003 |
| `uso_ocasional_pct` | DOUBLE | Participação de imóveis de uso ocasional no total. Unidade: percentual. Domínio: 0 a 100 |
| `perfil_ocupacao` | STRING | Classificação pelo peso do uso ocasional. Domínio: `Ocupacao permanente`, `Ocupacao mista`, `Veraneio ou turismo` |
| `moradores_em_domicilios_qtd` | DOUBLE | Moradores em domicílios particulares permanentes ocupados. Unidade: pessoas. Linhagem: SIDRA 4712 v/382 |
| `media_moradores_domicilio` | DOUBLE | Média de moradores por domicílio. Unidade: moradores por domicílio. Linhagem: SIDRA 4712 v/5930 |
| `pib_total_mil_brl` | DOUBLE | PIB a preços correntes. Unidade: mil reais. Linhagem: SIDRA 5938 v/37 |
| `pib_per_capita_brl` | DOUBLE | PIB por habitante. Unidade: reais |
| `vab_total_mil_brl` | DOUBLE | Valor adicionado bruto total. Unidade: mil reais |
| `vab_agropecuaria_mil_brl` | DOUBLE | VAB da agropecuária. Unidade: mil reais |
| `vab_industria_mil_brl` | DOUBLE | VAB da indústria. Unidade: mil reais |
| `vab_servicos_mil_brl` | DOUBLE | VAB dos serviços, exclusive administração pública. Unidade: mil reais |
| `vab_administracao_mil_brl` | DOUBLE | VAB da administração, defesa, educação e saúde públicas. Unidade: mil reais |
| `setor_dominante` | STRING | Setor com maior VAB. Domínio: `Agropecuaria`, `Industria`, `Servicos`, `Administracao publica` |

---

## `gold.telecom.dim_empresa`

Dimensão de prestadoras de SCM atuantes no RS.

**Chave surrogate:** `sk_empresa` · **Chave natural:** `cnpj`
**Cardinalidade:** 716 linhas

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_empresa` | INT | Chave surrogate. Inteiro sequencial determinístico gerado pela ordenação do CNPJ. Domínio: 1 a 716 |
| `cnpj` | STRING | CNPJ da prestadora, 14 dígitos sem formatação. Chave natural e chave de identidade competitiva |
| `empresa` | STRING | Nome comercial da prestadora. Determinado funcionalmente pelo CNPJ |
| `grupo_economico` | STRING | Grupo econômico declarado. Domínio: 16 categorias no RS. `OUTROS` agrega todos os provedores regionais, 55% das linhas |
| `porte` | STRING | Porte segundo a Anatel. Domínio: `Pequeno Porte`, `Grande Porte` |
| `grupo_identificado` | BOOLEAN | Indica grupo diferente de `OUTROS`. Domínio: `true`, `false` |

---

## `gold.telecom.dim_tecnologia`

Dimensão de tecnologias de acesso. A tecnologia **não** determina funcionalmente o
meio físico, por isso a chave natural é composta.

**Chave surrogate:** `sk_tecnologia` · **Chave natural composta:** `tecnologia` + `meio_acesso`

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_tecnologia` | INT | Chave surrogate. Inteiro sequencial determinístico gerado pela ordenação de tecnologia e meio de acesso |
| `tecnologia` | STRING | Tecnologia de acesso declarada. Parte da chave natural composta. Domínio: 24 categorias — `FTTH`, `ETHERNET`, `HFC`, `VSAT`, `ADSL2`, entre outras |
| `meio_acesso` | STRING | Meio físico de transmissão. Parte da chave natural composta. Domínio: `Fibra`, `Rádio`, `Satélite`, `Cabo Metálico`, `Cabo Coaxial` |
| `e_fibra` | BOOLEAN | Indica meio de acesso igual a `Fibra`. Domínio: `true`, `false`. Sustenta as análises de penetração de fibra |

---

## `gold.telecom.dim_faixa_velocidade`

Dimensão de faixas de velocidade contratada. A coluna `ordem_faixa` estabelece a
ordenação correta das categorias, que não é alfabética.

**Chave surrogate:** `sk_faixa` · **Chave natural:** `faixa_velocidade`
**Cardinalidade:** 5 linhas

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_faixa` | INT | Chave surrogate, igual a `ordem_faixa`. Domínio: 1 a 5 |
| `faixa_velocidade` | STRING | Faixa de velocidade contratada. Chave natural. Domínio: `0Kbps a 512Kbps`, `512kbps a 2Mbps`, `2Mbps a 12Mbps`, `12Mbps a 34Mbps`, `> 34Mbps` |
| `ordem_faixa` | INT | Ordem crescente de velocidade. Unidade: posição ordinal. Domínio: 1 (mais lenta) a 5 (mais rápida) |
| `e_defasada` | BOOLEAN | Indica faixa abaixo de 34 Mbps. Domínio: `true` para as faixas 1 a 4, `false` para a faixa 5. Base do cálculo de `defasagem_pct` |

### Conteúdo integral

| `sk_faixa` | `faixa_velocidade` | `ordem_faixa` | `e_defasada` |
|---|---|---|---|
| 1 | `0Kbps a 512Kbps` | 1 | `true` |
| 2 | `512kbps a 2Mbps` | 2 | `true` |
| 3 | `2Mbps a 12Mbps` | 3 | `true` |
| 4 | `12Mbps a 34Mbps` | 4 | `true` |
| 5 | `> 34Mbps` | 5 | `false` |

---

## `gold.telecom.dim_segmento`

Dimensão lixo (*junk dimension*) combinando tipo de pessoa e tipo de produto, ambos
de baixa cardinalidade. As flags recortam os numeradores da análise sem descartar
nenhum acesso.

**Chave surrogate:** `sk_segmento` · **Chave natural composta:** `tipo_pessoa` + `tipo_produto`

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_segmento` | INT | Chave surrogate determinística |
| `tipo_pessoa` | STRING | Natureza do assinante. Domínio: `Pessoa Física`, `Pessoa Jurídica` |
| `tipo_produto` | STRING | Tipo de produto. Domínio: `INTERNET`, `LINHA_DEDICADA`, `M2M`, `OUTROS` |
| `e_pessoa_fisica` | BOOLEAN | Indica assinante pessoa física, qualquer produto. Domínio: `true`, `false` |
| `e_pessoa_juridica` | BOOLEAN | Indica assinante pessoa jurídica, qualquer produto. Domínio: `true`, `false` |
| `e_internet` | BOOLEAN | Indica produto `INTERNET`, qualquer tipo de pessoa. Domínio: `true`, `false` |
| `e_linha_dedicada` | BOOLEAN | Indica produto `LINHA_DEDICADA`. Domínio: `true`, `false` |
| `e_m2m` | BOOLEAN | Indica produto `M2M`, comunicação entre máquinas. Domínio: `true`, `false` |
| `e_residencial` | BOOLEAN | Indica pessoa física com produto `INTERNET`, recorte residencial estrito. Domínio: `true`, `false` |

---

## `gold.telecom.fato_acessos`

Tabela de fatos de acessos de banda larga fixa no RS.

**Grão:** município × empresa × tecnologia com meio de acesso × faixa de velocidade ×
segmento, para um período de referência
**Medida:** `acessos_qtd`, aditiva em todas as dimensões

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `periodo` | STRING | Período de referência, formato AAAA-MM. Atributo degenerado. Domínio: `2026-07` |
| `sk_municipio` | INT | Chave estrangeira para `gold.dim_municipio` |
| `sk_empresa` | INT | Chave estrangeira para `gold.dim_empresa` |
| `sk_tecnologia` | INT | Chave estrangeira para `gold.dim_tecnologia`, cuja chave natural na origem é composta por tecnologia e meio de acesso |
| `sk_faixa` | INT | Chave estrangeira para `gold.dim_faixa_velocidade` |
| `sk_segmento` | INT | Chave estrangeira para `gold.dim_segmento` |
| `acessos_qtd` | INT | Quantidade de acessos ativos. Unidade: acessos. Medida aditiva. Domínio: inteiro ≥ 1. Linhagem: `silver.acessos.acessos_qtd` |

---

## `gold.telecom.mercado_municipio`

Tabela analítica com uma linha por município do RS. Expõe sete numeradores de acesso
e dois denominadores de mercado.

**Grão:** município · **Cardinalidade:** 497 linhas

### Identificação e contexto

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_municipio` | INT | Chave surrogate do município. Linhagem: `gold.dim_municipio` |
| `codigo_ibge` | STRING | Código IBGE do município, 7 dígitos. Chave natural |
| `nome_municipio` | STRING | Nome oficial do município |
| `mesorregiao_nome` | STRING | Mesorregião geográfica do IBGE. Domínio: 7 categorias no RS |
| `regiao_intermediaria_nome` | STRING | Região geográfica intermediária do IBGE |
| `porte_populacional` | STRING | Faixa de porte populacional. Domínio: 4 categorias |
| `populacao_residente_hab` | DOUBLE | População residente. Unidade: habitantes. Linhagem: SIDRA 4714 v/93 |
| `area_km2` | DOUBLE | Área da unidade territorial. Unidade: km². Linhagem: SIDRA 4714 v/6318 |
| `densidade_demografica_hab_km2` | DOUBLE | Densidade demográfica. Unidade: habitantes por km². Proxy de custo de rede por assinante: quanto menor a densidade, maior o custo de cobertura. Linhagem: SIDRA 4714 v/614 |
| `pib_per_capita_brl` | DOUBLE | PIB por habitante. Unidade: reais. Linhagem: SIDRA 5938, ano 2021 |
| `setor_dominante` | STRING | Setor com maior VAB. Domínio: `Agropecuaria`, `Industria`, `Servicos`, `Administracao publica` |

### Denominadores de mercado

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `domicilios_total_qtd` | DOUBLE | Total de domicílios recenseados, todas as espécies. Unidade: domicílios. Denominador de mercado endereçável. Linhagem: SIDRA 4711 v/617 c3/59993 |
| `domicilios_ocupados_qtd` | DOUBLE | Domicílios particulares permanentes ocupados. Unidade: domicílios. Denominador conservador. Linhagem: SIDRA 4712 v/381 |
| `domicilios_uso_ocasional_qtd` | DOUBLE | Domicílios de uso ocasional, típicos de veraneio. Unidade: domicílios. Linhagem: SIDRA 4711 c3/60003 |
| `uso_ocasional_pct` | DOUBLE | Participação de imóveis de uso ocasional no total. Unidade: percentual. Domínio: 0 a 100 |
| `perfil_ocupacao` | STRING | Classificação pelo peso do uso ocasional. Domínio: `Ocupacao permanente`, `Ocupacao mista`, `Veraneio ou turismo` |

### Numeradores de acesso

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `acessos_total_qtd` | LONG | Total de acessos de banda larga fixa, todos os tipos. Unidade: acessos. Linhagem: soma de `gold.fato_acessos` por município |
| `acessos_pf_qtd` | LONG | Acessos de pessoa física, qualquer produto. Unidade: acessos. Linhagem: filtro por `dim_segmento.e_pessoa_fisica` |
| `acessos_pj_qtd` | LONG | Acessos de pessoa jurídica, qualquer produto. Unidade: acessos. Linhagem: filtro por `dim_segmento.e_pessoa_juridica` |
| `acessos_residencial_qtd` | LONG | Acessos de pessoa física com produto `INTERNET`, recorte residencial estrito. Unidade: acessos |
| `acessos_internet_qtd` | LONG | Acessos com produto `INTERNET`, pessoa física e jurídica. Unidade: acessos |
| `acessos_dedicado_qtd` | LONG | Acessos com produto `LINHA_DEDICADA`. Unidade: acessos. Tipicamente corporativos de maior valor |
| `acessos_m2m_qtd` | LONG | Acessos com produto `M2M`, comunicação entre máquinas. Unidade: acessos |

### Penetração e mercado não atendido

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `penetracao_ocupados_pct` | DOUBLE | Acessos totais sobre domicílios ocupados. Unidade: percentual. Denominador conservador; superestima municípios de veraneio |
| `penetracao_enderecavel_pct` | DOUBLE | Acessos totais sobre domicílios recenseados. Unidade: percentual. **Métrica principal de penetração deste MVP** |
| `penetracao_residencial_pct` | DOUBLE | Acessos residenciais estritos sobre domicílios recenseados. Unidade: percentual |
| `contratos_por_domicilio` | DOUBLE | Acessos totais ÷ domicílios recenseados. Unidade: contratos por domicílio. Mesma razão de `penetracao_enderecavel_pct`, em escala unitária. Valores acima de 1 indicam múltiplos contratos por imóvel, típicos de mercados saturados com competição intensa |
| `saldo_domicilios_qtd` | DOUBLE | Diferença entre domicílios recenseados e acessos totais. Unidade: domicílios. Pode ser negativo |
| `domicilios_sem_acesso_qtd` | DOUBLE | Mercado não atendido, com piso em zero. Unidade: domicílios. Linhagem: maior valor entre 0 e `saldo_domicilios_qtd` |
| `penetracao_acima_100` | BOOLEAN | Sinaliza penetração endereçável acima de 100%. Domínio: `true`, `false`. Indica múltiplos contratos por imóvel ou acessos declarados no município da prestadora |
| `mercado_saturado` | BOOLEAN | Sinaliza penetração endereçável igual ou superior a 90%. Domínio: `true`, `false` |

### Estrutura competitiva

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `n_provedores_qtd` | LONG | CNPJs distintos com acesso declarado. Unidade: prestadoras. Valor 1 indica monopólio de fato |
| `provedor_lider` | STRING | Nome da prestadora com maior número de acessos no município |
| `share_lider_pct` | DOUBLE | Participação do provedor líder. Unidade: percentual. Domínio: 0 a 100 |
| `hhi` | DOUBLE | Índice Herfindahl-Hirschman. Unidade: índice adimensional. Domínio: 0 a 10.000. Fórmula: soma dos quadrados dos shares percentuais por CNPJ |
| `concentracao` | STRING | Classificação do HHI. Domínio: `Baixa` (abaixo de 1.500), `Moderada` (1.500 a 2.499), `Alta` (2.500 ou mais). Limiares usuais de defesa da concorrência |

### Perfil tecnológico

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `fibra_pct` | DOUBLE | Participação da fibra óptica nos acessos. Unidade: percentual. Domínio: 0 a 100. Linhagem: `dim_tecnologia.e_fibra` |
| `defasagem_pct` | DOUBLE | Participação de acessos abaixo de 34 Mbps. Unidade: percentual. Domínio: 0 a 100. Linhagem: `dim_faixa_velocidade.e_defasada` |
| `faixa_modal` | STRING | Faixa de velocidade com maior volume de acessos no município. Domínio: as 5 faixas de `dim_faixa_velocidade` |

---

## `gold.telecom.prestadora_atuacao`

Tabela analítica com uma linha por prestadora atuante no RS.

**Grão:** CNPJ · **Cardinalidade:** 716 linhas

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `sk_empresa` | INT | Chave surrogate da prestadora. Linhagem: `gold.dim_empresa` |
| `cnpj` | STRING | CNPJ da prestadora, 14 dígitos. Chave natural |
| `empresa` | STRING | Nome comercial da prestadora |
| `grupo_economico` | STRING | Grupo econômico declarado. `OUTROS` para provedores regionais |
| `porte` | STRING | Porte segundo a Anatel. Domínio: `Pequeno Porte`, `Grande Porte` |
| `acessos_total_qtd` | LONG | Total de acessos da prestadora no RS. Unidade: acessos |
| `acessos_pf_qtd` | LONG | Acessos de pessoa física. Unidade: acessos |
| `acessos_pj_qtd` | LONG | Acessos de pessoa jurídica. Unidade: acessos |
| `share_estadual_pct` | DOUBLE | Participação no total de acessos do estado. Unidade: percentual. Domínio: 0 a 100 |
| `municipios_qtd` | LONG | Municípios do RS em que a prestadora tem acesso declarado. Unidade: municípios. Domínio: 1 a 497 |
| `acessos_por_municipio_qtd` | DOUBLE | Média de acessos por município atendido. Unidade: acessos por município. Indica densidade da operação |
| `municipios_lider_qtd` | LONG | Municípios em que a prestadora é a maior em acessos. Unidade: municípios |
| `fibra_pct` | DOUBLE | Participação da fibra na base da prestadora. Unidade: percentual. Domínio: 0 a 100. Indica modernidade da rede |
| `pj_pct` | DOUBLE | Participação de pessoa jurídica na base. Unidade: percentual. Domínio: 0 a 100. Indica orientação corporativa da operação |
| `perfil_atuacao` | STRING | Classificação por abrangência geográfica. Domínio: `Municipal` (1 município), `Regional` (2 a 10), `Multirregional` (11 a 50), `Estadual` (mais de 50) |

---

## `gold.telecom.indice_oportunidade`

Índice composto de oportunidade comercial por município. Combina três componentes em
percentil, com peso igual.

**Grão:** município · **Cardinalidade:** 497 linhas

> **O índice é ferramenta de priorização analítica, não recomendação de
> investimento.**

| Coluna | Tipo | Descrição, domínio e linhagem |
|---|---|---|
| `codigo_ibge` | STRING | Código IBGE do município, 7 dígitos. Chave natural |
| `nome_municipio` | STRING | Nome oficial do município |
| `mesorregiao_nome` | STRING | Mesorregião geográfica do IBGE |
| `porte_populacional` | STRING | Faixa de porte populacional. Domínio: 4 categorias |
| `perfil_ocupacao` | STRING | Classificação pelo peso do uso ocasional. Domínio: `Ocupacao permanente`, `Ocupacao mista`, `Veraneio ou turismo` |
| `domicilios_total_qtd` | DOUBLE | Total de domicílios recenseados. Unidade: domicílios. Denominador de mercado endereçável |
| `domicilios_sem_acesso_qtd` | DOUBLE | Mercado não atendido, piso em zero. Unidade: domicílios. **Componente 1 do índice** |
| `acessos_total_qtd` | LONG | Total de acessos no município. Unidade: acessos |
| `penetracao_enderecavel_pct` | DOUBLE | Acessos totais sobre domicílios recenseados. Unidade: percentual |
| `defasagem_pct` | DOUBLE | Participação de acessos abaixo de 34 Mbps. Unidade: percentual. **Componente 2 do índice** |
| `fibra_pct` | DOUBLE | Participação da fibra óptica. Unidade: percentual. Domínio: 0 a 100 |
| `pib_per_capita_brl` | DOUBLE | PIB por habitante. Unidade: reais. **Componente 3 do índice** |
| `n_provedores_qtd` | LONG | CNPJs distintos atuando no município. Unidade: prestadoras |
| `hhi` | DOUBLE | Índice Herfindahl-Hirschman. Domínio: 0 a 10.000 |
| `concentracao` | STRING | Classificação do HHI. Domínio: `Baixa`, `Moderada`, `Alta` |
| `p_nao_atendido` | DOUBLE | Percentil de `domicilios_sem_acesso_qtd` no universo de municípios. Domínio: 0 a 1 |
| `p_defasagem` | DOUBLE | Percentil de `defasagem_pct` no universo de municípios. Domínio: 0 a 1 |
| `p_renda` | DOUBLE | Percentil de `pib_per_capita_brl` no universo de municípios. Domínio: 0 a 1 |
| `indice_oportunidade` | DOUBLE | Média simples dos três percentis, multiplicada por 100. Unidade: índice adimensional. Domínio: 0 a 100. Pesos iguais por escolha neutra declarada |

---

