# Performance, bateria, rede e push — o device é limitado e o usuário sente

> No servidor a métrica é throughput; no **mobile é a experiência num aparelho fraco, com bateria
> finita, rede cara e intermitente, e um usuário que sente cada ms de travamento**. Performance
> aqui é requisito de produto (e de loja: ANR/crash derrubam ranking), não polimento opcional.
> Otimize com **perfil/medição** (mesma disciplina do `schematize-web`/CWV), nunca por achismo.

## 1. Startup e responsividade da UI

- **Cold start é a primeira impressão:** minimize trabalho no lançamento — nada de I/O de rede/disco
  pesado no `application:didFinishLaunching`/`Application.onCreate`. Inicialização preguiçosa,
  splash que dá lugar a conteúdo rápido, meça o **time-to-interactive**.
- **Nunca trabalho pesado na main/UI thread:** parsing, cripto, I/O, imagem — fora da thread de UI
  (coroutines/async, GCD, isolates). Frame budget é ~16ms (60fps) / ~8ms (120fps); estourar =
  jank. **ANR (Android)** por bloquear a main thread é rejeição/desinstalação.
- **Listas eficientes:** reciclagem (RecyclerView/LazyColumn/List/FlatList), paginação, sem
  re-render/re-layout desnecessário. Imagem dimensionada e cacheada, não bitmap gigante em
  memória.
- **Memória:** perfis de device baixo; evite leaks (listeners/observers não removidos, contexto
  retido); teste sob pressão de memória — o SO **mata** o app que abusa.

## 2. Bateria e recursos — o usuário culpa o app que drena

- **Wakelocks e background com parcimônia:** trabalho em background é caro e **restrito pelo SO**
  (Doze/App Standby no Android, limites de background no iOS). Use as APIs certas —
  **WorkManager (Android) / BGTaskScheduler (iOS)** — que agendam respeitando bateria, não loops
  próprios.
- **Coalescer e adiar:** agrupe sync/upload em janelas (device carregando + no Wi-Fi para trabalho
  pesado), não a cada evento. GPS contínuo, sensores e polling são vilões de bateria — use o menor
  nível de precisão/frequência que resolve.
- **Sem polling onde há push:** substitua "pergunta ao servidor a cada X" por push/sync sob evento
  (§4). Polling em foreground constante queima bateria e dados.

## 3. Rede — cara, lenta e intermitente

- **Assuma rede ruim por padrão** (casa com `offline-sync.md`): timeouts curtos com retry+backoff,
  operações canceláveis, degradação graciosa. Nunca UI travada esperando request.
- **Minimize bytes:** compressão (gzip/br), payload enxuto, **delta sync** em vez de full-refresh,
  paginação. Respeite **rede medida/roaming** — não baixe vídeo/asset pesado em dados móveis sem
  consentimento.
- **Cache HTTP + camada local:** ETag/Cache-Control, imagens em cache de disco com política de
  evicção. A UI lê do local; a rede reidrata (`offline-sync.md`).
- **Coalescing de requests:** deduplique chamadas idênticas em voo; batelada onde a API permite.
- **Adaptação à qualidade da rede:** em rede fraca, baixe thumbnails/qualidade menor; prefetch só
  em Wi-Fi. Detecte tipo de conexão e adapte.

## 4. Push notifications — engajamento sem abuso, e com segurança

- **Canais oficiais:** **APNs (iOS) / FCM (Android)**. O token de push é registrado no **backend da
  casa** e **atado à sessão/usuário** — rotacionado no logout/troca de usuário (`iam-mobile.md` §5),
  senão o push de um vai pro device de outro. O **registro carrega `env`/`build_id`**: push é
  **efeito externo**, e fora de `prd` vai pra **sandbox/projeto FCM de teste** — o backend de prd
  **recusa** token de build de não-prd (`entrega-lojas.md` §7).
- **Payload não é confiável nem secreto:** trate a notificação como **gatilho**, não como fonte de
  verdade — o app busca o dado real autenticado ao abrir. **Nunca** mande PII/segredo/token no
  payload de push (passa por serviço de terceiro e aparece na tela de bloqueio).
- **Permissão just-in-time e com valor:** peça permissão de notificação **no momento** em que o
  valor está claro (não na 1ª abertura, seca). Respeite a recusa; ofereça granularidade
  (categorias/canais Android).
- **Sem spam:** frequência e relevância são gate de retenção e de loja. Rate-limit no servidor,
  respeita quiet hours/preferências, opt-out fácil. Push de marketing sem consentimento = desinstala.
- **Silent/background push com limite:** úteis pra sync oportunista, mas o SO **limita** (não conte
  com entrega garantida nem alta frequência). Data push é best-effort; o sync de verdade é a
  reconciliação (`offline-sync.md`), não "confio que o push chegou".

## 5. Assets e tamanho do app

- **App menor instala e converte mais:** app thinning / **App Bundle (Android)** e **on-demand
  resources / dynamic feature modules** pra não empacotar tudo. Imagens em formato eficiente
  (WebP/HEIC/vetor), sem assets órfãos.
- **Orçamento de tamanho** monitorado por release (regressão de tamanho é dívida rastreável).

## 6. Medir, não adivinhar (entra na DoD)

- **Perfile antes de otimizar:** Instruments (iOS), Android Studio Profiler/Perfetto, Flipper —
  ache o gargalo real (CPU, alocação, jank, energia) em vez de "otimizar" o que não dói.
- **Métricas de gate:** cold start, jank/frozen frames, crash-free, consumo de bateria/rede em
  cenário padrão — acompanhadas por release (casa com `entrega-lojas.md` §6 e `observabilidade.md`).
- **Teste em device fraco e rede ruim:** o device topo de linha do dev mente. Matriz de teste inclui
  aparelho de baixo custo, SO antigo suportado, e rede throttled/offline.
