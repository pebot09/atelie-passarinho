# Escola Passarinho — Gestão de Faltas e Reposições

## Visão geral

App web single-page para gestão de faltas e reposições de uma escola de artes (cerâmica).
React 18 + Babel standalone + Tailwind CDN, **tudo em um único `index.html`**. Dados no
Firebase Realtime Database (REST + SSE). Sem build, sem CI: todo commit na `main` vai pro ar.

> ⚠️ **Regra permanente:** a cada mudança, atualizar o `DIARIO.md` (ver seção no fim).

---

## Arquivos e deploy

| Arquivo | Descrição |
|---|---|
| `index.html` | **App 1** (faltas/reposições) — o app principal. Editar aqui. |
| `barro.html` | **App 2** (financeiro/barro) — standalone, lê `/state.json` só p/ leitura, grava em `/barro.json`. Outra frente de trabalho; **não mexer sem necessidade**. |
| `diagnostic.html` | Página de diagnóstico (checa leitura/escrita Firebase e carga do app). |
| `CLAUDE.md` | Este arquivo. |
| `DIARIO.md` | Diário de mudanças — **sempre atualizar**. |

- **Fluxo:** editar `index.html` → `git commit` → `git push` para `main`. GitHub Pages publica sozinho em ~1-2 min.
- **Deploy:** https://pebot09.github.io/atelie-passarinho/ · Repo: https://github.com/pebot09/atelie-passarinho · Branch: `main`.
- **Atenção no push:** o remoto às vezes recebe commits do `barro.html` de outra sessão. Se o push falhar como non-fast-forward, `git fetch origin main && git rebase origin/main` (conflito é improvável, pois os dois apps são arquivos separados) e empurrar de novo. Pushes "non-fast-forward" transitórios do proxy também acontecem — reenviar costuma resolver.
- **Verificação antes de commitar:** `python3 -c "s=open('index.html').read(); print(s.count('{')-s.count('}'), s.count('(')-s.count(')'), s.count('[')-s.count(']'))"`. As chaves e colchetes devem fechar (diff 0); parênteses têm baseline **-2** (vem de strings/emojis, não é erro).
- **Sem acesso ao Firebase do sandbox:** a sessão cloud não consegue ler/gravar no Firebase (proxy bloqueia 403). Para inspecionar dados, pedir ao usuário um export de `/state.json`.

---

## Firebase

- **URL:** `https://atelie-passarinho-default-rtdb.firebaseio.com`
- **Estado:** `/state.json` · **Snapshots automáticos:** `/snapshots/...` · **App 2:** `/barro.json`
- **Leitura:** pública. **Escrita:** requer `_auth: '109'` no payload.
- `DB_WRITE_SECRET = '109'`, `PROFESSOR_PIN = '109'` — **nunca mudar** (quebraria o acesso de todos).

### Regras de segurança (as corretas, já publicadas):
```json
{
  "rules": {
    ".read": true,
    "state":     { ".write": "newData.child('_auth').val() == '109'" },
    "barro":     { ".write": "newData.child('_auth').val() == '109'" },
    "snapshots": { "$s": { ".write": "newData.child('_auth').val() == '109'" } }
  }
}
```
> ⚠️ Firebase Rules usa `==` (não `===`). `snapshots` precisa do curinga `$s` porque a escrita é num **filho** (`/snapshots/<key>`), não no nó. O DELETE de snapshots antigos é bloqueado pela regra (newData nulo) — snapshots **não** são podados no servidor; é aceito de propósito.

---

## Estrutura do estado (`/state.json`)

```js
{
  turmas:       [{ id, diaSemana, horario, capacidade, alunos[], observacao }],
  faltas:       [{ id, alunoNome, turmaId, datas[], status, reposicaoId?, semAntecedencia?, cancelamentoId?, criadoPor?, criadoEm? }],
  reposicoes:   [{ id, alunoNome, faltaId?, ausenciaId?, creditoId?, turmaOrigemId, turmaReposicaoId, dataReposicao,
                   realizada, tipo, vagaConsumedFaltaId?, vagaConsumedAusenciaId?, vagaExtra?, semVagaOficial?, semFaltaVinculada?, pago?, criadoPor?, criadoEm? }],
  vagas:        [{ id, turmaId, data, faltaId?, ausenciaId?, vagaExtra?, cancelada? }],
  ausencias:    [{ id, alunoNome, turmaId, mesAno, tipo, creditoReposicao, creditoUsado, criadoPor? }],
  creditos:     [{ id, alunoNome, turmaId, dataExpiracao, usado, criadoPor?, criadoEm? }],   // crédito extra avulso
  acessos:      [{ id, codigo, alunoNome, turmaId, watchlist? }],                              // link do aluno (?c=codigo)
  notas:        [{ id, turmaId, texto, professor, ts }],                                       // bloco de notas por turma
  aulasCanceladas:  [{ id, turmaId, data, ts, professor }],                                    // aula cancelada (dia inteiro)
  vagasOcultas:     [{ id, turmaId, data|null, ts, professor }],                               // data null = turma inteira "até segunda ordem"
  resumosDiarios:   { "YYYY-MM-DD": { data, geradoEm, turmas:[{turmaId, presentes[], faltaram[], repondo[{nome,origemTurmaId}], esperados, cancelada}] } }, // snapshots imutáveis, ~3 meses
  manutencao:   null | { ativo, ligadoEm, ligadoPor },                                         // modo manutenção (expira só em 6h)
  log:          [{ id, ts, professor, descricao, origem? }],                                   // 'origem':'aluno'|'sistema' distingue autor
  snapshots:    [],                                                                            // snapshots manuais
  estatisticas: { faltasExpiradas[], reposCanceladas[], feriasExpiradas[], creditosExpirados[] },
  _updatedAt:   timestamp
}
```

---

## Constantes importantes

```js
STORAGE_KEY = 'atelie_passarinho_v3'   // cache localStorage
PROFESSOR_PIN = '109' / DB_WRITE_SECRET = '109'   // NUNCA mudar
PROFESSOR_KEY = 'atelie_professor'     // localStorage: professor logado
PROFESSORES = ['Carol', 'Gláucio', 'Laura', 'Pedro']
MAX_LOG = 6000 / LOG_RETENTION_DAYS = 365          // log retém ~1 ano
MANUTENCAO_ADMIN = 'Pedro' / MANUTENCAO_MAX_HORAS = 6
CREDITO_EXPIRA_GRACE_DIAS = 7
TURMA_EXTRA_ID = 'turma-extra'         // "Alunos Extras & Ex-alunos" (diaSemana/horario null)
```

---

## Lógica central

### `normalizeState(data)`
Roda em todo load/sync. Contém todas as migrations (corrige recesso, recria vagas/faltas órfãs,
deduplica `vagaConsumedFaltaId`, liga reposições a vagas/ausências, reassocia créditos órfãos de
trocas de turma antigas, etc.). Sempre termina em `computeVagasExtras` + `stripVagasCanceladas`.
Mescla `resumosDiarios` (locais + servidor) para não apagar snapshots recém-congelados.

### `computeVagasExtras(...)`  →  `target = max(0, 8 - alunosEfetivos + faltasCount - totalRepos - existingFaltaVagas)`
Se todos os alunos faltam na data, não gera vaga extra.

### Prioridade de reposição — **por expiração** (helpers `reposicaoRights` / `pickReposicaoRight`)
Ao marcar reposição, consome o direito que **vence antes** (falta pendente, crédito de férias ou
crédito extra), independente do tipo. Faltas "sem antecedência" só valem dentro de 2h da aula.
**Vale igual no fluxo do aluno e do professor** (o professor consome automaticamente; `rightToActionFields`).

### Cancelamento de reposição (aluno) — 3 faixas por horas até a aula
`<2h`: perde o crédito/direito (`CANCEL_REPOSICAO_SEM_CREDITO` → falta expira p/ estatística).
`<24h`: converte para "sem antecedência". `≥24h`: normal.
Ao **marcar** reposição numa aula `<2h`, o aluno recebe um aviso extra (pode perder o crédito se cancelar).

### Expiração automática (`CLEANUP` + `SYNC_FROM_FIREBASE`, via `expireStaleItems`)
- **Faltas pendentes:** expiram 30 dias após a data; vão p/ `faltasExpiradas` + log "Sistema".
- **Férias (ausências):** removidas quando o crédito vence (`getAusenciaExpiry` = fim do mês +30d) → `feriasExpiradas` + log.
- **Créditos extras não usados:** removidos 7 dias após `dataExpiracao` → `creditosExpirados` + log.
- Nunca expira item vinculado a uma reposição **pendente** (evita órfão).

### `CLEANUP` / `SYNC_FROM_FIREBASE`
Limpam faltas marcadas cuja reposição foi realizada **E** cujas datas já passaram:
`!(f.status==='marcada' && f.reposicaoId && realizadasIds.has(f.reposicaoId) && f.datas.every(d => d < td))`.

### `isRecesso(dateStr)` — ano-agnóstico
Recesso: **12 a 19 de julho**, **dez (≥20)** e **jan (≤10)**. Vagas em recesso são removidas; ausências em recesso têm `creditoReposicao: 0`.

---

## Funcionalidades (resumo)

**Painel do professor** (PIN `109`, 3 abas):
- **Turmas:** cards por turma; ao abrir, tabela por aluno (grid) com contagens ✋ faltas ativas · 🔄 reposições agendadas · 🎟️ crédito extra · 🏖️ crédito férias (ⓘ abre legenda). Nº de faltas em **vermelho** se alguma expira em ≤7 dias. Gerenciar (renomear, mover de turma — bloqueia homônimo no destino, histórico, remover). Notas da turma. Link do aluno (copiar/abrir/QR). Bloqueia nome repetido na mesma turma. Botão **🔧 modo manutenção** (fim da aba).
- **Faltas & Reposições:** grupos Faltas (registrar — oferece escolha comum/sem antecedência para aulas de hoje e <24h; retroativas até 3 meses — / cancelar), Reposições (registrar — consome direito automático / cancelar), Férias, Crédito Extra, **Cancelar Aula** (dia inteiro), **Ocultar Vagas** (antecipado: uma aula ou turma inteira; só esconde dos alunos).
- **Painel:** Lista atual, Histórico (log; editável só por Pedro no desktop), Registro. **Resumo do Dia** (card no fim): navegável por setas/calendário; hoje/futuro ao vivo, passado lê snapshot imutável; mostra Recesso/Feriado/aula cancelada.
- **Toast "Desfazer"** (5s) em toda ação de dados — restaura o estado anterior (snapshot), sem passar pelas políticas de cancelamento.

**Painel do aluno** (`?c=CODIGO`):
- Faltas com tag: **Reposta** (verde) / **Marcada** (azul) / **A marcar** (âmbar). Alerta de expiração "em X dias".
- Marca reposição escolhendo vaga; consome o direito que vence antes. Avisos para aula <2h.
- Modo manutenção bloqueia marcações (só Pedro/admin grava).

**PWA:** favicon base64 em `<link id="favicon">`, apple-touch-icon dinâmico, tela cheia no iOS.

---

## Riscos / pontos em aberto (prioridade)

1. 🔴 **Backup fora do Firebase** é a rede de segurança contra perda/ransomware. Hoje manual (botão ⬇ Backup, só Pedro). **Recomendado automatizar.**
2. 🟡 **Segurança do `109`** (segredo público no source + leitura pública do banco): risco baixo, estrago reversível com backup. Correção definitiva = Firebase Auth real. Não urgente.
3. 🟡 **Concorrência de escrita** (lost update): sincroniza o estado inteiro; se outro dispositivo grava no mesmo instante, pode descartar a marcação local. Raro; não confirmado em produção (o episódio de "falta sumida" foi atribuído a falha de internet). Só atacar se reaparecer.

---

## Diário

Há um `DIARIO.md` com o registro cronológico das mudanças. **Toda alteração feita no app deve
adicionar uma entrada nova no topo do `DIARIO.md`** (data, o que mudou, por quê), no mesmo commit.
