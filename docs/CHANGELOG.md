# Changelog

## [1.1.7] — 2026-11-13

### Adicionado
- O card **Sem autorização** agora é clicável: rola até a seção de cartões cadastrados fora do grupo e abre automaticamente os veículos, facilitando o vínculo dos cartões de carros que mudaram de regional.

## [1.1.6] — 2026-11-10

### Adicionado
- Histórico de tentativas de envio à driver list no período selecionado, incluindo falhas e envios parciais.
- Resultado do último envio junto ao motorista na seção de cartões sem autorização.

### Corrigido
- A placa expandida continua aberta após o envio; o feedback não desaparece ao recriar a tabela.
- O texto distingue comando aceito pela API de vínculo aplicado no equipamento, que depende da conexão do GO.
- Correção do ID retornado ao criar o registro `AddInData`.

Todas as mudanças relevantes deste projeto são documentadas aqui.
Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/) e
[Versionamento Semântico](https://semver.org/lang/pt-BR/).

## [1.0.9] — 2026-11-09

### Adicionado
- Persistência com `AddInData`: o período selecionado é restaurado ao reabrir o add-in.
- **Ledger de vínculos**: cada envio bem-sucedido à authlist é registrado (veículo, motorista,
  cartões, placa, data/hora e usuário). A tabela mostra "✓ vinculado em dd/mm por usuário" e o
  botão passa a "Reenviar", evitando duplicidade — já que a authlist não é legível pela API.

### Observações
- Requer `Get`/`Add`/`Set` em `AddInData`; sem permissão, o add-in segue funcionando em memória.

## [1.0.8] — 2026-11-09

### Adicionado
- Ação **Vincular cartões** (por placa) e **Vincular** (por linha) para enviar o cartão à
  *Authorized Driver List* (authlist) do equipamento via `TextMessage`/`DriverAuthListContent`.

### Observações
- A authlist não é legível pela API; o add-in envia o comando, mas o status continua baseado em grupos.
- Requer authlist habilitada no GO e permissão de `Add` em `TextMessage`.

## [1.0.7] — 2026-11-09

### Alterado
- A seção "Leituras válidas" passou a exibir apenas as leituras **autorizadas**; as **sem autorização** ficam apenas na seção própria (fim da duplicação).
- KPI "Leituras válidas" agora detalha "N autorizadas • M fora do grupo".

### Adicionado
- Tooltip no status "Não autorizado" mostrando os grupos do motorista e do veículo (diagnóstico).

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
