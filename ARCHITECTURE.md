# Arquitetura — Resort Platform

Este documento aprofunda as decisões técnicas do Resort Platform. Para visão geral de produto, funcionalidades e stack, ver [README.md](./README.md).

---

## Arquitetura

Monorepo Turborepo com duas aplicações Next.js independentes, compartilhando banco e regras de negócio via packages internos — nunca comunicação HTTP direta entre elas:

```
                 ┌────────────────────┐        ┌────────────────────┐
                 │   apps/public       │        │   apps/admin        │
                 │  (hóspede, sem auth)│        │ (funcionário, auth) │
                 └──────────┬──────────┘        └──────────┬───────────┘
                            │  Server Components/Route Handlers
                            │  chamam packages/api em processo
                            ▼                               ▼
                      ┌─────────────────────────────────────────┐
                      │              packages/api                │
                      │  services (regras de negócio) + handlers │
                      └───────────────────┬───────────────────────┘
                                           ▼
                      ┌─────────────────────────────────────────┐
                      │           packages/database               │
                      │        Prisma Client + schema              │
                      └───────────────────┬───────────────────────┘
                                           ▼
                                   PostgreSQL (Neon)
```

`packages/shared` (schemas Zod, enums, constantes, utils) não depende de nada e é a base de tudo — inclusive dos dois apps, para tipagem e validação client-side. Regra de fronteira: Client Components nunca importam `packages/api` nem `packages/database` (dependem de Prisma/Node APIs que não existem no browser) — sempre fazem `fetch()` para o Route Handler local do próprio app. Server Components e Route Handlers, por rodarem no servidor, importam `packages/api` diretamente, sem HTTP interno.

### Por que dois apps separados em vez de um único com áreas públicas/admin

- Superfícies de risco diferentes: o portal público nunca tem sessão autenticada; separar fisicamente elimina uma classe inteira de erro de "vazei uma rota admin sem middleware".
- Deploys independentes: um incidente ou rollback no Admin não deveria nunca poder derrubar o portal do hóspede.
- Sistemas de design propositalmente distintos (o público é mobile-first orientado a descoberta; o admin é um dashboard denso).

---

## Multi-tenancy e isolamento

Toda entidade do sistema, exceto `Hotel`, pertence a exatamente um `hotelId`. As garantias:

1. **Toda query** que recebe um identificador de entidade relacionada verifica que ela pertence ao `hotelId` do contexto atual antes de ler ou escrever.
2. Quando não pertence — **inclusive quando a entidade existe, só que em outro hotel** — a resposta é sempre `404 NotFoundError`, nunca `403 PermissionError`. Isso é deliberado: distinguir os dois erros permitiria a um usuário malicioso confirmar, por tentativa e erro, que um determinado ID existe em algum hotel — mesmo sem acessar seu conteúdo (proteção contra enumeração de IDs / IDOR).
3. Role e isolamento de tenant são **duas checagens independentes**, nessa ordem: primeiro a permissão da operação, depois o pertencimento ao hotel. Uma tentativa de ação por um usuário sem permissão suficiente é barrada antes mesmo de o sistema confirmar se a entidade em questão existe — evitando que a tentativa de ação revele informação sobre a existência do recurso.
4. `hotelId`, `userId` e role nunca vêm do corpo da requisição — sempre resolvidos a partir da sessão autenticada (Admin) ou do slug da URL (portal público).

---

## Mapas e roteamento

- Coordenadas de locais e pontos de navegação são `Float` normalizados entre `0.0` e `1.0`, representando posição proporcional na imagem do mapa — nunca pixel, nunca coordenada GPS. Isso garante que marcadores e rotas funcionem em qualquer resolução de tela sem recalcular nada no backend.
- A malha de navegação é modelada como grafo: pontos (nós) e caminhos (arestas bidirecionais) desenhados visualmente pelo funcionário sobre o mapa. A distância de cada caminho é sempre derivada das coordenadas dos dois pontos que conecta — nunca um valor digitado manualmente, e é recalculada automaticamente sempre que um dos pontos é movido.
- O cálculo de menor caminho usa Dijkstra sobre esse grafo. A escolha do algoritmo é adequada ao problema: pesos não-negativos (distâncias), grafo pequeno (dezenas a poucas centenas de nós por resort), e sem heurística admissível de "distância até o destino" pronta que justificasse A* — a diferença de custo computacional entre os dois é irrelevante nesse volume.
- Rotas sugeridas agrupam eventos publicados do dia por proximidade de horário/local e são sempre recalculadas em tempo real, nunca persistidas — o estado "o que está acontecendo agora" muda o tempo todo e não deveria nunca ficar desatualizado em cache implícito no banco.

---

## Pipeline de importação de PDF (LLM + revisão humana + publicação em massa)

```
Upload do PDF
   │
   ▼
Registro da sessão de importação criado no banco (status: PROCESSING)
   │
   ▼
PDF enviado a um LLM externo com um contrato de saída fixo:
   cada campo extraído (data, horário, nome, local, categoria, descrição)
   vem acompanhado de um nível de confiança (HIGH / MEDIUM / LOW / FAILED)
   │
   ▼
Eventos extraídos ficam disponíveis para revisão — campos com confiança
baixa ou falha são destacados na UI para correção manual
   │
   ▼
Funcionário confirma (com correções, se necessário)
   │
   ▼
Eventos reais são criados com status Draft
   │
   ▼
Cada evento criado pode ser editado individualmente via modal
(foto, descrição, ajuste de horário/local, etc.)
   │
   ▼
Funcionário seleciona os eventos que deseja publicar e confirma
a publicação em massa dos selecionados de uma só vez
```

Pontos de design deliberados:

- **O contrato de saída do LLM é fixo e desacoplado do provedor** — o adapter que fala com o LLM é intercambiável sem alterar o restante do pipeline.
- **Nenhum conteúdo é publicado automaticamente.** A confirmação humana é uma etapa obrigatória antes de qualquer evento existir de fato no sistema; mesmo depois de confirmado, o evento nasce em `Draft`.
- Uma correção manual em qualquer campo é tratada como fonte de confiança máxima — o campo corrigido deixa de ser sinalizado como pendente de revisão.
- **A etapa de publicação em massa** existe porque, após uma importação, é comum vários eventos precisarem do mesmo tipo de ajuste (foto, descrição) antes de irem ao ar — em vez de publicar um por um, o funcionário edita o que for necessário e publica todos os selecionados numa única ação.

---

## Estrutura do monorepo (referência)

```
resort-platform/
├── apps/
│   ├── public/                 # Portal do hóspede — sem auth
│   │   └── app/[hotelSlug]/    # mapa, roteiro, rotas
│   └── admin/                  # Painel administrativo — autenticado
│       └── app/(dashboard)/    # dashboard, locais, eventos, roteiros,
│                                # mapas, navegação, importação
│
├── packages/
│   ├── shared/                 # Schemas Zod (DTOs), enums, constantes, utils
│   ├── database/                # Schema Prisma, client singleton, seed
│   └── api/                     # Services (regra de negócio), handlers,
│                                 # middleware (auth/rbac/isolamento),
│                                 # navegação (Dijkstra), integração LLM
│
├── turbo.json
└── package.json                 # workspaces
```

`packages/shared` não depende de nada. `packages/database` depende só de `packages/shared`. `packages/api` depende de `packages/database` e `packages/shared`. Nenhum package depende dos apps — o fluxo de dependência é sempre de fora (apps) para dentro (packages).
