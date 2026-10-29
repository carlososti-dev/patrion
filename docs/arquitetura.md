# Patrion: proposta de stack e arquitetura (v2, stack Python/FastAPI)

Objetivo: SaaS de gestão de bens patrimoniais para prefeituras, com o escopo do protótipo Lovable como referência, cobrindo as lacunas já levantadas (login, multi-tenant, tipos corretos, depreciação, inventário, exigências do Tribunal de Contas).


## 1. Stack (decidida: padrão dos outros projetos da equipe)

| Camada | Escolha | Por quê |
|---|---|---|
| Backend | Python 3.12 + FastAPI | Stack que a equipe já usa; OpenAPI automático. |
| Validação | Pydantic v2 | Schemas de entrada/saída; `Decimal` para valores. |
| ORM / migrações | SQLAlchemy 2 + Alembic | Migrações versionadas, funciona bem com RLS. |
| Banco | PostgreSQL 16 | `numeric`/`date` corretos, Row-Level Security por prefeitura, JSONB. |
| Autenticação | Sessão/JWT própria com senha em Argon2 e 2FA (TOTP); Authlib para OIDC (gov.br depois) | Sem dependência de serviço pago. |
| Jobs | Procrastinate (fila no próprio Postgres) | Depreciação mensal e relatórios pesados sem Redis no início. |
| Relatórios PDF | Jinja2 + WeasyPrint | Termos de responsabilidade e relatórios com layout fiel. |
| Arquivos | Armazenamento compatível com S3 (MinIO local, S3/Magalu Cloud em produção) | Fotos, notas fiscais, termos assinados. |
| Frontend | React + Vite + TypeScript + Tailwind + shadcn/ui + TanStack Query/Router | Mesma base visual do protótipo Lovable. |
| Contrato front/back | Cliente TypeScript gerado do OpenAPI do FastAPI (orval) | Tipos sempre sincronizados entre API e telas. |
| Repositório | `backend/` (uv, ruff, pytest) e `frontend/` (pnpm, Vitest, Playwright) | |
| Infra / CI | Docker Compose, GitHub Actions; hospedagem em região Brasil | LGPD e preferência em contratos públicos. |

## 2. Multi-tenant (uma prefeitura = um tenant)

- Banco único, esquema único, coluna `tenant_id` em todas as tabelas de negócio.
- **Row-Level Security no Postgres**: cada requisição abre transação com `SET app.tenant_id = ...`; as políticas impedem ler ou gravar dados de outra prefeitura mesmo se o código esquecer um filtro.
- Hierarquia dentro do tenant: Prefeitura → Órgão/Secretaria → Unidade administrativa → Local físico.
- Um usuário pode pertencer a mais de uma prefeitura (ex.: contador terceirizado), com papel por vínculo.
- Caminho de evolução: se um cliente exigir isolamento físico, o mesmo código roda com banco dedicado para ele.

## 3. Autenticação e autorização

- Login por e-mail/senha com 2FA opcional, recuperação de senha, bloqueio após tentativas.
- Papéis por tenant: **Administrador**, **Gestor de Patrimônio**, **Responsável de Unidade**, **Comissão de Inventário**, **Consulta/Auditor** (somente leitura, para controle interno e TCE).
- Permissões checadas na API (dependências do FastAPI) e escopo por unidade quando aplicável.
- **Trilha de auditoria** imutável: quem, quando, o quê, valores antes/depois, IP.

## 4. Modelo de dados (núcleo)

Valores sempre `numeric(15,2)`, datas `date`/`timestamptz`, nada de texto livre para isso.

- `tenants`, `users`, `memberships` (user × tenant × papel)
- `orgaos`, `unidades`, `locais`, `servidores_responsaveis`
- `contas_contabeis` (PCASP, grupo 1.2.3.1.x bens móveis / 1.2.3.2.x imóveis), com vida útil e valor residual padrão por conta
- `fornecedores`
- `bens`: nº de tombamento (único por tenant), descrição, conta contábil, tipo (móvel/imóvel), forma de ingresso (compra, doação, cessão, comodato, fabricação própria), data de aquisição, valor de aquisição, empenho, nota fiscal, fornecedor, vida útil, valor residual, estado de conservação, situação (ativo, em manutenção, cedido, baixado), local e responsável atuais
- `movimentacoes`: transferência entre unidades/locais/responsáveis, com termo de responsabilidade
- `baixas`: motivo (alienação, inservível, extravio, furto, doação), processo administrativo, data, valor contábil na baixa
- `depreciacoes`: lançamento mensal por bem (método linear, conforme MCASP / NBC TSP 07), valor do mês e acumulado; fechamento mensal trava o período
- `reavaliacoes` (pós-MVP, mas tabela já prevista)
- `inventarios`, `inventario_itens`: comissão, período, escopo, item encontrado/não encontrado/divergente, foto
- `anexos` (polimórfico: bem, movimentação, baixa, inventário)
- `audit_log`

## 5. Módulos do MVP

1. **Acesso e prefeituras**: cadastro do tenant, usuários, papéis, auditoria.
2. **Estrutura organizacional**: órgãos, unidades, locais, responsáveis, contas contábeis.
3. **Incorporação de bens**: cadastro individual e em lote (mesma NF), geração de tombamento, etiqueta com QR Code, fotos e anexos.
4. **Movimentação**: transferências e emissão de termo de responsabilidade em PDF.
5. **Baixa** de bens com processo e motivo.
6. **Depreciação mensal** automática e fechamento de mês.
7. **Inventário**: abertura por comissão, conferência por leitura de QR no celular (PWA, funciona offline), relatório de divergências.
8. **Relatórios**: inventário analítico e sintético por conta contábil (conciliação com a contabilidade), bens por unidade/responsável, depreciação do período, movimentações e baixas. Exportação CSV/PDF.

**Depois do MVP**: integração com o sistema contábil (SIAFIC), layouts específicos de prestação de contas por TCE estadual, login gov.br, bens imóveis com georreferenciamento, reavaliação/impairment, app nativo.

## 6. Arquitetura em uma frase

Monólito modular (API FastAPI com um pacote por domínio acima) + SPA React, sobre um Postgres com RLS. Simples de operar com equipe pequena, e cada módulo pode virar serviço separado no futuro se for preciso.
