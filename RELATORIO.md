# RELATORIO.md — Análise Técnica do Projeto Spotlight

> **Projeto:** Spotlight — Sistema de Descoberta de Filmes com Inteligência Artificial  
> **Contexto:** TCC · COTIL/UNICAMP · 2026  
> **Revisão:** Arquitetura de Software — Perspectiva Técnica  
> **Data:** Junho de 2026

---

## Sumário

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [Robustez](#2-robustez)
3. [Escalabilidade](#3-escalabilidade)
4. [Manutenibilidade](#4-manutenibilidade)
5. [Adesão às Boas Práticas — Clean Code e SOLID](#5-adesão-às-boas-práticas--clean-code-e-solid)
6. [Conformidade com a Proposta do Projeto](#6-conformidade-com-a-proposta-do-projeto)
7. [Recomendações Prioritárias](#7-recomendações-prioritárias)

---

## 1. Visão Geral da Arquitetura

O projeto adota uma arquitetura em camadas bem definida para Flutter:

```
lib/
├── models/       → entidades de domínio (imutáveis, com fromJson/copyWith)
├── services/     → adaptadores de API (TMDb, Gemini, Groq, Supabase)
├── providers/    → gerenciamento de estado (Provider / ChangeNotifier)
├── features/     → telas organizadas por funcionalidade (hub, chat, details, etc.)
├── widgets/      → componentes reutilizáveis de UI
└── app/          → raiz do app, constantes globais, HomeScreen
```

O padrão principal é **Provider** (ChangeNotifier), com separação entre camadas de dados, domínio e apresentação. A entrada da aplicação em `main.dart` é responsável pela composição do grafo de dependências via `MultiProvider`. O fluxo de dados é unidirecional: serviços alimentam providers, providers alimentam a UI.

---

## 2. Robustez

### Pontos Fortes

| Mecanismo | Localização | Descrição |
|---|---|---|
| Hierarquia de exceções tipada | `lib/services/app_exceptions.dart` | Uso de `sealed class SpotlightException` com subclasses `NetworkException`, `ParseException`, `SpotlightAuthException`. Permite tratamento diferenciado por tipo. |
| Retry com backoff | `TMDBService._withRetry()` | Lógica de retry de até 2 tentativas com delay de 1s, diferenciando erros 4xx (sem retry) de 5xx e timeouts. |
| Proteção de race condition | `ReviewsProvider`, todos `_load*` | Guard `if (_selectedMovie?.id != movie.id) return;` impede que respostas de requisições desatualizadas sobrescrevam estado atual ao navegar rapidamente. |
| Debounce de busca | `SearchProvider` | Timer de 500ms cancela requisições anteriores ao digitar, evitando flood de chamadas à API. |
| Filtragem de conteúdo futuro | `TMDBService._filterFutureContent()` | Remove itens com data de lançamento posterior a 31/12/2026, evitando poluição da UI com conteúdo fictício. |
| Fallback de IA | `ChatProvider.sendMessage()` | Fluxo Gemini → Groq → mensagem de erro garante disponibilidade do chat mesmo com falha do provedor primário. |
| Imutabilidade nos getters | Todos os providers | `List.unmodifiable()` nos getters públicos impede mutação acidental do estado interno. |
| Agendamento pós-frame | `MoviesProvider` | `SchedulerBinding.instance.addPostFrameCallback` evita chamar `notifyListeners` durante o ciclo de build. |

### Pontos de Atenção

- **Silent catches generalizados:** Vários métodos em `ReviewsProvider` e `TMDBService` usam `catch (_) {}` sem logging. Isso elimina ruído de UI corretamente (degradação graciosa), mas dificulta diagnóstico de falhas em produção. A ausência de um sistema de logging estruturado (ex: `package:logger`) é a principal lacuna de observabilidade.

- **Detecção de erro por conteúdo de string:** Em `ChatProvider.sendMessage()`, o código verifica erros do Gemini com `reply.contains('cota')` ou `reply.contains('Erro ao conectar')`. Esta abordagem é frágil: qualquer mudança na mensagem de erro do SDK quebra silenciosamente o fallback.

- **Favoritos com dados incompletos:** `FavoritesProvider._rowToMovie()` reconstrói objetos `Movie` com `rating: 0.0`, `synopsis: ''` e `genre: ''`. Ao abrir a tela de lista favorita, o usuário verá dados parciais até que `openDetails` seja chamado e recarregue os dados do TMDb.

- **Ausência de testes automatizados:** O único arquivo em `test/` é o template padrão do Flutter (smoke test de contador), completamente desconexo da aplicação real. Não há nenhum teste unitário, de integração ou de widget implementado para a lógica de negócio.

---

## 3. Escalabilidade

### Pontos Fortes

| Aspecto | Localização | Descrição |
|---|---|---|
| Paralelismo de requisições | `MoviesProvider.loadData()` | `Future.wait` dispara 15 requisições simultaneamente à API do TMDb, minimizando o tempo de carregamento da tela Hub. |
| Backend serverless | Supabase | A escolha de Supabase elimina gestão de infraestrutura; a camada PostgreSQL e autenticação escalam automaticamente. |
| Cache de imagens | `cached_network_image` + `flutter_cache_manager` | Evita re-download de assets de imagem, reduzindo consumo de banda e latência. |
| Cache de mídia no Supabase | `midias_cache` (table) + `ReviewService._upsertMidiaCache()` | Evita consultas repetidas ao TMDb para o mesmo conteúdo ao vincular avaliações. |

### Limitações Estruturais

- **Ausência de paginação:** Todas as listas (`fetchTrending`, `fetchGenre`, etc.) retornam apenas a primeira página da API (20 itens por padrão). Não existe scroll infinito ou carregamento progressivo. Para uma versão com catálogo expandido, isso se tornaria um gargalo de UX.

- **MoviesProvider com 15+ campos codificados:** O provider carrega 15 seções fixas na inicialização. Adicionar uma nova seção implica: novo campo privado, novo getter, nova chamada em `Future.wait`, nova atribuição por índice (`results[N]`). Este padrão é o que mais compromete a escalabilidade do código no longo prazo.

- **Histórico de chat limitado a 100 linhas:** O limite arbitrário em `ChatProvider.loadHistory()` (`.limit(100)`) não é configurável e não expõe paginação. Um usuário ativo perderia contexto histórico silenciosamente.

- **Sem cache de dados entre sessões:** Todo estado é perdido ao reiniciar o app. Providers como `MoviesProvider` e `SearchProvider` não persistem localmente (ex: via `shared_preferences` ou `hive`). Cada abertura do app consome as mesmas 15 requisições à API do TMDb.

- **Serviços estáticos não testáveis:** A natureza estática de `TMDBService`, `GeminiService` e `GroqService` impede qualquer forma de mock ou substituição para testes, staging ou ambientes alternativos.

---

## 4. Manutenibilidade

### Pontos Fortes

- **Estrutura de pastas orientada a feature:** `lib/features/hub`, `lib/features/chat`, `lib/features/details` — novos desenvolvedores encontram rapidamente onde está cada funcionalidade.

- **`DetailsController` como parameter object:** O uso de um controller imutável que agrega dados e callbacks para a `DetailsView` desacopla a view do estado dos providers, tornando-a testável e reutilizável sem dependências implícitas.

- **Hierarquia de exceções documentada:** `app_exceptions.dart` é bem comentado, com a distinção de nomenclatura `SpotlightAuthException` vs `AuthException` do Supabase devidamente explicada.

- **Providers com interface limpa:** Getters imutáveis, métodos públicos com nomes claros (`loadData`, `openDetails`, `toggleFavorite`, `clearError`), sem exposição desnecessária de estado interno.

- **Análise estática ativada:** `flutter_lints` configurado via `analysis_options.yaml`.

### Pontos de Atenção

- **`TMDBService` monolítico:** Com mais de 350 linhas e responsabilidades que abrangem filmes, séries, atores, episódios, trailers, imagens, avaliações de conteúdo e provedores de streaming, esta classe viola o SRP e será o principal ponto de fricção para manutenção futura.

- **`ReviewsProvider` com dupla responsabilidade:** O provider gerencia tanto o estado do overlay de detalhes (filme selecionado, elenco, conteúdo relacionado, episódios) quanto as avaliações/comentários. São duas preocupações distintas que poderiam ser separadas em `DetailsProvider` e `ReviewsProvider`.

- **Índices mágicos em `MoviesProvider`:** A atribuição `_topSeries = results[0]`, `_topMovies = results[1]`, ..., `_appleNew = results[11]` cria um acoplamento frágil entre a ordem dos `Future.wait` e as variáveis de destino. Um reordenamento acidental de qualquer entrada gera um bug difícil de rastrear.

- **Teste inexistente:** A ausência de testes torna qualquer refatoração ou adição de funcionalidade um processo arriscado. O template de teste padrão sugere que testes nunca foram uma prioridade no desenvolvimento.

- **`Movie.toMinimalJson()` com nota de issue não resolvida:** O método contém um comentário multi-linha reconhecendo que `genre_ids` não é serializado corretamente, o que causa perda de dado de gênero no histórico de chat. A issue está documentada no código mas não foi corrigida.

- **Mistura de idiomas em identificadores de domínio:** Nomes de tabelas e campos do Supabase (`avaliacoes`, `nota`, `comentario`, `usuario_id`, `midia_id`) aparecem como strings literais espalhadas por `ReviewService` e `ChatProvider`. Uma camada de constantes ou DTO dedicado evitaria typos silenciosos.

---

## 5. Adesão às Boas Práticas — Clean Code e SOLID

### Single Responsibility Principle (SRP)

| Classe | Avaliação |
|---|---|
| `AuthService` | ✅ Responsabilidade única: wrapping de auth do Supabase |
| `ReviewService` | ✅ CRUD de avaliações + cache de mídia (aceitável) |
| `SearchProvider` | ✅ Exclusivamente estado de busca com debounce |
| `ThemeProvider` | ✅ Apenas toggle de tema |
| `TMDBService` | ❌ Responsabilidades de filmes, séries, atores, episódios, trailers, imagens, avaliações, provedores e busca numa única classe |
| `ReviewsProvider` | ❌ Overlay de detalhes (seleção, elenco, relacionados, episódios) + avaliações/comentários |
| `HomeScreen` | ⚠️ Gerencia navegação entre abas + diálogos de edição/exclusão de comentários |

### Open/Closed Principle (OCP)

- **Parcialmente atendido.** Adicionar um novo provedor de streaming exige apenas inserir uma entrada em `_providerSlugMap` — fechado para modificação. Porém, adicionar um novo provedor de IA (além de Gemini e Groq) exige modificar diretamente `ChatProvider.sendMessage()`, violando o princípio.

### Liskov Substitution Principle (LSP)

- **Atendido** na hierarquia de exceções: qualquer `SpotlightException` pode ser capturada genericamente sem quebrar semântica.
- **Não aplicável** aos serviços estáticos.

### Interface Segregation Principle (ISP)

- Dart não usa interfaces explícitas frequentemente, mas os providers expõem APIs coesas e minimalistas — sem métodos desnecessários para os consumidores.

### Dependency Inversion Principle (DIP)

| Ponto | Avaliação |
|---|---|
| `ReviewsProvider(ReviewService, AuthProvider)` | ✅ Injeção de dependência via construtor |
| `TMDBService` (estático) | ❌ Sem abstração; impossível substituir ou mockar |
| `GeminiService` / `GroqService` (estáticos) | ❌ Sem abstração; fallback acoplado diretamente no provider |
| `ChatProvider` / `FavoritesProvider` com `Supabase.instance.client` | ⚠️ Dependência de singleton global — dificulta testes |

### Clean Code — Observações Adicionais

**Boas práticas observadas:**
- Nomes de métodos e variáveis expressivos (`openDetails`, `toggleFavorite`, `_filterFutureContent`, `_normalizeBrRating`)
- Uso consistente de `final` para imutabilidade
- Evitação de `null` desnecessário com `??` e `?.`
- `List.unmodifiable()` em getters públicos
- `copyWith` no modelo `Movie`
- `operator ==` e `hashCode` no modelo `Movie` baseados em ID

**Práticas que podem melhorar:**
- `catch (_) {}` sem logging é um anti-padrão de depuração
- `results[0]...[14]` são números mágicos sem contexto
- Detecção de erro via `String.contains()` é frágil (`reply.contains('cota')`)
- Comentário de issue aberta dentro de `toMinimalJson()` deveria ser um bug tracker, não código

---

## 6. Conformidade com a Proposta do Projeto

A proposta — "Sistema de Descoberta de Filmes com Inteligência Artificial" — está amplamente atendida. A tabela abaixo mapeia cada entregável esperado ao que foi implementado:

| Requisito | Status | Implementação |
|---|---|---|
| Descoberta de filmes e séries | ✅ Completo | Hub com 15 seções (gêneros, provedores, tendências, Oscar) |
| Busca por título | ✅ Completo | `SearchView` + `SearchProvider` com debounce de 500ms |
| Detalhes de filme/série | ✅ Completo | `DetailsView` com elenco, episódios, relacionados, avaliação de conteúdo, trailer, "onde assistir" |
| Assistente de IA | ✅ Completo | Pipeline Gemini + TMDb com fallback Groq; histórico persistido no Supabase |
| Autenticação de usuário | ✅ Completo | Email/senha + Google OAuth, com deep linking mobile e fluxo localhost para desktop |
| Favoritos | ✅ Completo | Persistência no Supabase com `upsert`; otimista (local-first) |
| Avaliações e comentários | ✅ Completo | CRUD completo com JOIN de perfil de autor; RLS implícita via `usuario_id` |
| Perfil de usuário | ✅ Completo | Exibição de dados, toggle de tema, logout |
| Tema dark/light | ✅ Completo | `ThemeProvider` com `ThemeData.useMaterial3` |
| Responsividade (mobile/desktop) | ✅ Completo | Breakpoint em 1024px com sidebar desktop e bottom nav mobile |
| Conteúdo localizado (pt-BR) | ✅ Completo | Todos os endpoints com `language=pt-BR&region=BR`; títulos brasileiros via translations da API |
| Detalhes de ator | ✅ Completo | `ActorDetailView` com filmografia do ator |
| Detalhes de episódio (séries) | ✅ Completo | `EpisodeDetailView` com seletor de temporada |

**Lacuna identificada:** A tela de perfil não exibe o histórico de avaliações do usuário (suas próprias reviews). Um usuário não consegue rever o que já avaliou sem navegar para o item manualmente — funcionalidade natural para um sistema de descoberta.

---

## 7. Recomendações Prioritárias

As recomendações abaixo estão ordenadas por impacto no contexto de um TCC:

### P1 — Alta Prioridade

1. **Adicionar testes unitários para os Providers:** Começar por `SearchProvider` e `MoviesProvider`, que têm lógica de estado testável sem dependências externas. Um teste simples valida debounce, estado de loading e tratamento de erro.

2. **Eliminar os índices mágicos em `MoviesProvider`:** Trocar `results[0]...results[14]` por desestruturação nomeada ou um enum, eliminando o risco de regressão silenciosa.

3. **Substituir detecção de erro por string por tipo:**
   ```dart
   // Atual (frágil)
   if (reply.contains('cota') || reply.contains('Erro'))
   
   // Recomendado
   try {
     reply = await GeminiService.generateResponse(prompt);
   } on GeminiQuotaException {
     reply = await GroqService.generateResponse(prompt);
   }
   ```

### P2 — Média Prioridade

4. **Adicionar logging estruturado nos silent catches:** Mesmo `debugPrint` nos blocos `catch (_) {}` críticos (ex: persistência de chat, sincronização de favoritos) melhora drasticamente a depuração.

5. **Separar `ReviewsProvider` em dois:** `DetailsStateProvider` (filme selecionado, elenco, relacionados) e `ReviewsProvider` (avaliações e comentários). Reduz o tamanho da classe e torna cada um mais fácil de testar.

6. **Corrigir `Movie.toMinimalJson()`:** O campo `genre_ids` deveria ser serializado com o ID de gênero reverso a partir do nome, ou `genre` deveria ser incluído explicitamente, para evitar perda de dado de gênero no histórico de chat.

### P3 — Baixa Prioridade (melhorias futuras)

7. **Abstrair `TMDBService` em serviços menores:** `MovieService`, `TvService`, `ActorService`, `SearchService` — cada um com no máximo 80–100 linhas.

8. **Introduzir abstração para os serviços de IA:** Uma interface `AiChatService` permitiria adicionar ou trocar provedores sem modificar `ChatProvider`.

9. **Histórico de avaliações no perfil do usuário:** Completaria o ciclo de descoberta → avaliação → revisão.

10. **Paginação na tela Hub e nos resultados de busca:** Essencial para uma versão pública do produto.

---

_Relatório gerado para fins acadêmicos — TCC COTIL/UNICAMP 2026._
