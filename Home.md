# Permify AuthZ Documentation

Bem-vindo à documentação oficial do Permify para a plataforma ClickBus. Este repositório contém guias completos para implementação de autorização fine-grained usando Permify + Keycloak.

## 📋 Conteúdo

### 🏠 Visão Geral
- [Visão Geral](Home) - Esta página
- [PRD — Permify AuthZ](PRD---Permify-AuthZ) - Product Requirements Document
- [RFC — Decisões de Design](RFC---Decisões-de-Design) - Arquitetura e decisões técnicas

### 🏗️ Arquitetura
- [Arquitetura AuthN + AuthZ](Arquitetura-AuthN-+-AuthZ) - Separação entre autenticação e autorização
- [Hierarquia Org → Company → Module](Hierarquia-Org-→-Company-→-Module) - Modelo de hierarquia multi-tenant
- [Fluxo a Nível de Recurso](Fluxo-a-Nível-de-Recurso) - Controle granular de recursos
- [Política de Permissões](Política-de-Permissões) - Regras e padrões de segurança

### 🛠️ Desenvolvimento
- [Integração NestJS + Spring Boot](Integração-NestJS-+-Spring-Boot) - Guias práticos de integração
- [Integração no BFF (Guards)](Integração-no-BFF-(Guards)) - Implementação de guards no backend
- [Referência da API REST](Referência-da-API-REST) - Endpoints e exemplos
- [Exemplos Práticos](Exemplos-Práticos) - Casos de uso e implementações
- [Onboarding de Times](Onboarding-de-Times) - Guia para novos times

### ⚙️ Operação
- [Configuração do Permify](Configuração-do-Permify) - Setup e configuração
- [Análise de Custo](Análise-de-Custo) - Estimativas e comparações

### 📚 Referência
- [Permify vs Keycloak](Permify-vs-Keycloak) - Comparação entre soluções

## 🚀 Começando

1. Leia a [Visão Geral](Arquitetura-AuthN-+-AuthZ) para entender a arquitetura
2. Consulte o [PRD](PRD---Permify-AuthZ) para requisitos e decisões de produto
3. Siga o guia de [Integração NestJS + Spring Boot](Integração-NestJS-+-Spring-Boot) para implementação
4. Use os [Exemplos Práticos](Exemplos-Práticos) como referência

## 📥 SDKs Oficiais

| Linguagem | SDK | Instalação |
|:---|:---|:---|
| Node.js/TypeScript | [permify-node](https://github.com/Permify/permify-node) | `npm install @permify/permify-node` |
| Java | [permify-java](https://github.com/Permify/permify-java) | Maven/Gradle |
| Go | [permify-go](https://github.com/Permify/permify-go) | `go get github.com/Permify/permify-go` |
| Python | [permify-python](https://github.com/Permify/permify-python) | `pip install permify` |
| Ruby | [permify-ruby](https://github.com/Permify/permify-ruby) | `gem install permify` |
| PHP | [permify-php](https://github.com/Permify/permify-php) | Composer |
| C# | [permify-csharp](https://github.com/Permify/permify-csharp) | NuGet |

## 🔗 Links Úteis

- [Permify Official](https://permify.co/) - Site oficial
- [Permify GitHub](https://github.com/Permify) - Repositórios oficiais
- [Keycloak Documentation](https://www.keycloak.org/documentation) - Documentação do Keycloak

---

**Platform Team · 2026**