# Dueto — Documentação Técnica

PWA de finanças pessoais para casais, em arquivo único (`index.html`), hospedado em `rodrig0almeida.github.io`. Este documento existe pra tirar as explicações mais longas de dentro do HTML e manter o arquivo do app mais enxuto.

Versão coberta por este documento: **3.8.1**

---

## 1. Visão geral

O Dueto organiza a vida financeira em **cofres** (vaults) — potes de dinheiro com nome, saldo e meta opcional — dentro de um **mês** de referência (renda, despesas fixas/variáveis, empréstimos). Tudo sincroniza com uma planilha Google Sheets via Google Apps Script.

A partir da v2.8, o app ganhou **espaços pessoais**: além do espaço "Casa" (compartilhado pelo casal), cada pessoa pode ter cofres, despesas e empréstimos só dela, escondidos da Casa e do outro perfil.

---

## 2. Modelo de dados

Estado global em memória, cada um com sua função `persistX()` que salva local e agenda envio à nuvem:

| Variável | O que guarda |
|---|---|
| `vaults` | Cofres: `{id, name, emoji, balance, goal, ownerId, isPersonalDefault, sourceBalance, createdAt}` |
| `txns` | Histórico de movimentações de cofre: `{id, vaultId, type, amount, note, billId?, loanId?, ts}` |
| `loans` | Empréstimos de pessoa a cofre: `{id, vaultId, who, total, remaining, installments, paidInstallments, startMonthKey, note, ts}` |
| `bills` | Despesas/contas: `{id, name, defaultAmount, kind, targetVaultId?, payFromVaultId?, payToVaultId?, createdMonthKey, dueMonthKey?, endMonthKey?, ownerId}` |
| `monthData` | Objeto por mês (`"2026-07"` etc.): `{income, billOverrides, billsPaid, hiddenBills, vaultTransfers, fairExpenses, incomeTs}` |
| `names` | `{a, b, photoA, photoB, _ts}` — nomes/fotos dos dois perfis principais |
| `appSettings` | `{extraProfiles}` — perfis C, D... criados via "+ Adicionar perfil" |
| `localSettings` | Preferências só do aparelho (tema, Modo Dev, `currentProfile`) — **nunca sincronizado** |

### Chaves sincronizadas
`SYNC_KEYS = ['vaults','txns','loans','bills','monthData','names','appSettings']` — qualquer mudança nessas dispara `scheduleCloudPush()` (debounce de 2s).

### `ownerId` — o campo que faz tudo funcionar
- `null`/ausente → pertence à **Casa** (compartilhado).
- `'a'` ou `'b'` → pertence ao perfil pessoal daquela pessoa.
- Vaults, bills e (indiretamente, via `who`/vault) loans respeitam isso.

---

## 3. Espaços pessoais (Casa / Pessoa A / Pessoa B)

- `currentScope()` lê `localSettings.currentProfile` (`'casa'` por padrão).
- `inScope(ownerId)` decide se um item pertence ao espaço ativo.
- `visibleVaults()` = `vaults.filter(v => inScope(v.ownerId))`.
- Seletor de espaço fica em **Configurações → Ver cofres e contas de** — só aparece opção de pessoa se ela tiver nome preenchido em Perfis.
- **Renda é sempre da Casa**, nunca pessoal — não existe "renda" no espaço de uma pessoa. Pra mandar dinheiro da Casa pra uma pessoa, existe a despesa especial "Enviar para pessoa" (ver seção 5), que deposita numa carteira pessoal automática (`isPersonalDefault: true`, nome `"Carteira Pessoal — {nome}"`).
- Despesas pessoais são pagas **direto de um cofre da própria pessoa** (não do dinheiro físico/vale/conta da Casa) — ver `confirmPayPersonalBill()`.
- Empréstimos entre cofres (seção 6) permitem mão única: cofre pessoal pode emprestar pra Casa; Casa nunca vê nem pode escolher um cofre pessoal como destino.

---

## 4. Contas fixas/variáveis (`bills`)

Cada `bill` tem:
- `kind`: `'fixa'` ou `'variavel'` (variável permite ajustar valor mês a mês via `billOverrides`).
- `targetVaultId` (opcional): ao pagar, deposita automaticamente nesse cofre.
- `payFromVaultId`/`payToVaultId` (só em despesas geradas por empréstimo entre cofres — ver seção 6): pagamento sempre volta pro cofre de origem, sem escolha de destino.
- `createdMonthKey`: primeiro mês em que a conta aparece.
- `endMonthKey` (opcional): último mês em que aparece — usado pra "Repetir: Só este mês" ou "Por X meses". Sem isso, repete pra sempre.

### Como uma conta é removida
No botão ✕ → sheet com duas opções:
- **Ocultar só este mês** (`hideBillThisMonth`) — marca `entry.hiddenBills[id]=true` daquele mês; continua ativa nos outros.
- **Apagar de vez** (`deleteBill`) — remove de `bills` e limpa referências (`hiddenBills`/`billsPaid`/`billOverrides`) de todos os meses.

### `pruneOrphanedBillData()`
Roda a cada carregamento: remove entradas de `billOverrides`/`billsPaid`/`hiddenBills` que apontam pra um `billId` que não existe mais em `bills` (resíduo de exclusões antigas, de antes dessa limpeza existir).

---

## 5. Movimentação de dinheiro entre cofres e pessoas

| Ação | Função | O que faz |
|---|---|---|
| Guardar / Retirar | `confirmMove()` | Ajusta saldo do cofre; guardar pede a origem (conta/físico/vale) |
| Enviar (transferir) | `confirmTransfer()` | Move saldo entre dois cofres, ou pra "conta"/"dinheiro" (sai do sistema de cofres) |
| Emprestar a pessoa | `confirmLoan()` | Pessoa tira dinheiro do cofre; gera registro em `loans` |
| Emprestar a outro cofre | `confirmVaultLoan()` | Cofre A empresta a Cofre B; cria despesa "Devolver empréstimo" com vencimento **no mês seguinte à data real de hoje** (não ao mês navegado) |
| Enviar para pessoa (despesa) | `getOrCreatePersonalVault()` + pagamento normal | Casa paga uma despesa com destino à carteira pessoal automática de alguém |
| Distribuir sobra pelas metas | `autoDistributeLeftover()` | Cria despesa "Guardar para meta" **e já marca como paga na hora** (evita duplicar se tocado de novo); prioriza pagar de conta bancária |

### Composição por origem (`sourceBalance`)
Cada cofre guarda `{fisico, vale, conta}` — quanto do saldo veio de cada fonte. Mostrado no detalhe do cofre (`detailSourceBreakdown`). Saldo antigo sem origem rastreada é exibido como "conta bancária" (só na exibição, não altera o valor real guardado).

Funções auxiliares: `addVaultSource()`, `addVaultSourceSplit()`, `removeVaultSourceProportional()` (retiradas/pagamentos sem origem exata — remove proporcionalmente à mistura atual), `moveVaultSourceProportional()` (transferências/empréstimos entre cofres — leva a "cor" do dinheiro junto).

---

## 6. Empréstimos entre cofres

`confirmVaultLoan()`: tira valor de um cofre, deposita em outro (Casa↔Casa, ou Pessoal→Casa), e cria uma despesa **`payFromVaultId`/`payToVaultId`** com vencimento no mês seguinte — pagar essa despesa devolve o dinheiro automaticamente, sem cofre de destino escolhível (sempre volta pro cofre que emprestou).

---

## 7. Sincronização com a nuvem

Arquitetura: Google Apps Script expõe `doGet()` (lê célula) e `doPost()` (sobrescreve célula) numa planilha. Sem autenticação própria — a segurança é a URL ser secreta.

### Pull-before-push (v3.5–3.6)
Antes de **cada envio**, `pushToCloud()` chama `pullAndMergeBeforePush()`:
1. Busca (GET) o estado atual da nuvem.
2. **Arrays com ID** (`vaults`, `txns`, `loans`, `bills`) e entradas de `billsPaid`/`billOverrides`/`hiddenBills`: qualquer item que só existe na nuvem (adicionado por outro aparelho) é somado ao local — nunca sobrescreve o que já existe local.
3. **Campos de valor simples sem ID** (renda por pessoa/fonte, nomes e fotos de perfil): cada campo tem um carimbo de data (`incomeTs`, `names._ts`) atualizado a cada edição. No merge, quem editou por último vence, campo a campo.
4. Só depois disso monta e envia o payload.

Isso reduz bastante (mas não elimina 100%) o risco de dois aparelhos sincronizando por perto sobrescreverem um ao outro.

### Timeout e retry
- `SYNC_TIMEOUT_MS = 15000` — todo fetch tem `AbortController` com esse limite.
- Falhas muito rápidas (<500ms) são tratadas como rate-limit/cold-start do Apps Script e tentam de novo automaticamente (até `SYNC_MAX_RETRIES = 2`) com backoff crescente.

### Log de sincronização
`syncLog` (array, local-only, últimos 50 eventos) — visível em **Configurações → Modo Dev → Log de sincronização**, com exportar (`navigator.share` ou download) e limpar.

### Criptografia + biometria (opcional, Modo Dev)
- AES-GCM 256 bits (Web Crypto nativa) + PBKDF2 pra derivar chave da senha.
- Face ID/digital via WebAuthn só libera a senha já guardada localmente — não substitui a senha.
- Ao ativar, `pushToCloud()` cifra o payload inteiro antes de enviar. **Nesse modo, o pull-before-push é pulado** (não decifra no meio do envio) — payload da nuvem é usado como está, sem merge, se estiver cifrado.

### Diálogos de confirmação
`window.confirm()` nativo pode ser **bloqueado silenciosamente** por bloqueadores de conteúdo/extensões — foi a causa real de "não consigo apagar X" reportado algumas vezes. Por isso, ações destrutivas frequentes (excluir cofre, excluir despesa, excluir/remover perfil) usam **sheets de confirmação próprias do app** em vez de `confirm()`. Ainda restam alguns `confirm()` nativos em ações menos críticas (avisos de saldo negativo, etc.) — candidatos a receber o mesmo tratamento se algum dia derem o mesmo problema.

---

## 8. Verificação de atualização

`checkForUpdateOnBoot()`, chamada ao final de `loadAll()`: busca a própria página (`cache: 'no-store'`), extrai `APP_VERSION` via regex, e se for diferente da versão rodando, mostra uma faixa breve ("🆕 Atualizando para vX…") e recarrega sozinho ~1,2s depois — sem perguntar. Usa `sessionStorage` pra não entrar em loop se o recarregamento não pegar a versão nova por algum motivo de cache.

---

## 9. Limitações conhecidas

- **Merge de valores simples** cobre renda e nomes/fotos, mas não cobre todo campo solto do app (ex: taxas, preferências dentro de `appSettings` além de `extraProfiles`) — esses continuam "local sempre vence" no pull-before-push.
- **Perfis extras (C, D...)** criados por "+ Adicionar perfil" são mais leves que A/B — não participam de renda, cofres pessoais ou escopo Casa/Pessoal, servem só como identidade/avatar extra (ex: em empréstimos de pessoa).
- **Composição por origem** em cofres muito antigos pode ter uma parte "não rastreada" atribuída à conta bancária por padrão de exibição, já que o rastreio só existe a partir de quando foi implementado.
- Ainda existem alguns `window.confirm()` nativos em fluxos secundários — ver seção 7.

---

## 10. Histórico resumido de versões relevantes

| Versão | O que mudou |
|---|---|
| 2.8.x | Espaços pessoais (Casa/Pessoa A/Pessoa B), seletor de perfil |
| 2.9.x | Renda voltou a ser só da Casa; empréstimos visíveis por quem pegou, não só por cofre |
| 2.10.x | Despesas com duração (todos os meses / só este mês / por X meses) |
| 3.0–3.2 | Perfil B/perfis extras opcionais e removíveis; log de sync, timeout/retry, criptografia + Face ID (portados do Corrida+); correções de modo escuro |
| 3.3–3.4 | Botão de distribuir sobra pelas metas; meta editável/desativável manualmente; correção de bug de transações duplicadas (match por ID em vez de texto) |
| 3.5–3.6 | Pull-before-push com merge por ID e por carimbo de data |
| 3.7 | Correção: quitar vários empréstimos agora credita o cofre de verdade; opção de ocultar despesa só no mês |
| 3.8 | Verificação de atualização automática ao abrir o app |
