# Escola Passarinho — Gestão de Faltas e Reposições

## Visão geral

App web single-page para gestão de faltas e reposições de uma escola de artes. Construído em React 18 + Babel standalone + Tailwind CDN, tudo em um único arquivo HTML. Dados persistidos no Firebase Realtime Database via REST API + SSE.

---

## Arquivos

| Arquivo | Descrição |
|---|---|
| `/Volumes/PBOT HD/DOWNLOADS 2/atelie-passarinho-dev.html` | Arquivo de desenvolvimento — **sempre editar aqui** |
| `/Volumes/PBOT HD/DOWNLOADS 2/atelie-passarinho-repo/index.html` | Cópia para GitHub Pages — **nunca editar diretamente** |

**Regra obrigatória:** toda alteração no dev deve ser imediatamente sincronizada com o repo e publicada:
```bash
cp "/Volumes/PBOT HD/DOWNLOADS 2/atelie-passarinho-dev.html" "/Volumes/PBOT HD/DOWNLOADS 2/atelie-passarinho-repo/index.html"
cd "/Volumes/PBOT HD/DOWNLOADS 2/atelie-passarinho-repo"
git add index.html
git commit -m "mensagem"
git push
```

---

## Firebase

- **URL:** `https://atelie-passarinho-default-rtdb.firebaseio.com`
- **Endpoint de estado:** `/state.json`
- **Segredo de escrita:** `109` (campo `_auth` em todo PUT)
- **Leitura:** pública (sem auth)
- **Escrita:** requer `_auth: '109'` no payload, validado pela regra no path `state`

### Regras de segurança (corretas):
```json
{
  "rules": {
    ".read": true,
    "state": {
      ".write": "newData.child('_auth').val() == '109'"
    }
  }
}
```
> ⚠️ Firebase Rules usa `==` não `===`. Regra deve ser no path `state`, não na raiz.

### Função de escrita no app:
```js
const DB_URL = 'https://atelie-passarinho-default-rtdb.firebaseio.com';
const DB_WRITE_SECRET = '109';
function saveToFirebase(payload) {
  return fetch(DB_URL + '/state.json', {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ...payload, _auth: DB_WRITE_SECRET })
  });
}
```

---

## Estrutura do estado (Firebase /state.json)

```js
{
  turmas: [{ id, diaSemana, horario, capacidade, alunos[], observacao }],
  faltas: [{ id, alunoNome, turmaId, datas[], status, reposicaoId, semAntecedencia? }],
  reposicoes: [{ id, alunoNome, faltaId, turmaOrigemId, turmaReposicaoId, dataReposicao,
                 realizada, vagaConsumedFaltaId?, vagaConsumedAusenciaId?,
                 vagaExtra, semVagaOficial, tipo, criadoPor?, criadoEm? }],
  vagas: [{ id, turmaId, data, faltaId?, ausenciaId?, vagaExtra? }],
  ausencias: [{ id, alunoNome, turmaId, mesAno, tipo, creditoReposicao, creditoUsado }],
  acessos: [{ codigo, alunoNome, turmaId }],
  log: [{ id, ts, professor, descricao }],
  estatisticas: { faltasExpiradas[] },
  _updatedAt: timestamp
}
```

---

## Constantes importantes

```js
STORAGE_KEY = 'atelie_passarinho_v3'   // cache localStorage
PROFESSOR_PIN = '109'                   // PIN de acesso ao painel
PROFESSOR_KEY = 'atelie_professor'      // localStorage: professor logado
PROFESSORES = ['Carol', 'Gláucio', 'Laura', 'Pedro']
```

---

## Lógica central

### `normalizeState(data)`
Roda em todo load/sync. Contém todas as migrations de dados. Sempre termina chamando `computeVagasExtras`. Ordem das migrations importantes:
1. Corrige ausências em meses de recesso (crédito = 0)
2. Recria vagas para faltas pendentes sem vaga
3. Liga reposições `semVagaOficial`/`vagaExtra` a vagas reais disponíveis
4. Liga reposições antigas a ausencias
5. **Deduplica `vagaConsumedFaltaId`** (race condition: dois alunos reservando mesmo slot)
6. **Recria faltas órfãs** (vagas com faltaId que não existe mais no array de faltas)
7. Remove vagas em datas de recesso
8. Prune log (90 dias)
9. Recomputa vagasExtras

### `computeVagasExtras(turmas, faltas, reposicoes, vagasBase, td, ausencias)`
Calcula quantas vagas extras gerar por turma/data:
```
target = max(0, 8 - alunosEfetivos + faltasCount - totalRepos - existingFaltaVagas)
```

### `isRecesso(dateStr)`
Verifica datas de recesso do ateliê (ano-agnóstico): julho, dezembro/janeiro.

### `CLEANUP` e `SYNC_FROM_FIREBASE`
Limpam faltas marcadas cuja reposição já foi realizada **E cujas datas de falta já passaram**:
```js
!(f.status === 'marcada' && f.reposicaoId && realizadasIds.has(f.reposicaoId) && f.datas.every(d => d < td))
```
> ⚠️ O `&& f.datas.every(d => d < td)` foi adicionado para evitar que vagas sumam antes da aula acontecer.

---

## Painel do professor

- Acessível pelo PIN `109`
- 3 abas: **Turmas**, **Faltas & Reposições**, **Painel**
- Apenas `Pedro` logado em desktop (`pointer: fine`) vê o botão de editar o log
- **RESUMO DE HOJE** — card expansível no final da aba Painel, mostra todas as turmas do dia com presentes, faltas e reposições
- Renomear aluno: clicar no nome na seção Turmas → input inline → Enter salva, Esc cancela. A action `RENAME_ALUNO` propaga para faltas, reposições, ausências e acessos.

---

## Painel do aluno

- Acessado via `?c=CODIGO` na URL
- Mostra faltas ativas com tag colorido:
  - **Reposta** (verde): reposição já realizada
  - **Marcada** (azul): reposição agendada mas não realizada
  - **Pendente** (âmbar): sem reposição ainda
- Vagas disponíveis para booking de reposição

---

## Ícone / PWA

- Favicon embutido em base64 no `<link id="favicon">`
- iOS: `<link rel="apple-touch-icon">` criado dinamicamente referenciando o mesmo href do favicon
- Header do app usa `<img src={document.getElementById('favicon').href}>` — não duplica o base64
- Android funciona automaticamente com o favicon

---

## Recesso do ateliê

Períodos (ano-agnósticos):
- **Julho** completo
- **Dezembro** (a partir de ~20/12) e **Janeiro** (até ~20/01)

Ausências em meses de recesso têm `creditoReposicao: 0` automaticamente.
Vagas em datas de recesso são removidas por migration.
No painel de ausências: datas de recesso mostram 🏠 em vez de tachado.

---

## Bugs corrigidos nesta sessão (jun/2026)

### Race condition de vagaConsumedFaltaId duplicado
Dois alunos reservando o mesmo slot simultaneamente resultavam em dois registros com o mesmo `vagaConsumedFaltaId`. Migration em normalizeState detecta duplicatas e reatribui a segunda reposição à próxima vaga real disponível.

### Falta sumindo antes da aula (Bia, Qui 09h 28/05)
Falta de aluno que repôs ANTES da data da aula era limpa prematuramente. Fix: só limpar quando `f.datas.every(d => d < td)`.

### Vagas órfãs (Laura e Mariana, Qua 19h 03/06)
Vagas com `faltaId` que não existe mais no array de faltas. Migration recria a falta a partir dos dados da reposição vinculada, para que o professor veja quem está ausente.

### Regras Firebase apagando dados
Regras com `===` (inválido no Firebase Rules) e no path errado causaram bloqueio de writes. Correto: `==` no path `state`.

---

## Deploy

- **GitHub Pages:** https://pebot09.github.io/atelie-passarinho/
- **Repo:** https://github.com/pebot09/atelie-passarinho
- Branch: `main`
- Todo commit vai direto pro ar — não há staging/CI
