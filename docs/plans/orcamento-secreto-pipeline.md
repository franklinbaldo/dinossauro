# Plano: Pipeline do Orçamento Secreto (Cross-referencing)

**Status:** v1 — plano inicial de pipeline
**Escopo:** estabelecer infraestrutura de dados para o rastreio do "orçamento secreto", cruzando promessas políticas de campanhas/ads com a execução real (via baliza/PNCP e transferências)
**Estimativa:** 4–6 semanas (~10 PRs incrementais)

---

## § Problema e Contexto

O projeto "dinossauro" atua sob a premissa de que políticos fazem promessas em campanhas (marketing político, redes sociais, propaganda direcionada) e a execução do orçamento muitas vezes não é transparente, configurando um "orçamento secreto". O objetivo é "exumar" essas discrepâncias.

**Não é o foco**: denúncias automatizadas, painéis de investigação jornalística manual extensiva.
**Foco**: a INFRAESTRUTURA de dados robusta e analítica (Civil Tech) que unifica bases heterogêneas sob uma ontologia comum e possibilita consultas.

A stack foca na robustez para o futuro e simplicidade na operação:
- **Web**: Curva & Concreto (cobogó como Design System).
- **Dados**: formato Parquet persistido no Internet Archive (via pipeline em Python, inspirado no Baliza).
- **Inteligência**: LLM-first (identidade de projeto → claim).

Repositórios relacionados no ecossistema:
- `franklinbaldo/baliza` (reuso de infra e dados do PNCP)
- `franklinbaldo/cobogo` (Design System Curva & Concreto)
- `franklinbaldo/causaganha` (referência para automação e extração civil tech)

---

## § Fontes de Dados

1. **Execução Orçamentária**
   - **PNCP (Portal Nacional de Contratações Públicas)**: Fonte da execução real na ponta. Aproveitar-se do ecossistema de recursos maduros do `baliza` (via consumo dos artefatos Parquet do Baliza).
   - **Portal da Transparência (Transferências Federais)**: Principalmente para rastreio de transferências fundo a fundo, convênios (Plataforma +Brasil) e emendas Pix.

2. **Câmara/Senado APIs (Emendas)**
   - Extração do fluxo legislativo das emendas: autor (parlamentar), valor, beneficiário, status de execução, e tipo de emenda (RP9, RP8, RP6, RP7).

3. **Marketing Político (Promessas & Claims)**
   - **AdLibrary (Meta/Facebook)** / **WhereSpentMyMoney**: Coleta de anúncios focados em entregas e promessas regionais.
   - **TSE (Prestação de Contas)**: Despesas de campanhas (pagamentos a produtoras, agências) e promessas oficiais/planos de governo.
   - **Twitter/X (opcional)**: Fontes secundárias se a API ou scrapers se mostrarem viáveis/com baixo risco.

---

## § Modelo de Dados (Ontologia)

Inspirado na hierarquia de `PNCPResource` do `baliza`, a arquitetura do dinossauro será dividida por recursos de domínio. Cada camada tem identidade própria.

### Hierarquia Lógica

```
EmendaParlamentar          ← A proposição orçamentária do parlamentar
  ├── Origem (Autor)       ← Identidade do parlamentar, base TSE/Câmara
  ├── Beneficiario         ← Identidade do município ou ONG/entidade recebedora
  └── Projeto (Claim-core) ← O objeto do gasto pretendido

ExecucaoReal               ← O dinheiro sendo gasto de fato
  ├── Transferencia        ← O repasse do governo federal (Portal Transparência)
  └── Contrato/Compra      ← A ponta (PNCP/baliza)

MarketingClaim             ← O que foi falado para o eleitorado
  ├── Campanha/Ad          ← AdLibrary Meta, panfletos digitais
  └── Entidade Prometida   ← Link com o Projeto/Emenda
```

### Mapping de Campos (Schema)

A tabela de junção e resolução de identidades ("Mesa de Cruzamento") mapeia as pontas das três pirâmides:

| Tabela (Artefato) | Campo | Tipo DuckDB | Origem Principal | Descrição |
|---|---|---|---|---|
| `emendas_canonical` | `id_emenda` | VARCHAR | API Câmara | PK da emenda |
| `emendas_canonical` | `cpf_autor` | VARCHAR | API Câmara/TSE | O parlamentar |
| `emendas_canonical` | `cnpj_beneficiario` | VARCHAR | API Câmara | O município/entidade |
| `execucao_canonical` | `id_emenda` | VARCHAR | Portal Transp. | FK para rastrear a origem |
| `execucao_canonical` | `cnpj_fornecedor` | VARCHAR | PNCP / baliza | Quem executou a obra |
| `execucao_canonical` | `valor_executado` | DOUBLE | PNCP / baliza | Montante real pago |
| `marketing_canonical`| `id_ad` | VARCHAR | Meta AdLibrary | PK do claim |
| `marketing_canonical`| `cpf_autor` | VARCHAR | Meta AdLibrary | Pagador do Ad (parlamentar) |

### Pseudocódigo de Identidade

**Python (Pipeline de Dados / LLM)**:

```python
# LLM-first extraction pipeline example
def extract_claim(ad_text: str) -> dict:
    prompt = f"Extraia o município e o tipo de obra prometida neste Ad: '{ad_text}'"
    return llm.invoke(prompt)

def match_claim_to_execution(claim_data: dict, db_connection) -> list:
    return db_connection.execute("""
        SELECT e.id_emenda, ex.valor_executado
        FROM emendas e
        JOIN execucao ex ON e.id_emenda = ex.id_emenda
        WHERE e.municipio = ? AND e.tipo_obra = ?
    """, (claim_data["municipio"], claim_data["tipo_obra"])).fetchall()
```

**TypeScript (Frontend Schema)**:

```typescript
// Astro / Schema definition for UI
export interface CrossReferenceRow {
  parlamentarNome: string;
  promessaAdUrl: string;
  valorPrometido: number;
  valorExecutadoPNCP: number;
  status: 'Confirmado' | 'Divergente' | 'Não Executado';
}
```

---

## § Sequência Incremental de PRs

A infraestrutura será construída incrementalmente. A meta é ter algo testável a cada merge.

| PR | Descrição | Escopo |
|---|---|---|
| **PR 0** | Scaffold inicial | Setup Astro, Tailwind/Cobogó para o front, e setup Python (UV, Ruff, Pytest) para o pipeline de dados. Sem dados reais, apenas "Hello World". |
| **PR 1** | Ingestão: Emendas | Script em Python que extrai da Câmara/Senado, normaliza e persiste no IA (Parquet). |
| **PR 2** | Ingestão: Transferências | Pipeline similar para extrair dados do Portal da Transparência (transferências fundo a fundo). |
| **PR 3** | Ingestão: PNCP (via Baliza) | Leitura dos Parquets consolidados pelo `baliza` para a "ponta da lança". |
| **PR 4** | Ingestão: Meta Ads | Integração com AdLibrary/WhereSpentMyMoney para coletar claims. |
| **PR 5** | Modelagem da Tabela Canônica | Script DuckDB/PyArrow juntando as fontes na tabela `CrossReference` (resolvendo CPFs/CNPJs). |
| **PR 6** | Pipeline de Inteligência | Script Python + LLM rodando offline sobre o parquet para normalizar o "Projeto" entre emendas e ads. |
| **PR 7** | Frontend: Infra de Dados | Setup do Astro lendo Parquets do Internet Archive (Server-side query/WASM). |
| **PR 8** | Frontend: UI Base (Cobogó) | Layout Curva & Concreto, navegação e componentes genéricos. |
| **PR 9** | Frontend: Views e Viz | View por "Parlamentar", "Município" e a visualização do cross-reference. |
| **PR 10** | Integração Contínua | Actions do Github para o pipeline semanal e deploy. |

---

## § Frontend & Visualização

- **Framework**: Astro (SSG ou SSR híbrido).
- **Design System**: O frontend usará o repositório `franklinbaldo/cobogo` ou adotará os princípios do modernismo brasileiro (tipografia marcante, cores neutras com blocos de contraste).
- **Páginas previstas**:
  - `/parlamentar/[id]`: O deputado X prometeu (ads) vs o que destinou (emendas) vs o que chegou na ponta (PNCP/baliza).
  - `/municipio/[id]`: Quais emendas caíram ali, se viraram obras reais e quem levou o crédito.
  - `/programa/[id]`: Visão agregada (Ex: "Asfalto", "Saúde").

---

## § Riscos e Mitigações

| Risco | Mitigação |
|---|---|
| **Marketing data é instável** (Rate limits de APIs do TSE e Meta Ads, bans) | Persistir checkpoints diários; fallbacks (usar última partição boa); não estourar quotas. |
| **LLM falhar ao relacionar Ad com Obra** | O cruzamento deve ter um "Confidence Score" (Baixo, Médio, Alto). O front expõe a heurística, não uma "verdade absoluta". |
| **Mudanças no formato do Portal da Transparência/Emendas** | Monitoramento de schema e testes nos PRs de ingestão. |
| **Dependência do Internet Archive** | Edge Caching (Cloudflare) na frente do IA ou empacotar em SQLite/DuckDB local agregados menores. |

---

## § Não-Objetivos

- Investigação jornalística humana automatizada.
- Processos judiciais ou geração de dossiês para órgãos de controle.
- Tooling de denúncia (não haverá formulários para usuários submeterem crimes).
- O foco é exclusivamente na **INFRAESTRUTURA** de dados que possibilita análise posterior.
