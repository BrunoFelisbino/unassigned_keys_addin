# Changelog

Todas as mudanças relevantes deste projeto são documentadas aqui.
Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/) e
[Versionamento Semântico](https://semver.org/lang/pt-BR/).

## [1.0.6] — 2026-11-09

### Adicionado
- Número do cartão nas leituras válidas (via `User.keys`).
- Agrupamento das leituras válidas e dos "sem autorização" **por placa**.
- Top 10 placas recolhidas (`<details>`), com detalhes ao clicar.

### Alterado
- Tabelas de válidas/sem autorização passam a ser expansíveis e resumidas por veículo.

## [1.0.5] — 2026-11-09

### Adicionado
- Colunas de **Motorista** e **E-mail** nas leituras válidas.
- KPI **Sem autorização** e painel "Cartões cadastrados, mas sem autorização no veículo".
- Regra de autorização por grupos (motorista × veículo), ignorando o grupo raiz.

## [1.0.4] — 2026-11-09

### Adicionado
- Leitura de **cartões válidos** (`DriverChange`).
- KPIs de leituras válidas, motoristas distintos e média de validações/dia.
- Painel "Viagens validadas por dia".

## [1.0.3] — 2026-11-09

### Corrigido
- Linhas vazias causadas por eventos de segurança ("Create key", "Restore key", etc.).
- Extração de serial para cartões alfanuméricos (hex).

## [1.0.2] — 2026-11-09

### Alterado
- Config no formato compatível com o Fleet Health Score (`ActivityLink/`, `files: {}`).
- Removido `page: customReports` e `isSigned`.

## [1.0.1] — 2026-11-09

### Corrigido
- Tema claro do add-in de referência (`#f5f8fc`), corrigindo a página preta.

## [1.0.0] — 2026-11-09

### Adicionado
- Versão inicial: Audit de cartões não cadastrados, KPIs, mapa, filtros e tabela.
