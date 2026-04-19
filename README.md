# Finanças

App pessoal de controle financeiro mensal — single-file, offline-first, roda no navegador sem backend, sem build, sem dependências.

## Features

- **Contas** com status (em aberto, pago, pago por terceiros, postergado), tipo (fixa, variável, eventual), parcela, débito automático e flag de recorrência
- **Postergar conta** aplicando juros/multa em %: a original sai do saldo do mês e uma cópia corrigida vai pro próximo mês (cancelada automaticamente se você pagar a original antes de fechar)
- **Fechar mês**: grava histórico, replica contas recorrentes, materializa postergadas, zera entradas e avança a competência
- **Entradas** do mês (ex: salário) com correção a qualquer momento
- **Resumo** com saldo (entradas − pagas por você − em aberto), despesas e sugestão de investimento conforme faixa de saldo
- **Alertas** de contas vencendo, vencidas, última parcela, postergadas e sem valor definido
- **Histórico** dos últimos meses com gráfico em canvas (barras de entradas vs. saídas)
- **Busca** por nome, filtros por status e ordenação (status, nome, valor, vencimento, parcela)
- **Backups**: criação automática 1x/dia no login + manual sob demanda, até 30 backups com cleanup FIFO, restore de backup local ou de arquivo externo (sempre com pre-restore automático)
- **Export CSV** de contas do mês ou do histórico mensal (BOM UTF-8, separador `;`, abre direto no Excel)
- **PWA instalável** no iPhone/Android, funciona offline após a primeira visita e notifica quando há nova versão

## Stack

- HTML + CSS + JavaScript puro, tudo em `index.html`
- Sem framework, sem build step, sem dependências externas
- Persistência: **IndexedDB** (`cfp_db_v2`)
- Service Worker (`sw.js`) com cache versionado
- Manifest PWA (`manifest.webmanifest`)

## Como usar

**Opção 1 — local**: clonar o repo e abrir `index.html` no navegador.

**Opção 2 — PWA**: acessar a versão publicada no GitHub Pages e usar "Adicionar à Tela de Início" (iOS Safari) ou "Instalar app" (Chrome/Android).

A tela de login pede uma senha inicial (definida na primeira abertura). Troque em **Menu → Alterar Senha** antes de usar de verdade.

## Arquitetura

### Modelo de dados

Tudo vive num único objeto `S` persistido em IndexedDB:

```
S = {
  comp:      { mes: 1..12, ano: number },   // competência atual
  entradas:  number,                         // soma das entradas do mês
  contas:    Conta[],
  historico: HistoricoRow[],
  log:       LogEntry[],                     // últimas 20 atividades
  sortCol?:  string,
  sortDir?:  'asc' | 'desc'
}

Conta = {
  id:      string,         // crypto.randomUUID()
  nome:    string,         // máx 80 chars
  valor:   number | null,  // null = "a definir"
  venc:    string | null,  // 'YYYY-MM-DD'
  tipo:    'Fixa' | 'Var.' | 'Eventual',
  parcela: string,         // 'N/A' | 'Acumulado' | 'n/N'
  debito:  'Sim' | 'Não',
  rec:     boolean,        // recorrente: replica ao fechar mês
  status:  'aberto' | 'pago' | 'terceiros' | 'postergado',
  pendingCopia: PendingCopia | null
}

PendingCopia   = { nome, valor, venc, tipo, parcela, debito }
HistoricoRow   = { mes, ano, entrada, saida, saldo }
```

### Persistência (IndexedDB)

Três object stores, todos em `cfp_db_v2`:

| Store     | keyPath     | Conteúdo                                                            |
|-----------|-------------|---------------------------------------------------------------------|
| `state`   | `id`        | Registro único `{ id: 1, data: S }`                                 |
| `config`  | `key`       | Ex.: `{ key: 'pwd', value: { algo, salt, iter, hash } }`            |
| `backups` | `filename`  | `{ filename, created_at, type, size_bytes, n_contas, comp, payload }` |

### Senha e autenticação

- **PBKDF2** com 150 000 iterações, hash SHA-256, saída de 256 bits
- Salt de 16 bytes gerado com `crypto.getRandomValues`, armazenado em hex
- Formato persistido: `{ algo: 'pbkdf2', salt, iter, hash }`
- Hashes SHA-256 antigos (formato legado `string`) são aceitos e **migrados automaticamente** pra PBKDF2 no primeiro login bem-sucedido

### Fechar mês

Ao fechar a competência atual:

1. Calcula `{ entrada, saida, saldo }` do mês e faz upsert no `historico[]`
2. Avança competência: `mes + 1`, ou `mes = 1, ano++` se passar de dezembro
3. Monta a lista de contas do próximo mês:
   - Filtra `rec === true`, descartando as com parcela `n/N` onde `n === N` (quitadas)
   - Para as mantidas, incrementa a parcela (`proxParcela`)
   - Adiciona uma cópia pra cada `pendingCopia` de conta postergada
4. Substitui `S.contas` pela nova lista e zera `entradas`
5. Cria row inicial no histórico do novo mês se ainda não existir

### Postergar conta

- Valor corrigido = `valor × (1 + juros/100)`
- Status vira `postergado` e o registro ganha `pendingCopia` com os dados da cópia pro próximo mês
- Parcela `Acumulado` vira `N/A` na cópia (não faz sentido acumular acumulado)
- Se a original for paga antes do fechamento, `pendingCopia` é zerada (cópia cancelada)

### Parcelas

- `N/A` — sem controle de parcelas
- `Acumulado` — dívida rolante, não parcelada
- `n/N` — parcela atual / total. Última parcela quando `n === N`; fechar mês com rec e `n === N` quita e remove

### Service Worker

Cache versionado (`cfp-<version>`) com estratégia por tipo de request:

- **HTML / navegação** → network-first; em falha, cai pro cache e, em último caso, serve `./index.html` (permite rotas diretas offline)
- **Outros assets mesmo-origem** → cache-first; se miss, busca da rede e cacheia respostas `ok && type === 'basic'`
- **Update**: ao detectar nova versão, o app mostra um toast persistente com link *"recarregar"* que envia `postMessage('skipWaiting')` e chama `location.reload()`

A cada release, `APP_VERSION` (em `index.html`) e `CACHE_VERSION` (em `sw.js`) **precisam** ser bumpados juntos — senão o SW antigo serve HTML velho indefinidamente.

### Convenções de UI

- **Event delegation** — um único listener global em `document`; handlers no registry `ACTIONS`, selecionados por `data-act`. Args auxiliares vêm em `data-id`, `data-target`, `data-file`, `data-f`, `data-s`. Não há `onclick` inline no HTML
- **`confirmDialog({ title, msg, ok, cancel, danger })`** — retorna `Promise<boolean>`, substitui o `confirm()` nativo
- **`withLoading(msg, fn)`** — envolve uma função async exibindo overlay com spinner até resolver/rejeitar
- **`esc(str)`** — escapa `& < > " '` antes de qualquer interpolação em `innerHTML`
- **Validação em tempo real** — `setFieldError(id, msg)` marca o input com classe `.invalid` e mostra `.field-err` abaixo; bindings em `bindValidations()`
- **Teclado** — Enter num modal aciona o botão com `data-primary="1"`; Escape fecha o modal do topo (exceto `m-loading`)
- **ARIA** — `initUI()` aplica `role="dialog"`, `aria-modal="true"` e `aria-labelledby` em todos os overlays

## Versão

**v5.0** — arquivo único em produção na raiz, PWA completo, backups automáticos, gráfico de histórico.
