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
- Persistência: **IndexedDB** (`cfp_db_v2`) com três stores: `state`, `config`, `backups`
- Service Worker (`sw.js`) com cache versionado — network-first pra HTML, cache-first pra assets
- Manifest PWA (`manifest.webmanifest`)

## Como usar

**Opção 1 — local**: clonar o repo e abrir `index.html` no navegador.

**Opção 2 — PWA**: acessar a versão publicada no GitHub Pages e usar "Adicionar à Tela de Início" (iOS Safari) ou "Instalar app" (Chrome/Android).

A tela de login pede uma senha inicial (definida na primeira abertura). Troque em **Menu → Alterar Senha** antes de usar de verdade.

## Segurança e privacidade

- 100% local: nenhum dado sai do dispositivo, não há servidor
- Senha armazenada com **PBKDF2 + salt** (150k iterações, SHA-256), com migração automática de hashes SHA-256 legados no primeiro login
- IDs gerados com `crypto.randomUUID()`
- Escape de HTML em toda a interpolação em DOM para evitar XSS

## Versão

**v5.0** — arquivo único em produção na raiz, PWA completo, backups automáticos, gráfico de histórico.

A cada release, `APP_VERSION` (em `index.html`) e `CACHE_VERSION` (em `sw.js`) são bumpados juntos pra garantir que o service worker invalide o cache antigo.
