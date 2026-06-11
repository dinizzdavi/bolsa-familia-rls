# Painel Gerencial — Programa Bolsa Família com Row Level Security (RLS)

Projeto da disciplina **Business Intelligence II** (CEUB). Painel gerencial dos repasses do
**Programa Bolsa Família** com **controle de acesso por região** via **Row Level Security (RLS)**
no Power BI, em dois cenários: **RLS estático** (uma função por região) e **RLS dinâmico**
(função única com `USERPRINCIPALNAME()`).

---

## 1. Como abrir o projeto

Formato **Power BI Project (`.pbip`)**.

1. Power BI Desktop recente. Se a opção `.pbip` não existir: **Arquivo → Opções → Recursos de versão prévia → "Salvar como projeto do Power BI (.pbip)"**.
2. Abra **`BolsaFamilia_RLS.pbip`**.
3. **Ajuste o caminho dos dados** (parâmetro **`CaminhoArquivo`** em **Transformar dados → Gerenciar parâmetros**) apontando para o CSV do Bolsa Família na sua máquina.
4. **Página Inicial → Atualizar**.

> A base completa (~204 MB / 2,3 milhões de linhas) não é versionada no GitHub (limite de 100 MB).
> Os números deste README referem-se à base completa.

---

## 2. Visão Geral e Política de Acesso

| Perfil | Acesso |
|---|---|
| **Admin** | Todos os dados do Brasil |
| **Gestor Regional** | Apenas os dados da sua região geográfica |

### Números da base (competência 202601)
| Indicador | Valor |
|---|---|
| Total de Repasses | R$ 1.550.985.743,00 |
| Total de Beneficiários (NIS distintos) | 2.293.602 |
| Total de Municípios atendidos | 5.556 |
| Ticket Médio (por beneficiário) | R$ 676,22 |
| Linhas (benefícios pagos) | 2.324.900 |
| UFs presentes | 27 (todas) |

---

## 3. Modelo Semântico

Esquema **floco de neve**: 1 fato + 2 dimensões de apoio.

### 3.1 Tabelas
**`fato_bolsa_familia`** (importada do CSV via Power Query)
| Coluna | Tipo | Observação |
|---|---|---|
| MÊS COMPETÊNCIA | Texto | AAAAMM |
| ANO | Texto | derivada (4 primeiros dígitos) |
| MÊS / MES | Texto | derivada (2 últimos dígitos) |
| UF | Texto | chave → `dim_regioes` |
| CÓDIGO MUNICÍPIO SIAFI | Texto | usada em "Total de Municípios" |
| NOME MUNICÍPIO | Texto | |
| CPF FAVORECIDO | Texto | mascarado |
| NIS FAVORECIDO | Texto | usada em "Total de Beneficiários" |
| NOME FAVORECIDO | Texto | |
| VALOR PARCELA | Decimal | convertida com localidade pt-BR (vírgula) |

**`dim_regioes`** — `Regiao`, `UF` (tabela de apoio).
**`dim_usuarios_rls`** — `email_usuario`, `regiao_permitida` (base do RLS dinâmico).

| email_usuario | regiao_permitida |
|---|---|
| gestor.norte@ministerio.br | Norte |
| gestor.nordeste@ministerio.br | Nordeste |
| gestor.centroeste@ministerio.br | Centro-Oeste |
| gestor.sudeste@ministerio.br | Sudeste |
| gestor.sul@ministerio.br | Sul |

### 3.2 Relacionamentos
| De (muitos) | Para (um) | Cardinalidade | Filtro cruzado |
|---|---|---|---|
| `fato_bolsa_familia[UF]` | `dim_regioes[UF]` | \*:1 | Única |
| `dim_regioes[Regiao]` | `dim_usuarios_rls[regiao_permitida]` | \*:1 | Única |

Cadeia do filtro de segurança: `dim_usuarios_rls → dim_regioes → fato_bolsa_familia`.

---

## 4. Power Query (tratamentos)

- **VALOR PARCELA**: convertida para Número Decimal **usando localidade Português (Brasil)** (o valor vem como `"1060,00"`, com vírgula decimal).
- **MÊS COMPETÊNCIA** (AAAAMM): separada em **ANO** (`Text.Start([MÊS COMPETÊNCIA],4)`) e **MES** (`Text.End([MÊS COMPETÊNCIA],2)`), ambas como texto.
- **dim_regioes** e **dim_usuarios_rls** criadas como tabelas de apoio.

---

## 5. Medidas DAX

```DAX
Total de Repasses = SUM ( fato_bolsa_familia[VALOR PARCELA] )

Total de Beneficiários = DISTINCTCOUNT ( fato_bolsa_familia[NIS FAVORECIDO] )

Total de Municípios = DISTINCTCOUNT ( fato_bolsa_familia[CÓDIGO MUNICÍPIO SIAFI] )

Ticket Médio = DIVIDE ( [Total de Repasses], [Total de Beneficiários] )
```

> Ticket Médio = valor médio por beneficiário.

---

## 6. Dashboard

**Página 1 — Visão Geral Nacional**: 4 cartões (Repasses, Beneficiários, Ticket Médio, Municípios),
gráfico de barras de Repasses por Região e segmentações por ANO e MES.

**Página 2 — Detalhe por Região/Estado**: tabela UF × (Municípios, Beneficiários, Repasses, Ticket Médio),
gráfico de barras Top 10 municípios por valor de repasse e segmentação por Região.

Tema visual dark personalizado aplicado (`Tema_BolsaFamilia_Dark.json`).

---

## 7. Row Level Security

### 7.1 RLS Estático (filtro em `dim_regioes[Regiao]`)
| Função | Filtro DAX |
|---|---|
| `Admin` | *(sem filtro — acesso total)* |
| `Gestor_Norte` | `[Regiao] = "Norte"` |
| `Gestor_Nordeste` | `[Regiao] = "Nordeste"` |
| `Gestor_CentroOeste` | `[Regiao] = "Centro-Oeste"` |
| `Gestor_Sudeste` | `[Regiao] = "Sudeste"` |
| `Gestor_Sul` | `[Regiao] = "Sul"` |

### 7.2 RLS Dinâmico — `Gestor_Regional_Dinamico` (filtro em `dim_usuarios_rls`)
```DAX
[email_usuario] = USERPRINCIPALNAME()
```

### 7.3 Teste local
**Modelagem → Exibir como**. Para o dinâmico, usar **Outro usuário** com um e-mail
(ex.: `gestor.sul@ministerio.br`).

---

## 8. Publicação (Power BI Service)
1. **Publicar** o relatório.
2. Conjunto de dados → **Segurança** (cadeado).
3. RLS estático: cada gestor → função da sua região.
4. RLS dinâmico: todos os gestores → `Gestor_Regional_Dinamico`.
5. `pedro.mpereira@ceub.edu.br` → função **`Admin`**.

---

## 9. Estrutura do repositório
```
BolsaFamilia_RLS.pbip                 Arquivo de projeto (abrir este)
BolsaFamilia_RLS.SemanticModel/       Modelo (tabelas, medidas, relacionamentos, RLS)
BolsaFamilia_RLS.Report/              Relatório (2 páginas)
Tema_BolsaFamilia_Dark.json           Tema visual dark personalizado
README.md                             Esta documentação
```

## 10. Referências
- Microsoft — *Row-level security (RLS) with Power BI*
- Microsoft — *USERPRINCIPALNAME (DAX)*
- Portal de Dados Abertos — MDS, *Bolsa Família – Pagamentos*
