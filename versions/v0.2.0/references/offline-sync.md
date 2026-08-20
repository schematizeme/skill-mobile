# Offline-first & sincronização — a rede é intermitente por padrão

> No desktop a rede é quase sempre presumida; no **mobile ela falha o tempo todo** — metrô,
> elevador, roaming, avião, sinal fraco. App da casa é **offline-first por desenho**: usável sem
> rede, e a sincronização é um **problema de dados distribuídos** (conflito, fila, reconciliação),
> não um "salva quando dá". Este é o piso; o servidor continua sendo a **fonte de verdade**.

Offline-first não é "cachear o último request". É tratar o dispositivo como uma **réplica local**
que aceita leitura e escrita sem rede, e reconcilia com o servidor quando a rede volta — sem
perder escrita do usuário e sem corromper o estado.

## 1. Princípios

- **A UI lê do local, sempre.** A tela renderiza do banco/local store (SQLite, Room, Core Data,
  Realm, SQLDelight), nunca direto do request. A rede **alimenta** o local; a UI **observa** o
  local. Rede caindo no meio não trava a tela.
- **Escrita é otimista + fila durável.** Ação do usuário grava local na hora (UI responde já) e
  enfileira uma **mutação durável** pra enviar. A fila sobrevive a fechar o app, matar o processo,
  reiniciar o device (persistida em disco, não em memória).
- **O servidor é a fonte de verdade.** O local é réplica; em conflito, a política de resolução
  decide, mas a **autoridade** (e a authz — `iam-mobile.md`) é do servidor. O cliente nunca é o
  juiz de saldo, estoque, preço ou permissão.
- **Idempotência ponta a ponta.** Toda mutação carrega uma **chave de idempotência** (ULID gerado
  no device) — reenvio após timeout não duplica. O servidor deduplica por essa chave.

## 2. A fila de mutações (outbox)

O padrão é uma **outbox** local: cada escrita do usuário vira um registro persistido
`{op_id (ULID), tipo, payload, base_version, criado_em, tentativas, estado}`.

- **Envio ordenado e em lote** quando a rede volta; respeita dependências (não envia "editar item"
  antes de "criar item").
- **Retry com backoff exponencial + jitter**; teto de tentativas → move pra **dead-letter** local
  e sinaliza ao usuário (não fica reenviando pra sempre queimando bateria/dados).
- **Estados explícitos por item:** `pendente → enviando → confirmado | conflito | falho`. A UI
  pode mostrar "sincronizando…"/"salvo"/"falhou, toque pra tentar".
- **Nada de segredo/PII em claro na fila.** A outbox mora no armazenamento do app; dado sensível
  em repouso é cifrado (`seguranca-mobile.md`, secure storage).

## 3. Sincronização — pull, push e delta

- **Delta sync, não full-refresh.** Sincronize por **cursor/timestamp/versão** (`updated_since`,
  change-token) — baixe só o que mudou. Full-refresh a cada abertura queima bateria, dados e
  bateria do servidor.
- **Paginação e limites:** sync incremental paginado; nunca "baixa tudo" num device com pouca
  memória/armazenamento.
- **Tombstones para deleção:** deletar é um evento sincronizável (marca `deleted_at`), senão o
  item apagado no servidor "ressuscita" no device. Soft-delete no protocolo de sync.
- **Relógio não confiável:** o relógio do device mente (fuso, ajuste manual). Use **versão do
  servidor**/vetor/HLC pra ordenar, não o wall-clock do cliente como verdade.

## 4. Conflitos — a decisão tem de ser explícita

Dois lados editaram o mesmo dado offline. **Não existe default seguro universal** — a política é
uma **decisão de produto que vira ADR**:

- **Last-write-wins (LWW):** simples, mas **perde escrita**. Só para dado onde perder é aceitável
  (ex.: preferência de UI). Nunca para dinheiro, estoque, texto do usuário.
- **Merge por campo / três-vias:** reconcilia campo a campo contra a `base_version`. Bom para
  formulários/registros estruturados.
- **CRDT** (ex.: texto colaborativo, contadores): convergência sem coordenação — quando o domínio
  pede edição concorrente de verdade.
- **Resolução pelo usuário:** quando o merge automático perde intenção, **mostre o conflito** e
  deixe o usuário escolher. Nunca descarte a escrita dele em silêncio.
- **Autoridade do servidor:** operação sensível (saldo, cupom, permissão) → o cliente propõe, o
  **servidor decide** e responde com o estado canônico; conflito vira "sua ação foi rejeitada
  porque…", não um merge otimista aceito no escuro.

> Piso: **nenhuma escrita do usuário some sem ele saber.** Conflito resolvido descartando dado é
> só legítimo se a política diz explicitamente que aquele dado é descartável — e isso está no ADR.

## 5. UX de estado de rede (honesta)

- **Mostre o estado real:** offline / sincronizando / sincronizado / falha. Nunca finja "salvo"
  quando está só na fila local não confirmada.
- **Degrada, não bloqueia:** o que exige rede (pagamento, verificação, ação sensível) fica
  claramente indisponível offline com mensagem — o resto do app continua usável (mesma filosofia
  de "degrada, não bloqueia" do IAM).
- **Sem spinner infinito:** timeout + estado de erro acionável em toda operação de rede.

## 6. Testes de offline/sync (entram na DoD)

- **Testes de reconciliação:** cria conflito determinístico (edita A e B a partir da mesma base) e
  prova que a política resolve como especificado — teste que **falha** se o merge regride.
- **Idempotência:** reenvio da mesma mutação não duplica no servidor (teste de integração).
- **Corte de rede no meio:** app killado com fila pendente → reabre → fila sobrevive e drena.
- **Relógio adverso:** device com relógio adiantado/atrasado não corrompe a ordenação.
- **Airplane-mode no CI/manual:** roteiro de QA (`/mobile-offline`) que exercita o app 100%
  offline e valida que nada trava nem perde dado.
