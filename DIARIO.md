# Diário de mudanças — Escola Passarinho (app 1 / index.html)

> Registro cronológico do que foi feito. **Adicionar entrada nova no topo** a cada mudança
> (data, o que mudou e por quê), no mesmo commit da alteração.

---

## 2026-09-29
- **Card "Ocultar Vagas"** (seção Faltas & Reposições, abaixo de Cancelar Aula). Permite esconder dos alunos, de forma antecipada, as vagas de uma aula futura específica (turma+data) ou de uma turma inteira "até segunda ordem". Novo array `vagasOcultas`; `isVagaOculta` filtra só a visão do aluno (professor continua vendo/usando). Actions `HIDE_VAGAS`/`UNHIDE_VAGAS` com log; lista com "Reexibir"; badge "oculta dos alunos" no painel do professor reflete as regras.

## 2026-08-27
- **Escolha comum/sem antecedência para falta de aula no mesmo dia**: o modal de classificação passou a aparecer também para aulas de hoje cujo horário já passou (antes caíam direto como comum).
- **Ordem das colunas** da tabela de contagens: crédito extra antes de férias.
- **Faltas em vermelho** na tabela quando o aluno tem falta pendente expirando em ≤7 dias; tabela adicionada também à **Turma Extra**.
- **Tabela de contagens por aluno** na seção Turmas (faltas ativas, reposições agendadas, créditos de férias e extra), com ⓘ/legenda; reescrita em CSS grid para o nome ocupar a largura do maior nome e os números se espaçarem.

## 2026-08-15
- Painel do aluno: tag "Pendente" → **"A marcar"**; alerta de expiração passa a mostrar "em X dias" em vez da data.

## 2026-08-12
- **Aviso ao aluno ao marcar reposição em aula <2h**: confirmação extra avisando que, se cancelar depois, perde o crédito/direito (vale p/ falta, férias, crédito extra).

## 2026-08-11
- **Bloqueio de nome repetido** de aluno na mesma turma (ao adicionar e ao mover de turma); feedback na UI + trava no reducer (`ADD_ALUNO`, `CHANGE_TURMA_ALUNO`).

## 2026-08-10
- **Expiração automática** de férias (quando o crédito vence) e de créditos extras não usados (7 dias após a data); vão para estatística e entram no log como "Sistema". Faltas pendentes expiradas também passam a logar. Blindado contra expirar item vinculado a reposição pendente.
- **Correção de homônimos**: aula extra e reposição sem falta vinculada usavam a 1ª turma com aquele nome; passam a usar `action.turmaOrigemId`. Histórico do aluno desambigua melhor (cobre ex-alunos).
- **Prioridade de reposição por expiração**: consome o direito que vence antes (falta/férias/extra), independente do tipo; mesmo recurso no fluxo do aluno e do professor (professor consome automaticamente — antes não consumia crédito sem seleção manual).

## 2026-08-05 / 08-06
- **Modo manutenção**: botão no fim da aba Turmas (só Pedro). Bloqueia alunos e demais professores (tela "Em manutenção" + recusa de gravação); flag no estado viaja pelo SSE (tempo real, mesmo em PWA salvo). Expira sozinho em 6h. Não entra no log.
- **Toast "Desfazer"** em todas as ações de dados (5s): restaura o estado anterior por snapshot, sem passar pelas políticas de cancelamento. Dispensado quando chega sync externo.

## 2026-07-10 a 07-16
- **Turma Extra**: botões Notas e Gerenciar; corrige labels "undefined undefined" (helper `turmaShortLabel`).
- Remove bloco de **Alertas** do Painel (a pedido).
- **Resumo do Dia** navegável (setas + calendário nativo) com snapshots imutáveis (`resumosDiarios`, ~3 meses); hoje/futuro ao vivo, passado lê snapshot; mostra Recesso/Feriado; altura fixa p/ a página não pular; blindado contra snapshot incompleto; correção de persistência (o SYNC apagava snapshots recém-congelados → passou a mesclar).
- **Cancelar aula inteira**: marca todos como falta regular, não abre vaga, desfaz reposições visitantes sem punição; reversível (`aulasCanceladas`).
- **Retenção do log = 1 ano** (MAX_LOG 6000) — antes travava em ~15 dias por causa do teto de 150.

## 2026-07-01 / 07-09
- **`CHANGE_TURMA_ALUNO`** implementado (a action não existia — trocar turma não fazia nada); migra faltas, reposições, ausências, acessos, vagas e **créditos** para a nova turma. Migration retroativa reassocia créditos órfãos de trocas antigas.
- **Faltas órfãs** de reposição cancelada sem crédito passam a expirar p/ estatística (causa raiz no `CANCEL_REPOSICAO_SEM_CREDITO`).
- Modal **Link do aluno**: botões Abrir link e QR code.

## 2026-06-16 a 06-26
- Favicon/apple-touch-icon no diagnostic.html; tela cheia iOS.
- **Faltas retroativas** (professor seleciona datas passadas até 3 meses) com escolha comum/sem antecedência.
- Cancelamento <24h e >2h converte crédito p/ "sem antecedência" em vez de perder.
- **Log**: distingue aluno de professor pelo campo `origem` (não pelo nome); logs para remover crédito, criar/excluir turma.
- **Bloco de notas por turma** com badge de não lidas.
- **Resumo de hoje** mostra nº de alunos esperados; corrige contagem dupla (aluno repondo na própria turma).
- Pergunta tipo (reposição/aula extra) ao selecionar vaga sem falta pendente.

## Antes de jun/2026 (base, resumido)
- App de faltas/reposições: turmas, faltas, reposições, vagas, ausências/férias (crédito de reposição), créditos extras, acessos (link do aluno), log, estatísticas.
- Firebase RTDB (REST + SSE) com controle de concorrência otimista; snapshots automáticos horários.
- Recuperação de dados após sobrescrita por cache antigo (incidente jun/2026); regras do Firebase corrigidas (`==`, path `state`).
