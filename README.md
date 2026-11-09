# Chaves Não Cadastradas — Add-in Geotab

Add-in para **MyGeotab** que audita o uso de cartões/chaves de motorista em veículos. Mostra:

- cartões **não cadastrados** no Geotab que foram usados (com placa, número do cartão e data/hora);
- **leituras válidas** (cartões cadastrados) com nome e e-mail do motorista, aglomeradas por placa;
- cartões **cadastrados, mas sem autorização** para o veículo em que foram usados;
- KPIs, mapa dos veículos e contagem de **validações por dia**.

> Versão atual: **1.0.8** · Licença **MIT** · Autor: Bruno Felisbino

---

## Visão geral

O add-in roda dentro do MyGeotab como uma página (`ActivityLink/`) servida pelo GitHub Pages.
Toda a lógica (UI + chamadas à API do Geotab) está em um único arquivo HTML autocontido,
o que evita problemas de CORS/asset no iframe do MyGeotab.

URLs publicadas:

| Recurso | URL |
|---|---|
| Config do add-in (usar no MyGeotab) | `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_addin_config.json` |
| Página do add-in | `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys.html` |
| Ícone | `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_icon.svg` |

---

## Funcionalidades

### KPIs

| KPI | Significado |
|---|---|
| Tentativas | Registros de uso de cartão não cadastrado no período |
| Cartões não cadastrados | Seriais distintos não cadastrados |
| Veículos afetados | Veículos distintos com tentativas |
| Última tentativa | Data/hora da tentativa mais recente |
| Leituras válidas | Leituras de cartões cadastrados |
| Sem autorização | Cartões cadastrados usados em veículo fora do grupo do motorista |
| Motoristas distintos | Motoristas distintos nas leituras válidas |
| Média validações/dia | Média de leituras válidas por dia |
| Última leitura válida | Data/hora da leitura válida mais recente |

### Seções

- **Mapa dos veículos com tentativas** — posição mais recente via `DeviceStatusInfo` (Leaflet + OpenStreetMap).
- **Resumo** — barras com os cartões não cadastrados mais usados.
- **Tentativas de cartões não cadastrados** — tabela com Data/Hora, Placa, Cartão (serial), Status.
- **Viagens validadas por dia** — barras por dia com a contagem de leituras válidas.
- **Leituras válidas autorizadas (cartões cadastrados)** — **top 10 placas** aglomeradas; clique na placa para expandir os detalhes (Data/Hora, Cartão, Motorista, E-mail, Status).
- **Cartões cadastrados, mas sem autorização no veículo** — mesmo formato, com os casos fora do grupo (não se repetem na seção de autorizadas; o tooltip do status mostra os grupos do motorista e do veículo). Cada placa tem o botão **Vincular cartões** e cada linha o botão **Vincular**, que enviam o cartão para a *driver list* (authlist) do equipamento.

### Filtros

- Período: **Hoje** (padrão), 7, 30 ou 90 dias
- Grupo
- Veículo
- Busca por placa, cartão ou motorista

Os filtros valem para todas as seções.

---

## Fontes de dados (API Geotab)

| Tipo (`Get`) | Uso |
|---|---|
| `Audit` | Uso de cartão **não cadastrado**. Serial e placa são extraídos do texto (`name` + `comment`). |
| `DriverChange` | **Leituras válidas** (motorista identificado). Cada evento = uma validação de viagem. |
| `User` | Nome, e-mail, cartões (`keys[].serialNumber`) e grupos (`companyGroups`). |
| `Device` | Placa (`licensePlate` ou prefixo do `name`) e grupos. |
| `Group` | Nomes e hierarquia de grupos. |
| `DeviceStatusInfo` | Última posição dos veículos para o mapa. |

### Observações de dados

- O `Audit` traz o **serial do cartão** no texto da mensagem. Ex.:
  `Please add the driver/passenger key 1234567890, as it is currently unassigned. The key was used on vehicle ABC1D23`.
- O `DriverChange` **não** traz o serial lido. Para exibir o cartão nas leituras válidas, usamos os cartões cadastrados do motorista (`User.keys`). Se o motorista tiver vários cartões, mostramos o primeiro com `+N` (o tooltip lista todos).
- Eventos de segurança do tipo "Create key"/"Restore key"/"Saved key backup" são ignorados para não gerar linhas vazias.

### Como a autorização é calculada

Uma leitura válida é marcada como **fora do grupo** quando **não há grupo em comum** entre:

- os grupos do motorista (`User.companyGroups`), e
- os grupos do veículo (`Device.groups`).

O grupo raiz da empresa (`GroupCompanyId`) é ignorado nessa comparação para permitir separar filiais/equipes.
Se algum dos lados não tiver grupos, o registro é tratado como autorizado (evita falso positivo).

> Essa é a regra padrão. Se a sua operação usa outro critério (ex.: grupo específico por filial ou lista de motoristas por veículo), ajuste a função `isAuthorized()` em `unassigned_keys.html`.

### Vínculo automático na driver list (authlist)

Quando o veículo muda de regional, o cartão de outra regional precisa ser **re-vinculado à
Authorized Driver List** do equipamento (IOX-NFCREADERA). O add-in permite enviar esse vínculo
pela API, sem abrir o MyGeotab item a item:

- **Vincular cartões** (por placa) — envia à authlist do veículo todos os cartões distintos que
  apareceram "sem autorização" naquela placa.
- **Vincular** (por linha) — envia apenas o cartão daquele motorista.

Internamente é enviado um `TextMessage` com `messageContent` do tipo `DriverAuthList`
(`DriverAuthListContent`), usando o `driverKey` do cadastro do motorista (`User.keys`) e
`addToAuthList: true`:

```js
api.call('Add', {
  typeName: 'TextMessage',
  entity: {
    device: { id: deviceId },
    isDirectionToVehicle: true,
    messageContent: {
      contentType: 'DriverAuthList',
      driverKey: user.keys[0],   // { keyId, driverKeyType }
      addToAuthList: true,
      clearAuthList: false
    }
  }
}, success, error);
```

**Requisitos e cuidados:**

- A **authlist precisa estar habilitada** no GO (parâmetro de configuração do IOX-NFCREADERA);
  sem isso o equipamento ignora a mensagem.
- É uma operação de **escrita**: exige permissão para `Add` de `TextMessage`.
- Existe **limite de cartões por equipamento** (motivo da fragmentação por regional). Por isso o
  vínculo é feito por veículo/cartão, não em massa para os 4 mil motoristas.
- A mensagem fica **na fila** e é aplicada quando o equipamento conectar.
- O status "sem autorização" continua baseado em **grupos** (não lê a authlist do equipamento,
  que não é exposta pela API); o botão é a ação operacional de correção, não um espelho da authlist.

---

## Estrutura do projeto

```text
unassigned_keys_addin/
├── unassigned_keys.html              # Add-in completo (UI + CSS + JS) — entry point
├── unassigned_keys_addin_config.json # Configuração do add-in para o MyGeotab
├── unassigned_keys_icon.svg          # Ícone exibido no menu do MyGeotab
├── translations/
│   ├── pt-BR.json                    # Tradução do nome do menu (pt-BR)
│   └── en.json                       # Tradução do nome do menu (en)
├── docs/
│   ├── INSTALL.md                    # Instalação e atualização no MyGeotab
│   ├── ARCHITECTURE.md               # Arquitetura, fluxo de dados e decisões técnicas
│   └── CHANGELOG.md                  # Histórico de versões
├── README.md                         # Este arquivo
├── LICENSE                           # MIT
└── .nojekyll                         # Necessário para o GitHub Pages
```

### Por que um único HTML?

O MyGeotab carrega o add-in em um `iframe`. Manter HTML/CSS/JS juntos dispensa configuração de
`files` no config e reduz risco de cache/CORS. Para editar, procure os blocos no próprio arquivo:

- `<style>` → estilos (tema claro)
- `shell()` → monta o layout
- `load()` → busca e processa os dados
- `render*()` → KPIs, tabelas, resumo, mapa
- `renderAgg()` → agrupamento por placa (top 10, expansível)

---

## Instalação

Passo a passo detalhado em [`docs/INSTALL.md`](docs/INSTALL.md). Resumo:

1. Publique os arquivos no GitHub Pages (branch `main`, raiz).
2. No MyGeotab: **Administração → Sistema → Add-ins** (ou equivalente).
3. Cadastre o add-in usando a URL do **config**:
   `https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_addin_config.json`
4. O add-in aparece em **ActivityLink → Chaves Não Cadastradas**.

---

## Atualização e cache

O GitHub Pages usa CDN e pode servir a versão anterior por alguns minutos. Para publicar uma
atualização de forma confiável:

1. Edite `unassigned_keys.html`.
2. Suba a `version` em `unassigned_keys_addin_config.json` e o `?v=N` da URL (ex.: `?v=9`).
3. Faça commit/push.
4. No MyGeotab, remova e re-adicione o add-in (ou aguarde o CDN) e dê **Ctrl+F5**.

Esse `?v=N` é o que força o navegador a buscar o novo HTML.

---

## Permissões necessárias

Para visualizar, o usuário precisa de leitura para: `Audit`, `DriverChange`, `User`, `Device`,
`Group` e `DeviceStatusInfo`. Em contas com restrição de segurança, o acesso ao **Audit** pode
exigir perfil/limpeza de segurança específica. Sem `Audit`/`DriverChange`, as listas ficam vazias.

Para usar o botão **Vincular** (authlist), é necessário permissão de **escrita** para `Add` de
`TextMessage` e a authlist habilitada no equipamento.

---

## Solução de problemas

| Sintoma | Causa provável | Ação |
|---|---|---|
| Página antiga/preta | Cache do CDN ou add-in desatualizado | Suba `version`/`?v`, re-registre o add-in, Ctrl+F5 |
| "0 registros" | Nenhum evento no período ou sem permissão | Troque para 7/30/90 dias; confira permissões |
| Linhas vazias (—/—) | Eventos de segurança "key" | Já filtrados; se persistir, envie o texto do Audit |
| Cartão "—" nas válidas | Motorista sem cartão em `User.keys` | Cadastre a chave no motorista |
| E-mail "—" | Conta não expõe e-mail no objeto `User` | Usar o login/`name` ou ajustar `load()` |
| "Sem autorização" vazio | Todos os motoristas/veículos no mesmo grupo | Ajustar a regra em `isAuthorized()` |
| "Vincular" sem efeito | Authlist desabilitada no GO ou equipamento offline | Habilitar a authlist; o GO aplica quando conectar |
| "Vincular" retorna erro | Sem permissão de `Add` em `TextMessage` ou cartão sem `keyId` | Conferir permissões e `User.keys` |
| 404 em `translations/*.json` | Arquivos ausentes | Já incluídos na raiz do Pages |

Detalhes técnicos em [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## Desenvolvimento local

Como é um HTML estático, basta abrir `unassigned_keys.html` no navegador — porém **sem a API do
Geotab** ele mostra "Abra este add-in dentro do MyGeotab". Para testar com dados reais, use o
add-in dentro do MyGeotab (ou um ambiente de teste Geotab).

Não há build, dependências nem `npm`. As únicas dependências externas são o **Leaflet** (CDN unpkg)
e os tiles do **OpenStreetMap**.

---

## Segurança e privacidade

- Nenhum dado é enviado para servidores externos. Tudo é consultado na API do Geotab e renderizado no navegador.
- O mapa usa tiles públicos do OpenStreetMap (apenas coordenadas aproximadas dos veículos).
- Não há armazenamento de credenciais.

---

## Licença

[MIT](LICENSE) © Bruno Felisbino
