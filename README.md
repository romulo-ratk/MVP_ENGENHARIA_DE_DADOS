# MVP de Engenharia de Dados — Mercado de Banda Larga Fixa nos Municípios do RS

Pipeline de dados construído no **Databricks** para analisar a estrutura
do mercado de banda larga fixa nos 497 municípios do Rio Grande do Sul, cruzando
dados regulatórios da Anatel com dados socioeconômicos do IBGE.

**ALUNO: RÔMULO A. T. KOHLER**

**Curso:** Pós-Graduação em Data Science & Analytics — PUC-Rio
**Disciplina:** Engenharia de Dados

---

## Sumário

- Contexto de Negócio e Perguntas 
- Carga dos Dados
- Modelagem e Catálogo de Dados 
- Pipeline de Dados
- Qualidade de Dados
- Análise de Dados
- Autoavaliação

---

## Contexto de Negócio e Perguntas 

### O problema

O mercado brasileiro de banda larga fixa passou por uma transformação profunda na
última década: a fibra óptica substituiu tecnologias legadas, e centenas de
provedores regionais surgiram disputando espaço com as grande operadoras. No Rio
Grande do Sul esse movimento é particularmente intenso — o estado tem um dos
ecossistemas de ISPs regionais mais densos do país.

Para um provedor que precisa decidir onde expandir rede, ou para um analista que
precisa entender a dinâmica competitiva do setor, a pergunta central é: **onde ainda
existe mercado não atendido, e como está estruturada a concorrência em cada
município?**

Os dados existem e são públicos. A Anatel publica mensalmente a base de acessos
declarada pelas prestadoras, com granularidade de município, CNPJ, tecnologia e faixa
de velocidade. O IBGE publica população, domicílios e PIB municipal. O que não existe
é a integração entre eles — e é isso que este pipeline entrega.

### Perguntas de negócio

Definidas antes da coleta.

1. **Quais municípios concentram o maior volume de domicílios ainda não atendidos?**
2. **A penetração varia com o porte do município? Existe faixa populacional
   sistematicamente mal servida?**
3. **Como se distribui o número de provedores por município? Quantos são monopólio de
   fato?**
4. **Qual a participação de fibra óptica por município e por região? Quais municípios ainda
   dependem de rádio, satélite ou cobre?**
5. **Quais municípios têm a base de velocidade mais defasada — potencial de upgrade
   de plano?**
6. **A penetração se relaciona com PIB per capita ou com o setor econômico
   dominante?**
7. **Considerando mercado não atendido, defasagem tecnológica e poder aquisitivo,
   quais municípios apresentam maior oportunidade comercial?**

### Fontes de dados

#### Anatel — Acessos Banda Larga Fixa (SCM)

Base declarada pelas prestadoras de Serviço de Comunicação Multimídia à Anatel, com
força regulatória. É o registro oficial de acessos ativos no país.

- **URL:** `https://www.anatel.gov.br/dadosabertos/paineis_de_dados/acessos/acessos_banda_larga_fixa.zip`
- **Formato:** CSV, separador `;`, encoding UTF-8
- **Cobertura:** nacional, série mensal desde 2007, particionada em arquivos anuais
- **Recorte utilizado:** Rio Grande do Sul, julho de 2026
- **Volume no recorte:** 71.608 linhas, 497 municípios, 716 CNPJs distintos

**Estrutura dos dados brutos** (16 colunas):

| Coluna | Descrição |
|---|---|
| `Ano`, `Mês` | Período de referência |
| `Grupo Econômico` | Grupo controlador declarado |
| `Empresa` | Nome comercial da prestadora |
| `CNPJ` | CNPJ da prestadora, 14 dígitos |
| `Porte da Prestadora` | Pequeno Porte ou Grande Porte |
| `UF`, `Município`, `Código IBGE Município` | Localização |
| `Faixa de Velocidade` | 5 faixas, de 0Kbps a acima de 34Mbps |
| `Velocidade` | Valor exato contratado |
| `Tecnologia` | 24 categorias (FTTH, ETHERNET, HFC, VSAT, ADSL2...) |
| `Meio de Acesso` | Fibra, Rádio, Satélite, Cabo Metálico, Cabo Coaxial |
| `Tipo de Pessoa` | Pessoa Física, Pessoa Jurídica |
| `Tipo de Produto` | INTERNET, LINHA_DEDICADA, M2M, OUTROS |
| `Acessos` | Quantidade de acessos ativos |

Dois arquivos auxiliares da mesma fonte foram ingeridos para validação: o total
nacional por mês e a densidade calculada pela própria Anatel por município.

#### IBGE — API de Localidades

Municípios do RS com código IBGE de 7 dígitos e hierarquia geográfica completa:
microrregião, mesorregião, região imediata e região intermediária.

- **URL:** `https://servicodados.ibge.gov.br/api/v1/localidades/estados/RS/municipios`

#### IBGE — API SIDRA

Quatro consultas ao Sistema IBGE de Recuperação Automática:

| Tabela | Variáveis | Conteúdo | Período |
|---|---|---|---|
| 4714 | 93, 6318, 614 | População residente, área territorial, densidade demográfica | Censo 2022 |
| 4712 | 381, 382, 5930 | Domicílios particulares permanentes ocupados, moradores, média | Censo 2022 |
| 4711 | 617 (c3: 59993, 59998, 60001, 60002, 60003) | Domicílios recenseados por espécie | Censo 2022 |
| 5938 | 37, 543, 498, 513, 517, 6575, 525 | PIB, impostos, VAB total e por setor | 2021 |

- **Padrão de URL:** `https://apisidra.ibge.gov.br/values/t/{tabela}/n6/in n3 43/v/{variáveis}/p/{período}`

O período do PIB é 2021 por ser o ano mais recente com desdobramento setorial
completo — detalhado na seção de Qualidade de Dados.

### Licença de uso

**Anatel.** Dados abertos publicados sob a Lei de Acesso à Informação (Lei
12.527/2011) e o Plano de Dados Abertos da Agência. Uso livre com citação da fonte.

**IBGE.** Dados públicos de livre utilização, disponibilizados por serviço público
gratuito, com citação da fonte.

Ambas as fontes permitem uso acadêmico e redistribuição. Conforme o item 4 do
enunciado, os dados não são disponibilizados neste repositório — apenas o código que
os coleta e transforma.

---

## Carga dos Dados

### Restrição da plataforma

O Databricks Free Edition restringe o acesso de saída à internet a um conjunto
limitado de domínios confiáveis. A documentação oficial indica que a **verificação de
identidade via LinkedIn** libera o acesso de saída — passo executado antes da
ingestão, sem o qual nenhuma das chamadas a `anatel.gov.br` ou `ibge.gov.br`
funcionaria.

Essa é a primeira decisão arquitetural do projeto e está registrada na autoavaliação
como dificuldade encontrada.

### Estratégia de ingestão

**Anatel.** Download do ZIP anual para o disco local do driver, descompactação e
cópia dos CSVs para um Volume do Unity Catalog (`/Volumes/bronze/telecom/raw`).
Volumes não lidam bem com escrita aleatória, por isso a descompactação acontece em
`/tmp` antes da cópia.

O arquivo anual de 2026 tem cerca de 680 MB descompactado e é ingerido **integralmente**
— abrangência nacional, todos os meses do ano. O recorte para RS e julho é decisão da
camada Silver, não da coleta, preservando o princípio de que a Bronze guarda o dado
como veio.

**IBGE.** Requisições HTTP diretas às APIs, com o JSON convertido em DataFrame Spark.
A API SIDRA retorna formato longo — uma linha por município por variável — e a
primeira linha do retorno é cabeçalho descritivo, descartada na ingestão.

### Metadados de controle

Toda tabela da camada Bronze recebe três colunas de rastreabilidade:

| Coluna | Conteúdo |
|---|---|
| `_arquivo_origem` | Nome do arquivo ou URL da requisição |
| `_fonte` | Identificação da fonte e da tabela de origem |
| `_data_ingestao` | Timestamp UTC da ingestão |

### Scripts

| Notebook | Responsabilidade |
|---|---|
| [`01_bronze_ingestao.ipynb`](notebooks/01_bronze_ingestao.ipynb) | Download, descompactação e ingestão das sete tabelas Bronze |

---

## Modelagem e Catálogo de Dados 

### Arquitetura Medalhão

O pipeline segue a progressão Bronze → Silver → Gold, com cada camada em um catálogo
próprio do Unity Catalog.

```
bronze.telecom          silver.telecom           gold.telecom
─────────────────       ──────────────────       ────────────────────────
anatel_acessos_raw      acessos                  dim_municipio
anatel_total_raw        empresas                 dim_empresa
anatel_densidade_raw    tecnologias              dim_tecnologia
ibge_municipios_raw     municipios               dim_faixa_velocidade
ibge_populacao_raw      qualidade_dados          dim_segmento
ibge_domicilios_raw                              fato_acessos
ibge_domicilios_        [dado limpo, tipado,     mercado_municipio
  especie_raw            no recorte do MVP]      prestadora_atuacao
ibge_pib_raw                                     indice_oportunidade

[dado como veio,                                 [modelo estrela +
 tudo como texto]                                 tabelas analíticas]
```

### Modelo dimensional — Esquema Estrela

```
                          dim_municipio
                                │
      dim_empresa ────────  fato_acessos  ──────── dim_tecnologia
                            /          \
              dim_faixa_velocidade    dim_segmento
```

**Grão da tabela de fatos:** município × empresa × tecnologia+meio de acesso × faixa
de velocidade × segmento, para o período de referência.

**Medida:** `acessos_qtd`, aditiva em todas as dimensões.

### Decisões de modelagem

**Chaves surrogate determinísticas.** Geradas por `row_number()` sobre a ordenação da
chave natural, não por `monotonically_increasing_id()`. Reexecutar o pipeline produz
exatamente as mesmas chaves, o que torna o resultado reproduzível.

**`dim_tecnologia` com chave composta.** A hipótese inicial era que cada tecnologia
usaria um meio físico fixo. O dado refutou: ETHERNET aparece sobre fibra e sobre cabo
metálico. Tratar a relação como funcional produziria chave duplicada na dimensão e
inflaria os acessos no join da fato. A chave natural é `tecnologia` + `meio_acesso`.

**Dimensão lixo (junk dimension).** `tipo_pessoa` (2 valores) e `tipo_produto`
(4 valores) foram combinados em `dim_segmento`. Criar duas dimensões separadas para
atributos de cardinalidade tão baixa seria desnecessário; o padrão de Kimball para
esse caso é a dimensão lixo. Ela carrega seis flags booleanas que recortam os
numeradores da análise sem descartar nenhum acesso.

**`dim_faixa_velocidade` com ordem explícita.** As cinco faixas são categorias de
texto sem ordenação natural. Sem a coluna `ordem_faixa`, qualquer gráfico ou ranking
sairia alfabético, com `> 34Mbps` antes de `0Kbps a 512Kbps`.

**Sem dimensão de tempo.** O MVP trabalha com um único período de referência; uma
dimensão de uma linha seria degenerada. O período é atributo da tabela de fatos. A
estrutura suporta a inclusão de `dim_tempo` quando a série histórica for incorporada.

**Sem SCD Tipo 2.** Pelo mesmo motivo: sem histórico, não há mudança a versionar. As
dimensões estão preparadas para receber versionamento em evolução futura.

**Redução de grão.** A coluna `Velocidade` (valor exato) tem 1.135 valores distintos
no recorte do RS. `Faixa de Velocidade`, com 5 categorias, responde a todas as
perguntas de negócio com cardinalidade muito menor. A velocidade exata foi descartada
na camada Silver.

### Análise de dependências funcionais

Verificada empiricamente sobre o recorte do RS, com resultado que orientou a
separação das dimensões:

| Determinante | Determinado | Verificação |
|---|---|---|
| `cnpj` | `empresa`, `grupo_economico`, `porte` | ✅ Zero inconsistências |
| `codigo_ibge` | `municipio`, `uf` | ✅ Zero inconsistências |
| `tecnologia` | `meio_acesso` | ❌ **Refutada** — ETHERNET usa fibra e cabo metálico |

### Tabelas analíticas da camada Gold

| Tabela | Grão | Conteúdo |
|---|---|---|
| `mercado_municipio` | município | Sete numeradores de acesso, dois denominadores de mercado, três penetrações, estrutura competitiva (HHI, share do líder, número de provedores) e perfil tecnológico |
| `prestadora_atuacao` | CNPJ | Pegada geográfica, share estadual, municípios em que é líder, composição PF/PJ e perfil de atuação |
| `indice_oportunidade` | município | Índice composto de oportunidade comercial com os três componentes em percentil |

### Catálogo de Dados

O catálogo é implementado diretamente no **Unity Catalog** via `COMMENT ON TABLE` e
`COMMENT ON COLUMN`, aplicados programaticamente nas camadas Silver e Gold. Cada
coluna documenta:

- **Descrição** — o que o campo representa
- **Tipo de dado** — implícito no schema Delta, visível no Catalog Explorer
- **Domínio de valores** — faixas para numéricos, categorias para categóricos
- **Linhagem** — fonte de origem e transformações aplicadas

### Convenção de nomes

A unidade de medida faz parte do nome de cada coluna:

| Sufixo | Unidade |
|---|---|
| `_qtd` | contagem de unidades |
| `_hab` | habitantes |
| `_pct` | percentual (0 a 100) |
| `_brl` | reais |
| `_mil_brl` | mil reais (unidade original do IBGE) |
| `_km2` | quilômetros quadrados |

---

## Pipeline de Dados

### Organização

O pipeline foi ramificado em quatro notebooks, um por etapa lógica. A separação
facilita a execução parcial durante o desenvolvimento e torna a documentação de cada
transformação mais direta.

| Notebook | Camada | Responsabilidade |
|---|---|---|
| [`01_bronze_ingestao.ipynb`](notebooks/01_bronze_ingestao.ipynb) | Bronze | Download e ingestão bruta das sete tabelas de origem |
| [`02_silver_transformacao.ipynb`](notebooks/02_silver_transformacao.ipynb) | Silver | Recorte, tipagem, normalização e verificações de qualidade |
| [`03_gold_modelagem.ipynb`](notebooks/03_gold_modelagem.ipynb) | Gold | Modelo estrela e cálculo das métricas de negócio |
| [`04_analise.ipynb`](notebooks/04_analise.ipynb) | — | Respostas às perguntas de negócio |

### Transformações da camada Bronze

Nenhuma transformação de conteúdo. A única alteração é a **normalização dos nomes de
coluna para snake_case**, exigida pelo formato Delta, que rejeita espaços e acentos em
nomes de coluna sem habilitar column mapping. `Código IBGE Município` torna-se
`codigo_ibge_municipio`; o conteúdo permanece íntegro e tudo é gravado como texto.

### Transformações da camada Silver

| Transformação | O que foi feito | Por quê | Impacto |
|---|---|---|---|
| **Recorte** | Filtro por UF = RS, ano = 2026, mês = 7 | Escopo do MVP | Redução do nacional para 71.608 linhas |
| **Tipagem** | `acessos` de texto para inteiro; valores do SIDRA para decimal | Bronze preserva tudo como texto | Habilita agregação e comparação |
| **Valores especiais** | `-`, `..`, `...` e `X` do SIDRA convertidos explicitamente para nulo | Conversão direta produziria nulo silencioso, sem registro do motivo | Torna a ausência de dado visível na análise de completude |
| **Separador decimal** | Vírgula para ponto no arquivo de densidade da Anatel | Formato brasileiro incompatível com cast numérico | Habilita a validação de acurácia |
| **Pivot do SIDRA** | Formato longo para largo, uma linha por município | A API retorna uma linha por município por variável | 497 linhas por consulta |
| **Redução de grão** | Descarte da coluna `Velocidade` (valor exato) | 1.135 valores distintos sem ganho analítico sobre as 5 faixas | Redução expressiva de cardinalidade |
| **Normalização** | Separação de `empresas`, `tecnologias` e `municipios` em tabelas próprias | Atributos com dependência funcional não pertencem ao grão da fato | Prepara o modelo dimensional |
| **Consolidação IBGE** | Join das cinco fontes do IBGE em uma tabela por município | Todas compartilham o mesmo grão | Uma linha por município com todos os atributos |
| **Setor dominante** | Maior VAB setorial, com nulo tratado como zero | `F.greatest()` retorna nulo se qualquer argumento for nulo | Classificação objetiva, sem julgamento subjetivo |

### Transformações da camada Gold

| Transformação | O que foi feito | Por quê |
|---|---|---|
| **Chaves surrogate** | `row_number()` sobre a chave natural ordenada | Determinismo e reprodutibilidade |
| **Substituição de chaves** | Chaves naturais trocadas por surrogate na tabela de fatos | Padrão de esquema estrela |
| **Agregação por município** | Sete numeradores de acesso via flags de `dim_segmento` | Permite analisar cada recorte sem descartar dados |
| **Cálculo de HHI** | Soma dos quadrados dos shares por CNPJ dentro de cada município | Medida padrão de concentração usada por autoridades antitruste |
| **Percentis** | `percent_rank()` com partição constante declarada | Evita que variáveis de escala maior dominem o índice composto |

### Validação de integridade referencial

Após a construção da tabela de fatos, o pipeline verifica que nenhuma linha e nenhum
acesso se perdeu ou multiplicou nos cinco joins com as dimensões, e que nenhuma chave
estrangeira ficou nula. Divergência aqui indicaria chave duplicada em alguma
dimensão — foi exatamente essa validação que revelou o problema da `dim_tecnologia`.

---

## Qualidade de Dados

As verificações estão implementadas nos notebooks das camadas em que são executadas.
A camada Silver traz as cinco dimensões exigidas, com registro consolidado na tabela
`silver.telecom.qualidade_dados`. A camada Gold traz a validação de integridade
referencial e a investigação da penetração acima de 100%.

### Completude

Contagem de nulos coluna a coluna em `silver.acessos` e `silver.municipios`.

**A base da Anatel está completa.** Zero nulos em todas as colunas, incluindo código
IBGE e quantidade de acessos. Conforme o enunciado prevê para bases curadas, a
evidência da verificação é o próprio resultado.

**O VAB setorial do IBGE não estava.** Detalhado abaixo.

### Consistência

| Verificação | Resultado |
|---|---|
| Municípios na Anatel × municípios no IBGE | 497 em ambos, cobertura total |
| Identidade contábil `PIB = VAB total + impostos` | *(a completar)* |
| Coerência entre SIDRA 4712 v/381 e SIDRA 4711 c3/59998 | *(a completar)* |
| Completude do VAB setorial | Detalhado abaixo |

### Unicidade

O grão declarado de `silver.acessos` é verificado contra a contagem de combinações
distintas da chave. Duplicata aqui indicaria erro de agregação e inflaria todas as
métricas da camada Gold. As três tabelas de dimensão também têm a unicidade da chave
verificada.

### Acurácia

Validação contra fonte externa independente: a densidade calculada pelo pipeline é
comparada com a densidade oficial publicada pela própria Anatel, município a
município. 


### Outliers

Distribuição da penetração residencial por quartil, e investigação dos casos acima de
100%.

---

### Problema 1 — Grupo econômico não serve como chave competitiva

**Detecção.** A distribuição de `Grupo Econômico` no recorte do RS mostra que 39.487
das 71.608 linhas — **55% do dado** — estão classificadas como `OUTROS`.

**Diagnóstico.** A Anatel preenche o grupo econômico apenas para as grandes
operadoras. Todos os provedores regionais caem no mesmo balde. Como o mercado gaúcho é
dominado por ISPs regionais, calcular market share ou HHI por grupo econômico
produziria um resultado sem sentido: mais da metade do mercado apareceria como uma
única empresa.

**Tratamento.** A identidade competitiva do projeto passou a ser o **CNPJ**. O grupo
econômico foi preservado como atributo de `dim_empresa`, com uma flag
`grupo_identificado` que distingue grandes grupos de provedores regionais.

**Impacto.** Decisão estrutural que atravessa todo o modelo. Sem ela, a análise
competitiva — três das sete perguntas — seria inválida.

---

### Problema 2 — Tecnologia não determina o meio de acesso

**Detecção.** A validação de integridade referencial da camada Gold acusou
multiplicação de linhas no join com `dim_tecnologia`: a soma de acessos da Gold era
maior que a da Silver.

**Diagnóstico.** A hipótese inicial de modelagem era que cada tecnologia usaria um
meio físico fixo — FTTH sempre fibra, VSAT sempre satélite. A verificação empírica
refutou: ETHERNET aparece tanto sobre fibra quanto sobre cabo metálico. Como
`meio_acesso` havia sido tratado como atributo da dimensão, a chave `tecnologia`
ficou duplicada, e o join multiplicou os acessos.

**Tratamento.** `meio_acesso` voltou para o grão da tabela `silver.acessos`, e
`dim_tecnologia` passou a ter chave composta (`tecnologia` + `meio_acesso`).

**Impacto.** Correção de uma inflação sistemática dos acessos. A validação de
integridade referencial passou a bater exatamente entre Silver e Gold.

**Aprendizado.** A dependência funcional foi assumida sem verificação. Testá-la
empiricamente antes de modelar teria evitado o retrabalho — e é a razão pela qual a
análise de dependências funcionais virou uma etapa explícita do notebook Silver.

---

### Problema 3 — Denominador de mercado inadequado

**Detecção.** 103 dos 497 municípios apresentavam penetração residencial acima de
100% — mais acessos do que domicílios. Alguns chegavam a 300%, e a coluna de
domicílios não atendidos ficava negativa.

**Diagnóstico.** A lista dos casos extremos revelou um padrão geográfico inequívoco:
Xangri-lá, Imbé, Capão da Canoa, Arroio do Sal, Tramandaí, Arambaré, Cidreira,
Torres, Osório — todo o Litoral Norte gaúcho — mais Gramado, Canela e Ipê, na região
turística de serra.

A causa é o denominador. O Censo classifica domicílios por espécie, e a variável 381
da tabela 4712 conta apenas **domicílios particulares permanentes ocupados**. Imóveis
de veraneio entram na categoria "não ocupado — uso ocasional" e ficam fora da
contagem. Mas casa de praia tem internet contratada o ano todo: câmera de segurança,
streaming, trabalho remoto. **O acesso existe, o domicílio não era contado.**

**Tratamento.** Ingestão da tabela 4711 do SIDRA, que traz domicílios recenseados por
espécie, e adoção da categoria 59993 (total) como denominador de mercado endereçável.
As duas penetrações foram preservadas na camada Gold, permitindo comparação.

**Resultado.** Os municípios com penetração acima de 100% caíram de **103 para 12** —
uma redução de 88%. A hipótese foi confirmada com dado, não com argumento.

**Subproduto.** A dimensão de município ganhou `uso_ocasional_pct` e
`perfil_ocupacao`, que classificam automaticamente municípios de veraneio ou turismo
sem lista manual.

---

### Problema 4 — Os 12 municípios remanescentes

**Investigação.** Os casos que permaneceram acima de 100% mesmo com o denominador
ampliado foram analisados pela composição entre pessoa física e jurídica.

**Duas causas distintas:**

**Concentração corporativa** (9 casos). Municípios pequenos, entre mil e três mil
domicílios, com participação de pessoa jurídica entre 25% e 33% — muito acima da
média estadual. Uma empresa com algumas dezenas de links em cidade de 1.300
domicílios move a agulha sozinha, e esses acessos estão em endereço comercial, não
residencial. O denominador domiciliar não cobre esse numerador.

**Saturação com competição intensa** (Alvorada). O caso mais interessante: 79.779
domicílios, 105.725 acessos, 44 provedores, e participação de pessoa jurídica de
apenas 7,5%. A hipótese inicial era declaração centralizada pela operadora
incumbente, mas o dado refutou — a OI tem apenas 24% de share no município. O mercado
está pulverizado: quatro provedores dividem 79% dos acessos.

A explicação é multiplicidade de contratos por domicílio em mercado saturado. Quando
vários provedores disputam o mesmo endereço, o resultado natural é o domicílio com
fibra de um e link de backup de outro. **1,32 contratos por domicílio** não é
anomalia: é a medida real de intensidade competitiva.

**Tratamento.** Nenhum registro foi descartado. A flag `penetracao_acima_100`
identifica os casos, `saldo_domicilios_qtd` preserva o valor bruto (que pode ser
negativo), e `domicilios_sem_acesso_qtd` aplica piso em zero para o cálculo do índice.

A coluna `contratos_por_domicilio` foi adicionada como leitura alternativa da mesma
razão — "1,32 contratos por domicílio" descreve intensidade competitiva, enquanto
"132% de penetração" parece erro.

---

### Problema 5 — VAB setorial não publicado

**Detecção.** O gráfico de penetração por setor econômico saiu vazio. A verificação de
completude mostrou `setor_dominante` nulo nos 497 municípios, apesar de
`pib_per_capita_brl` estar preenchido.

**Diagnóstico em três passos:**

1. A camada Bronze tinha as 3.479 linhas esperadas (497 × 7 variáveis) — o download
   funcionou
2. A inspeção dos valores brutos mostrou que apenas a variável 37 (PIB total) tinha
   número; as outras seis retornavam `...` em todos os municípios
3. O teste de anos anteriores confirmou: **2021 tem as sete variáveis; 2022 e 2023
   publicam apenas o PIB agregado**

A SIDRA publica o desdobramento setorial do PIB municipal com defasagem maior que o
PIB total. Como o parâmetro de período estava como `last`, a consulta trouxe 2023 —
o ano mais recente, e sem VAB setorial.

**Tratamento.** Período de referência do PIB fixado em **2021**. Adicionalmente, o
cálculo do setor dominante passou a aplicar `coalesce` para zero antes de
`F.greatest()`, já que a função retorna nulo se qualquer argumento for nulo, e a
ordem dos testes foi invertida para que empate em zero não classifique
indevidamente como agropecuária.

**Aprendizado.** Este é o melhor exemplo do valor da verificação de completude
atributo a atributo. O valor especial atravessou três camadas sem gerar erro de
execução, e só apareceu como gráfico vazio na análise final. Se a análise fosse feita
diretamente sobre a Bronze, o setor dominante teria saído nulo sem ninguém perceber.

Uma verificação explícita de completude do VAB foi adicionada à camada Silver para
que o problema seja detectado na origem em execuções futuras.

---

## Análise de Dados

As respostas completas, com consultas, tabelas e visualizações, estão em
[`04_analise.ipynb`](notebooks/04_analise.ipynb).

### Perguntas propostas e respectivas respostas:

1. **Quais municípios concentram o maior volume de domicílios ainda não atendidos?**

Em termos da quantidade de domicílios não atendidos as Top 10 cidades seriam:

Porto Alegre - 78.861
Gravataí - 53.957
Viamão - 32.433
Pelotas - 31.074
Caxias do Sul - 28.402
Santa Maria - 28.348
Cachoeirinha - 26.601
Bagé - 22.797
Canoas - 21.619
Rio Grande - 20.956
Porém, todas essas são cidades com porte populacional acima de 100 mil habitantes, o que nós faz analisar em conjunto a taxa de penetração, onde

Porto Alegre - 88,53%
Gravataí - 54,80%
Viamão - 69,76%
Pelotas - 81,29%
Caxias do Sul - 87,12%
Santa Maria - 78,62%
Cachoeirinha - 56,11%
Bagé - 57,56%
Canoas - 86,10%
Rio Grande - 79,64%
Analisando os dois recortes, podemos ver que apesar de algumas cidades terem um maior número de domicílios pendentes de atendimento, isso não se reflete em mercado potencial de atenuação devido as altas taxas de ocupação.

Mas podemos destacar as cidades de Gravataí, Cachoerinha e Bagé que possuem uma taxa de penetração abaixo de 60% o qual ainda há espaço para atendimento.

O mercado não atendido do RS se distribui de forma desigual entre volume e percentual. Municípios de grande porte concentram o maior número absoluto de domicílios sem acesso, mesmo tendo penetração acima da média — o volume vem da escala, não da carência.

Já os municípios com menor penetração percentual tendem a ser de porte menor e localizados fora dos eixos metropolitanos, onde a oferta é mais rarefeita.

Para estratégia comercial, as duas leituras servem a propósitos diferentes: volume absoluto indica onde há mais assinantes possíveis; penetração baixa indica onde a concorrência ainda não consolidou posição.

2. **A penetração varia com o porte do município? Existe faixa populacional
   sistematicamente mal servida?**

Fica claro que quanto maior o porte populacional maior é a taxa de penetração agregada. Com esse viés municípios de até 20 mil habitantes possuem em média 53% de taxa de penetração.

O boxplot mostra tanto o nível quanto a dispersão. Faixas com mediana baixa e caixa estreita indicam carência sistemática; faixas com mediana alta e muitos outliers indicam heterogeneidade — alguns municípios bem servidos convivendo com outros mal atendidos no mesmo porte.

A comparação entre penetração agregada e média simples também é informativa: quando a agregada é maior, os municípios grandes da faixa puxam o resultado para cima, o que significa que a média simples descreve melhor o município típico.
   
3. **Como se distribui o número de provedores por município? Quantos são monopólio de
   fato?**
   
Não temos nenhum município que tenha somente um provedor de internet, então monopólio claramente declarado não há.

Frente a isso analisamos cidades com poucos provedores e alto share do líder, isso aparece somente em 7 municípios de pequeno porte, onde o share do líder médio é 86%. Podemos analisar ainda que municípios que possuam até 10 provedores tem um share do líder médio de 69,80%, que também consideramos como um alto índice de ocupação.
   
4. **Qual a participação de fibra óptica por município e por região? Quais municípios ainda
   dependem de rádio, satélite ou cobre?**
   
Hoje temos uma alta participação de fibra óptica nos municípios ocupando 81,11% dos atendimentos.

Poucos municípios, com menos de 5 mil habitantes possuem ainda pouca ocupação de fibra óptica com níveis de 14,02% até menos de 50%.

A diferença entre participação agregada e média simples por região indica se a fibra
está concentrada nos municípios maiores. Agregada muito acima da média significa que
os grandes centros puxam o número, enquanto o município típico da região tem cobertura
menor.

Dependência de satélite tende a indicar áreas remotas sem alternativa terrestre;
dependência de rádio indica cobertura de baixo custo em áreas rurais ou periféricas;
cabo metálico indica infraestrutura legada de operadora incumbente.
   
5. **Quais municípios têm a base de velocidade mais defasada — potencial de upgrade
   de plano?**
   
93,69% já estão classificados como sendo acima de 34Mbps, o que denota de uma alta ocupação com o topo da faixa medida.

6. **A penetração se relaciona com PIB per capita ou com o setor econômico
   dominante?**
   
Percebemos que a distinção entre os setores econômicos é pequena, com uma variação de 2% entre os setores de Indústria, Administração Pública e Serviços. Somente o setor dominante de Agropecuária fica abaixo com 51,07% de penetração, e isso faz sentido pelo atuação de atendimento no campo ser mais restrito ao atendimento com fibra óptica.

Um coeficiente de correlação próximo de zero não significa ausência de relação —
pode indicar relação não linear, ou que outros fatores dominam. O gráfico de dispersão
em escala logarítmica ajuda a distinguir os dois casos.

Uma ressalva metodológica importante: o PIB per capita municipal é sensível a
distorções. Municípios pequenos com uma grande planta industrial ou usina apresentam
PIB per capita altíssimo sem que a renda das famílias acompanhe. Isso enfraquece a
variável como proxy de poder aquisitivo domiciliar.

Sobre o setor dominante, a diferença entre média e mediana revela se o resultado é
puxado por poucos casos extremos.

7. **Considerando mercado não atendido, defasagem tecnológica e poder aquisitivo,
   quais municípios apresentam maior oportunidade comercial?**
   
Considerando as análises realizadas, podemos observar que as oportunidades estão concentradas em municípios de pequeno porte de até 5 mil habitantes, pois possuem menos de 10 provedores ativos. 

---

### Discussão geral

Observamos que o mercado do Rio Grande do Sul já está saturado, regiões metropolitanas tem um alto índice de provedores para fornecimento de internet, assim como este mesmo range possui uma alta penetração de atendimento.
Há poucas oportunidades de construção ou atendimento de novas cidades, as oportunidades concentram-se em municípios pequenos de até 5 mil habitantes, mas mesmo assim esses já possuem um alto índice de atuação de empresas, normalmente tendo até 10 provedores com atendimento.

---

### Limitações da análise

**Recorte temporal único.** Todas as métricas são um retrato de julho de 2026. Sem
série histórica não é possível medir crescimento, entrada e saída de prestadoras, nem
migração tecnológica.

**Denominador de mercado.** Domicílios recenseados do Censo 2022 é o melhor proxy
público disponível, mas não equivale a domicílios passíveis de conexão — não existe
dado público de capacidade instalada de rede por município.

**Grupo econômico incompleto.** A Anatel classifica todos os provedores regionais
como `OUTROS`. A análise competitiva usa CNPJ, o que resolve o problema, mas impede
consolidar prestadoras do mesmo dono sob marcas diferentes.

**Defasagem do PIB.** O PIB municipal usa 2021 como referência, contra Censo de 2022 e
Anatel de julho de 2026. A comparação assume estabilidade relativa entre municípios —
razoável para ordenação e percentil, menos para valores absolutos.

**PIB per capita como proxy de renda.** Municípios pequenos com uma grande planta
industrial apresentam PIB per capita alto sem que a renda das famílias acompanhe. A
variável mede produção no território, não poder aquisitivo domiciliar.

**Local de declaração.** A Anatel registra o município do acesso conforme declarado
pela prestadora. Não há como validar externamente se a declaração corresponde ao
endereço do assinante.

---

## Autoavaliação

Os Objetivos propostos pelo trabalho foram alcançados, as 7 perguntas foram trabalhadas 
e respondidas através da modelagem realizada, com o decorrer do trabalho houveram algumas 
dificuldades elencadas melhor abaixo, que acabaram enriquecendo o trabalho ao longo de 
sua constituição. Ainda há a proposição de trabalhos futuros visando melhorar e expandir
as análises realizadas.

### Dificuldades encontradas

**Restrição de rede do Databricks Free Edition.** A plataforma limita o acesso de
saída à internet a domínios confiáveis, o que inviabiliza chamadas a APIs externas
por padrão. A solução — verificação de identidade via LinkedIn — não é óbvia e exigiu
leitura da documentação de limitações. Foi o primeiro obstáculo do projeto, e sem
resolvê-lo nada mais seria possível de forma programática.

**Restrições do formato Delta em nomes de coluna.** O Delta rejeita espaços e
acentos em nomes de coluna sem habilitar column mapping. Como a Anatel usa nomes como
`Código IBGE Município`, foi necessário normalizar para snake_case já na camada
Bronze — uma alteração técnica que precisou ser documentada para não conflitar com o
princípio de que a Bronze preserva o dado como veio.

**Dependência funcional assumida sem verificação.** A suposição de que tecnologia
determinaria o meio de acesso parecia óbvia e estava errada. O erro só apareceu na
validação de integridade referencial, depois que o modelo dimensional já estava
construído. O retrabalho foi significativo e a lição, direta: verificar dependências
empiricamente antes de modelar.

**Valores especiais silenciosos.** O caso do VAB setorial atravessou três camadas sem
gerar erro de execução. Não houve exceção, não houve aviso — apenas um gráfico vazio
no final. Ilustra por que a verificação de completude atributo a atributo é etapa
obrigatória, e não formalidade.

**Escolha do denominador de mercado.** A primeira versão usava domicílios ocupados
como denominador de penetração, o que parecia a escolha natural. Só a investigação
dos outliers revelou que a definição do Censo exclui imóveis de uso ocasional — e que
essa exclusão distorce sistematicamente municípios de veraneio. A correção exigiu
ingerir uma fonte adicional que não estava no plano original.

### Trabalhos futuros

**Série histórica.** A extensão mais valiosa. A Anatel publica desde 2007, e os
arquivos já estão disponíveis. Com histórico seria possível medir crescimento por
município, entrada e saída de prestadoras, migração de tecnologia, e o efeito da
consolidação de grupos econômicos sobre a concorrência. Exigiria adicionar
`dim_tempo` e implementar SCD Tipo 2 nas dimensões de empresa e município.

**Expansão nacional.** O pipeline é parametrizado por UF; trocar o filtro traz o
Brasil inteiro. Permitiria comparar o RS com outros estados e contextualizar os
resultados.

**Denominador corporativo.** A penetração corporativa não é normalizada por falta de
denominador equivalente. O Cadastro Central de Empresas do IBGE, que traz unidades
locais por município, resolveria.

**Custo de rede.** O índice de oportunidade não considera custo de implantação, que
varia com densidade, relevo e dispersão dos domicílios. Dados de altimetria e de
malha viária poderiam alimentar um componente de viabilidade.

**Ponderação empírica do índice.** Os três componentes têm peso igual por escolha
neutra declarada. Com série histórica seria possível calibrar pesos observando quais
municípios de fato receberam expansão de rede — e aí o índice deixaria de ser
descritivo para virar preditivo.

**Automação.** Os notebooks rodam manualmente em sequência. Um Job do Databricks com
as quatro tarefas encadeadas permitiria atualização mensal automática, acompanhando a
publicação da Anatel.

---
