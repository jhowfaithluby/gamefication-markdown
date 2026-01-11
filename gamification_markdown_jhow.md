# Plataforma de Gamificação - Especificação Técnica 

## Visão Geral

Plataforma de gamificação escalável, confiável e segura desenvolvida em .NET 10 e C# 13, seguindo padrões de clean architecture. Atende aplicações Web e móveis através de APIs HTTP e serviços assíncronos.

### Objetivos Principais
- Arquitetura em camadas com separação clara de responsabilidades
- API única com áreas públicas e administrativas
- Alta disponibilidade e tolerância a falhas
- Observabilidade completa com logs estruturados e métricas
- Segurança em todos os níveis

---

## Stack Tecnológico

### Backend
- **Framework**: .NET 10 (ASP.NET Core)
- **Linguagem**: C# 13
- **ORM**: Entity Framework Core 10
- **Banco de Dados**: PostgreSQL com replicação/sincronização
- **Cache**: Redis (cluster/Sentinel para HA)
- **Message Broker**: RabbitMQ (quorum queues e cluster)
- **Documentação**: OpenAPI/Swagger

### Infraestrutura
- **Load Balancer**: Nginx / Ingress Controller (Kubernetes)
- **Containerização**: Docker (multi-stage builds)
- **Orquestração**: Docker Compose (dev) / Kubernetes (prod)
- **Object Storage**: Cloudflare R2 / Amazon S3
- **Secrets Management**: Azure Key Vault / AWS Secrets Manager

---

## Topologia de Rede

```
Internet → FRONTs (Web/Mobile)
    ↓
DMZ
    └─ Load Balancer (Nginx/Ingress)
        ↓
APIs / Public App Network
    ├─ Gamification.Api (x2) → 4 vCPU / 4 GB cada
        ↓
Private App Network
    ├─ RabbitMQ cluster (3 nós quorum) → 2 vCPU / 4 GB cada
    ├─ PostgreSQL (Primary + Replica) → 4 vCPU / 8 GB + SSD + Backup
    ├─ Redis (Master + 2 Sentinels) → 2 vCPU / 8 GB
    └─ Gamification.Consumer → 2 vCPU / 4 GB
```

### Segurança da Topologia
- Apenas API pública exposta pelo load balancer
- Serviços internos em rede privada isolada
- TLS obrigatório para tráfego externo
- Políticas de rede e firewalls por porta/IP
- Clusters redundantes eliminam pontos únicos de falha
- Backups regulares e replicação assíncrona

---

## Estrutura de Projetos

```
src/
├── Gamification.Api/                    # API HTTP única
│   ├── Controllers/Public/              # Endpoints públicos
│   ├── Controllers/Admin/               # Endpoints admin (requer papel Admin)
│   ├── Middlewares/
│   ├── Filters/
│   └── Program.cs
│
├── Gamification.Consumer/               # Consumidor RabbitMQ (jobs assíncronos)
│   ├── Consumers/
│   └── Program.cs
│
├── Gamification.Application/            # Casos de uso e orquestração
│   ├── Services/
│   ├── DTOs/
│   └── Interfaces/
│
├── Gamification.Domain/                 # Núcleo de domínio
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Enums/
│   ├── Events/
│   └── Interfaces/
│
├── Gamification.Infra/                  # Implementações de infraestrutura
│   ├── Data/
│   │   ├── Context/
│   │   ├── Configurations/
│   │   ├── Migrations/
│   │   └── Repositories/
│   ├── Messaging/
│   ├── Cache/
│   └── ExternalServices/
│
└── Gamification.Common/                 # Utilitários compartilhados
    ├── Exceptions/
    ├── Extensions/
    ├── Helpers/
    ├── Constants/
    └── Attributes/

tests/
├── Gamification.UnitTests/
├── Gamification.IntegrationTests/
└── Gamification.E2ETests/
```

### Responsabilidades das Camadas

**Gamification.Api**
- Porta de entrada da aplicação
- Controllers organizados em áreas (públicas/admin)
- Validação, autenticação e autorização
- Middlewares, CORS, versionamento
- Documentação OpenAPI
- Sem lógica de negócio

**Gamification.Application**
- Casos de uso e orquestração
- Transformação DTOs ↔ Entidades
- Gerenciamento de transações
- Interfaces para abstrair infraestrutura

**Gamification.Domain**
- Núcleo da aplicação
- Entidades ricas e value objects
- Regras de negócio e invariantes
- Eventos de domínio
- Independente de frameworks

**Gamification.Infra**
- Implementação de interfaces
- Repositórios (EF Core)
- Integrações externas
- Cache, mensageria, storage
- Configurações e migrations

**Gamification.Common**
- Exceções customizadas
- Validadores e helpers
- Extensões e constantes
- Evita duplicidade de código

---

## Modelagem de Domínio

### Entidades Principais

**Player**
- Identificador único (Guid sequencial)
- `ExternalUserId` (único) - ID no sistema principal
- Estatísticas: pontos, nível atual, conquistas
- Missões em andamento
- Limites diários/semanais
- Campos de auditoria: `CreatedAt`, `UpdatedAt`, `DeletedAt?`

**Mission**
- Objetivo e descrição
- Data início/fim
- Pontuação
- Condições de conclusão
- Relação com Player
- Evento `MissionCompleted`

**Achievement / Badge**
- Prêmios permanentes desbloqueados
- Critérios de conquista
- Metadados (ícone, descrição)

**Level**
- Faixas de pontos
- Benefícios por nível
- Progressão do jogador

**Reward**
- Recompensas (cupons, benefícios)
- Condições de concessão
- Data de validade

**Leaderboard**
- Rankings por período
- Agregações de pontuação
- Cache para performance

### Boas Práticas de Domínio
- Usar `DateTimeOffset` ou timestamps UTC
- Índices e constraints em campos críticos
- Soft delete com `IsActive` ou `DeletedAt`
- Coluna `RowVersion` para controle de concorrência
- Regras de negócio nas próprias entidades

---

## Design da API

### Versionamento e Convenções
- Versionamento no path: `/api/v1/`
- Padrão RESTful

### Exemplos de Endpoints

| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/api/v1/players` | Listar jogadores (paginação/filtros) |
| POST | `/api/v1/players` | Criar novo jogador |
| GET | `/api/v1/players/{id}` | Obter detalhes do jogador |
| PUT | `/api/v1/players/{id}` | Atualizar dados do jogador |
| DELETE | `/api/v1/players/{id}` | Desativar jogador (soft delete) |
| POST | `/api/v1/players/{id}/missions/{missionId}/complete` | Concluir missão |

### Requisitos da API

**Validação**
- FluentValidation ou Data Annotations
- Erros estruturados em Problem Details (RFC 7807)
- Formato: `application/problem+json`

**Autenticação/Autorização**
- JWT Bearer com validação de emissor/audiência
- Políticas: `Player`, `Admin`
- HTTPS obrigatório em todos os ambientes

**Paginação**
- Cursor-based para listas grandes (leaderboards)
- Offset para listas menores
- Cabeçalho `X-Pagination` com metadados

**Idempotência**
- Cabeçalho `Idempotency-Key` para operações críticas
- Armazenamento de chaves processadas
- Evita execução duplicada

**Tratamento de Exceções**
- Middleware de exceções global
- Conversão para Problem Details
- Logs estruturados de erros

---

## Configuração e Secrets

### Gestão de Configurações
- **Configurações não-secretas**: `appsettings.json`
- **Credenciais e secrets**: Azure Key Vault / AWS Secrets Manager
- **Nunca** armazenar secrets em:
  - Variáveis de ambiente
  - Código fonte
  - Repositórios Git

### Boas Práticas
- Rotação periódica de chaves
- Permissões mínimas de leitura
- Diferentes secrets por ambiente
- Auditoria de acesso a secrets

---

## Mensageria (RabbitMQ)

### Topologia
- **Exchange**: `gamification.events` (tipo: topic)
- **Routing Keys**: `mission.completed`, `points.assigned`, etc.
- **Queues**: específicas por consumidor

### Alta Disponibilidade
- **Quorum Queues** com 3 nós
- Replicação de mensagens
- Failover transparente
- TTL e Dead Letter Queues (DLQ)

### Boas Práticas
- Identificadores únicos em eventos (idempotência)
- Registro de mensagens processadas
- `prefetchCount` entre 10-50 (ajustar por métricas)
- Dimensionar para picos de carga
- Monitoramento de filas e latência

---

## Caching Strategy (Redis)

### Topologia Redis
- 1 Master + 2 Réplicas
- Redis Sentinel para failover automático
- Sentinels em máquinas diferentes

### Políticas de Cache

**TTLs Sugeridos**
- Leaderboards em tempo real: 30 segundos
- Consultas pesadas: alguns minutos
- Tokens de idempotência: conforme regra de negócio

**Eviction Policies**
- LRU/LFU para otimizar memória
- `maxmemory` configurado
- Evitar chaves sem expiração

**Padrão Cache-Aside**
1. Consultar cache
2. Se miss, buscar no banco
3. Armazenar no Redis
4. Retornar ao cliente
5. Invalidar cache em updates

---

## Observabilidade

### Logging (Serilog)

**Configuração**
- Logs estruturados (JSON)
- Níveis: `Information`, `Warning`, `Error`
- Enriquecimento: correlation IDs, usuário, IP
- Destinos: Console, arquivo, Seq/ElasticSearch

**Boas Práticas**
- **Nunca** salvar logs em banco SQL relacional
- MongoDB opcional para análise histórica (com dimensionamento)
- Mascarar dados sensíveis (PII, tokens)
- Filtros para logs ruidosos
- `Enrich.WithMachineName()` e `Enrich.FromLogContext()`

### Métricas e Tracing (OpenTelemetry)

**Instrumentação**
- Auto-instrumentation: ASP.NET Core, HttpClient, EF Core
- Convenções semânticas para spans
- Evitar atributos de alta cardinalidade
- Instrumentação customizada quando necessário

### Health Checks

**Endpoints**
- `/health/live` - Aplicação rodando
- `/health/ready` - Dependências disponíveis

**Verificações**
- PostgreSQL
- Redis
- RabbitMQ
- Serviços externos

**Tags**
- `ready` - Para dependências
- `live` - Para processo

---

## DevOps e Infraestrutura

### Containerização (Docker)

**Multi-stage Builds**
- Etapa de build: compilação
- Etapa final: apenas artefatos
- Imagens pequenas e seguras

**Segurança**
- Executar como usuário não privilegiado (`USER app`)
- Docker Content Trust (assinatura de imagens)
- Scanners: Trivy/Grype no pipeline
- Remover dependências desnecessárias

### Orquestração (Kubernetes)

**Deployments**
- `replicas > 1` para APIs e consumidores
- Horizontal Pod Autoscaler (CPU/RPS)
- Readiness/Liveness probes em `/health/*`

**Configurações**
- ConfigMaps para configs não-secretas
- Secrets integrados com Key Vault
- Ingress com TLS (cert-manager/ACME)
- Rate limiting no Ingress

**Endpoints Admin**
- Rotas ou subcaminhos separados
- ACLs e políticas de firewall
- Autenticação mútua
- Restrição por IP/rede

**CI/CD**
- Pipeline: Build → Tests → Security Scan → Push → Deploy
- GitHub Actions / Azure DevOps
- Helm/Kustomize para deploy
- Rotação de credenciais

---

## Segurança

### TLS e Rede
- HTTPS obrigatório (TLS >= 1.2)
- Rejeitar conexões HTTP
- Secure transfer required em Storage
- SAS tokens apenas HTTPS
- Expirações curtas

### Criptografia
- Dados em repouso: AES-256 (padrão Storage)
- Customer Managed Keys (CMK) via Key Vault
- Criptografia em trânsito (TLS)

### Segurança de Código
- Nullable habilitado em C#
- Validação e sanitização de inputs
- Proteção contra SQL injection
- AntiForgery tokens
- Dependências atualizadas (Dependabot)
- Code scanning (SonarQube/Semgrep)

### Segurança Operacional
- IAM centralizado em clusters
- Logs de auditoria ativos
- Revisão periódica de permissões
- Alertas para eventos suspeitos

---

## Armazenamento de Arquivos (Object Storage)

### Serviços Suportados
- Cloudflare R2
- Azure Blob Storage

### Configurações de Segurança

**Transfer Security**
- Secure transfer required habilitado
- Apenas HTTPS permitido
- Rejeitar conexões HTTP

**Controle de Acesso**
- SAS tokens com escopos limitados
- Expirações curtas
- RBAC com usuários/grupos
- Geração controlada de tokens

**Proteção de Dados**
- Criptografia em repouso (AES-256)
- Soft delete habilitado
- Retenção imutável para dados críticos
- Versionamento de objetos

**Rede**
- Private Endpoints (Private Link)
- Firewall configurado
- Acesso via rede privada

### Organização
- Diretórios por entidade: `/players/{id}/uploads/`
- Nomes únicos: UUID + timestamp
- Evitar sobrescrever arquivos
- Versionamento de objetos

---

## Roadmap de Implementação

### 1. Estrutura Inicial
- [ ] Criar solution `.sln`
- [ ] Criar projetos: Api, Application, Domain, Infra, Common, Consumer
- [ ] Configurar dependências entre projetos

### 2. Entity Framework
- [ ] Modelar entidades de domínio
- [ ] Criar DbContext e configurações
- [ ] Gerar migrations iniciais
- [ ] Configurar conexões por ambiente
- [ ] Implementar replicação

### 3. Casos de Uso
- [ ] Criar serviços no Application
- [ ] Implementar regras de negócio
- [ ] Mapear DTOs (AutoMapper/Mapster)
- [ ] Definir interfaces

### 4. Controllers
- [ ] Criar controllers públicos
- [ ] Criar controllers admin
- [ ] Implementar versionamento
- [ ] Adicionar validação
- [ ] Configurar autenticação/autorização

### 5. Mensageria
- [ ] Definir exchanges e queues RabbitMQ
- [ ] Implementar publishers na API
- [ ] Criar consumers no Gamification.Consumer
- [ ] Configurar quorum queues
- [ ] Implementar idempotência

### 6. Caching
- [ ] Configurar cluster Redis
- [ ] Implementar cache-aside
- [ ] Criar serviços de idempotência
- [ ] Otimizar leaderboards
- [ ] Ajustar TTLs

### 7. Observabilidade
- [ ] Configurar Serilog em todos os projetos
- [ ] Integrar OpenTelemetry
- [ ] Configurar Prometheus/Grafana
- [ ] Criar dashboards
- [ ] Definir alertas

### 8. DevOps
- [ ] Criar Dockerfiles multi-stage
- [ ] Configurar docker-compose
- [ ] Preparar manifests Kubernetes
- [ ] Setup CI/CD pipeline
- [ ] Configurar Key Vault
- [ ] Scans de segurança

### 9. Testes
- [ ] Implementar testes unitários
- [ ] Configurar Testcontainers
- [ ] Criar testes de integração
- [ ] Desenvolver testes E2E
- [ ] Automatizar pipeline de testes

### 10. Revisão de Segurança
- [ ] Validar CORS
- [ ] Configurar headers de segurança
- [ ] Implementar rate limiting
- [ ] Revisar sanitização
- [ ] Validar RBAC
- [ ] Rotacionar chaves

---

## Considerações Finais

Esta especificação fornece uma base robusta para construir uma plataforma de gamificação escalável e segura. A arquitetura em camadas facilita manutenção e testes, enquanto as práticas de observabilidade e segurança garantem um ambiente confiável para produção.

**Princípios Fundamentais**
- Separação clara de responsabilidades
- Segurança em todos os níveis
- Observabilidade completa
- Alta disponibilidade
- Testes automatizados
- DevOps desde o início
