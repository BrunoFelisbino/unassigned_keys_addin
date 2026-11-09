# Arquitetura

## Visão geral

```text
MyGeotab
  └─ iframe
       └─ unassigned_keys.html
            ├─ shell()        → monta o layout (tema claro)
            ├─ load(api)      → consulta a API do Geotab
            │    ├─ Group
            │    ├─ Device
            │    ├─ User
            │    ├─ Audit
            │    ├─ DriverChange
            │    └─ DeviceStatusInfo
            └─ render*()      → KPIs, tabelas, resumo, mapa
```

O add-in usa a **API nova de add-ins** do Geotab:

```js
window.geotab.addin.unassigned_keys = function () {
  return {
    initialize: function (api, state, callback) { init(api); if (callback) callback(); },
    focus: function () { /* reajusta o mapa */ },
    blur: function () {}
  };
};
```

Todas as chamadas passam por um wrapper:

```js
function call(typeName, search) {
  return new Promise(function (ok, err) {
    apiRef.call('Get', { typeName: typeName, search: search || {} }, ok, err);
  });
}
```

## Entrada no menu

Definida em `unassigned_keys_addin_config.json`:

```json
{
  "name": "unassigned_keys",
  "version": "1.0.6",
  "items": [{
    "url": "https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys.html?v=7",
    "path": "ActivityLink/",
    "menuName": { "en": "Unassigned Keys", "pt-BR": "Chaves Não Cadastradas" },
    "svgIcon": "https://brunofelisbino.github.io/unassigned_keys_addin/unassigned_keys_icon.svg"
  }],
  "files": {}
}
```

- `path: "ActivityLink/"` coloca o item no menu ActivityLink (mesmo padrão do add-in Fleet Health Score).
- `files: {}` não é necessário, pois o HTML é autocontido.

## Fluxo de dados

### 1. Carregamento base

`Group`, `Device` e `User` são carregados primeiro para montar os mapas de referência:

- `devices[id] = { id, name, plate }`
- `deviceGroupIds[id] = [groupId, ...]`
- `users[id] = { id, name, email, groupIds, keys }`

A placa é derivada de `licensePlate`; se vazio, usa o prefixo de `name` antes de `|`.

### 2. Cartões não cadastrados — `Audit`

Busca `Audit` de `fromDate` a `toDate` e filtra eventos relacionados a chave/cartão:

```js
var txt = ((a.name || '') + ' ' + (a.comment || '')).replace(/\s+/g, ' ').trim();
```

- **Descarta** eventos de segurança (`create/restore/saved/api/backup/certificate/... key`).
- **Mantém** eventos com `key|chave|cart|inserir` ou `unassign|não atrib|nao atrib|não cadastr|...`.
- Extrai o **serial** (6–20 dígitos ou hex de 8–20) e a **placa** (padrão Mercosul/antigo `ABC1D23`).
- Descarta registros sem serial **e** sem placa (ruído).

### 3. Leituras válidas — `DriverChange`

Busca `DriverChange` no mesmo período. Um evento é considerado leitura válida quando existe
`driver.id` e ele não é `UnknownDriver`.

Cada leitura recebe:

- `plate` / `deviceId` do veículo;
- `driverName` / `driverEmail` do `User`;
- `serial` a partir de `User.keys` (primeiro cartão + `+N`, tooltip com todos);
- `authorized` calculado por `isAuthorized()`.

> `DriverChange` não carrega o serial lido. O serial exibido é o cadastro do motorista.

### 4. Autorização por grupo

```js
function isAuthorized(driverId, deviceId) {
  var ug = authGroups(users[driverId].groupIds);
  var dg = authGroups(deviceGroupIds[deviceId]);
  if (!ug.length || !dg.length) return true; // sem dados suficientes
  return ug.some(function (id) { return dg.indexOf(id) !== -1; });
}
```

`authGroups()` remove o grupo raiz `GroupCompanyId`. Assim, filiais/equipes distintas são
detectadas como "sem autorização"; se todos estiverem no mesmo grupo raiz, nada é marcado.

### 5. Vínculo na driver list (authlist)

O botão **Vincular** envia um `TextMessage` com `DriverAuthListContent` para o equipamento,
adicionando o cartão à *Authorized Driver List* (authlist) do IOX-NFCREADERA:

```js
api.call('Add', {
  typeName: 'TextMessage',
  entity: {
    device: { id: deviceId },
    isDirectionToVehicle: true,
    messageContent: {
      contentType: 'DriverAuthList',
      driverKey: { keyId: key.keyId, driverKeyType: key.driverKeyType },
      addToAuthList: true,
      clearAuthList: false
    }
  }
}, success, error);
```

- `driverKey` vem de `User.keys` (guardado em `users[id].rawKeys`).
- **Vincular cartões** (por placa) envia todos os motoristas distintos daquela placa.
- **Vincular** (por linha) envia só aquele motorista.
- A authlist **não é legível** pela API; o add-in não consegue exibir o estado atual, apenas enviar
  comandos de adicionar/remover. Por isso o status continua baseado em grupos.
- Requer authlist habilitada no GO, permissão de `Add` em `TextMessage` e respeita o limite de
  cartões por equipamento.

### 6. Persistência — `AddInData`

Um único registro por banco, com `addInId = "unassigned_keys"`, guarda um JSON versionado:

```json
{
  "schemaVersion": 1,
  "settings": { "period": "today" },
  "links": {
    "<deviceId>::<driverId>": {
      "deviceId": "...", "driverId": "...", "keyIds": ["..."],
      "plate": "ABC1D23", "driverName": "...",
      "linkedAt": "2026-11-09T12:00:00.000Z", "linkedBy": "usuario"
    }
  }
}
```

- Lido uma vez em `loadAddinData()` na inicialização; `details` é parseado de forma tolerante
  (string ou objeto) e `schemaVersion` permite migração futura.
- Gravado (`Add` na primeira vez, depois `Set`) somente ao trocar o período ou ao concluir um
  vínculo com sucesso.
- `groups: [{ id: "GroupCompanyId" }]` deixa o dado visível a quem tem acesso à empresa.
- **Degradação graciosa**: qualquer falha em `AddInData` é capturada e logada; o add-in opera em
  memória sem quebrar.
- Como a authlist não é legível, esse ledger é a memória operacional dos vínculos (quem/quando),
  usado para exibir "✓ vinculado" e trocar o botão para "Reenviar".

### 7. Mapa — `DeviceStatusInfo`

Para os veículos com tentativas não cadastradas, busca a última posição conhecida e plota
`circleMarker` no Leaflet, ajustando o zoom ao conjunto de pontos.

## Filtros e agregação

`apply()` filtra `allRows` (não cadastrados) e `validRows` (válidos) por grupo, veículo e texto.
`validRows` filtrado vira `filteredValid`.

`renderAgg()` (seções válidas e sem autorização):

- agrupa por placa;
- ordena por quantidade de leituras (decrescente);
- mostra **top 10** em `<details>` recolhidos;
- ao expandir, lista Data/Hora, Cartão, Motorista, E-mail e Status;
- exibe aviso quando há mais de 10 placas.

## Performance

- Período padrão **Hoje** reduz o volume de `Audit`/`DriverChange`.
- Detalhes limitados a 300 linhas por placa na expansão.
- `DriverChange` de períodos longos (90 dias) pode ser volumoso em frotas grandes.

## Decisões técnicas

| Decisão | Motivo |
|---|---|
| HTML autocontido | Compatibilidade com iframe do MyGeotab; sem build |
| API nova (`initialize/focus/blur`) | Padrão atual do Geotab |
| `?v=N` no config | Cache-busting do CDN do GitHub Pages |
| Serial de válidas via `User.keys` | `DriverChange` não expõe serial |
| Autorização por grupos | Modelo nativo de permissão do Geotab |

## Limitações conhecidas

- O serial exato de uma leitura válida não é identificável quando o motorista possui vários cartões.
- Em contas com todos os usuários no grupo raiz, a checagem de autorização não distingue filiais.
- O e-mail depende de o objeto `User` expor o campo; caso contrário, aparece `—`.
- Acesso ao `Audit` pode exigir permissões de segurança específicas.
