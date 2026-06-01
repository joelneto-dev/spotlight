# Spotlight

> **Porque você já passa tempo demais escolhendo o que assistir.**

Spotlight é um aplicativo Flutter de descoberta de filmes e séries movido por Inteligência Artificial. Desenvolvido como Trabalho de Conclusão de Curso (TCC) para o curso de Desenvolvimento de Sistemas do **COTIL/UNICAMP**, turma de 2026.

---

## Funcionalidades

- **Hub de Descoberta** — seções curadas por gênero (Ação, Terror, Ficção Científica, Animação, etc.), plataforma (Netflix, Prime, HBO Max, Disney+, Apple TV+) e tendências da semana
- **Spot AI** — assistente de chat com IA que recomenda filmes e séries usando Google Gemini (com fallback para Groq/Llama 3), integrado ao catálogo real do TMDb
- **Detalhes Completos** — elenco, trailers, avaliação de conteúdo brasileira (L/A10/A12/A14/A16/A18), onde assistir, relacionados por diretor/ator, e lista de episódios para séries
- **Avaliações e Comentários** — sistema de rating (1–10) com comentários, persistidos no Supabase com perfil do autor
- **Favoritos** — lista pessoal sincronizada na nuvem
- **Busca** — busca em tempo real com debounce, cobrindo filmes e séries
- **Autenticação** — login com e-mail/senha ou Google (OAuth)
- **Tema Dark/Light** — alternável pelo usuário
- **Responsivo** — layout adaptado para mobile (bottom nav) e desktop (sidebar, breakpoint ≥ 1024 px)

---

## Stack

| Camada | Tecnologia |
|---|---|
| Framework | Flutter (Dart 3.11+) |
| Estado | Provider (`ChangeNotifier`) |
| Backend & Auth | Supabase (PostgreSQL + Auth) |
| Catálogo de Filmes | The Movie Database (TMDb) API v3 |
| IA Principal | Google Gemini 2.5 Flash |
| IA de Fallback | Groq (Llama 3.3 70B) |
| Imagens | `cached_network_image` |

---

## Pré-requisitos

- [Flutter SDK](https://docs.flutter.dev/get-started/install) ≥ 3.11.5
- Conta no [Supabase](https://supabase.com/) com projeto configurado
- Token de acesso à API do [TMDb](https://developer.themoviedb.org/)
- Chave de API do [Google AI Studio (Gemini)](https://aistudio.google.com/app/apikey)
- (Opcional) Chave da [API do Groq](https://console.groq.com/keys) para fallback de IA

---

## Configuração

### 1. Clone o repositório

```bash
git clone <url-do-repositorio>
cd spotlight
```

### 2. Instale as dependências

```bash
flutter pub get
```

### 3. Configure os secrets

Crie o arquivo `lib/services/app_secrets.dart` (este arquivo é ignorado pelo `.gitignore`):

```dart
abstract class AppSecrets {
  static const String supabaseUrl      = 'https://seu-projeto.supabase.co';
  static const String supabaseAnonKey  = 'sua-anon-key';
  static const String tmdbToken        = 'seu-bearer-token-tmdb';
  static const String geminiApiKey     = 'sua-chave-gemini';
  static const String groqApiKey       = 'sua-chave-groq'; // opcional
}
```

Consulte `.env.example` na raiz do projeto para referência de onde obter cada chave.

### 4. Configure o banco de dados

No Supabase, crie as seguintes tabelas:

**`profiles`** — criada automaticamente via trigger no cadastro  
**`favoritos`** — `user_id`, `tmdb_id`, `media_type`, `title`, `poster_path`, `added_at`  
**`midias_cache`** — `tmdb_id` (unique), `titulo`, `tipo`, `sinopse`, `poster_path`, `backdrop_path`, `data_lancamento`, `ultima_sincronizacao`  
**`avaliacoes`** — `usuario_id`, `midia_id` (FK → midias_cache), `nota`, `comentario`, `criado_em`, `atualizado_em`  
**`chat_history`** — `user_id`, `role`, `content`, `metadata` (jsonb), `created_at`

### 5. Configure o Deep Link para Google OAuth (mobile)

- **Android:** registre o scheme `br.com.spotlight://login-callback` em `android/app/src/main/AndroidManifest.xml`
- **iOS:** adicione o mesmo scheme em `CFBundleURLSchemes` no `ios/Runner/Info.plist`

---

## Executando o projeto

```bash
# Mobile (conectar dispositivo ou emulador)
flutter run

# Web
flutter run -d chrome

# Verificar lint e análise estática
flutter analyze

# Rodar testes
flutter test
```

---

## Estrutura do Projeto

```
lib/
├── app/               # SpotlightApp, HomeScreen, constantes globais (AppTab, breakpoints)
├── features/
│   ├── hub/           # Tela principal com carrosséis de descoberta
│   ├── search/        # Busca em tempo real
│   ├── details/       # Overlay de detalhes (DetailsView + DetailsController)
│   ├── actor/         # Página de detalhes de ator
│   ├── chat/          # Chat com Spot AI
│   ├── list/          # Lista de favoritos
│   └── profile/       # Perfil e configurações
├── models/            # Movie, Review, CastMember, Episode, ChatMessage
├── providers/         # AuthProvider, MoviesProvider, SearchProvider,
│                      # FavoritesProvider, ReviewsProvider, ChatProvider, ThemeProvider
├── services/          # TMDBService, GeminiService, GroqService,
│                      # AuthService, ReviewService, AppConfig, AppSecrets
└── widgets/           # Componentes reutilizáveis (ui_components, navigation, auth_widgets)
```

---

## Fluxo do Chat com IA

O assistente **Spot AI** usa um pipeline de dois estágios:

1. **Extração de intenção:** Gemini analisa a mensagem e extrai termos de busca (ex: *"Quero algo como Matrix"* → `"Matrix"`)
2. **Enriquecimento do prompt:** Os termos são buscados no TMDb; os resultados reais do catálogo são injetados no prompt antes de chamar a IA novamente
3. **Fallback automático:** Se o Gemini falhar por cota ou erro, a mesma lógica é executada com Groq (Llama 3.3 70B)

Isso garante que as recomendações da IA sejam sempre títulos que existem no catálogo do aplicativo.

---

## Documentação Adicional

- [`docs/API.md`](docs/API.md) — detalhamento das integrações de API e Supabase
- [`RELATORIO.md`](RELATORIO.md) — análise técnica completa (arquitetura, robustez, SOLID, etc.)

---

## Licença

Distribuído sob a licença MIT. Consulte [`LICENSE`](LICENSE) para mais detalhes.

---

*TCC · Desenvolvimento de Sistemas · COTIL/UNICAMP · 2026*
