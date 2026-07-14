# qbx_truckrobbery — Manual

Assalto a carro-forte em movimento: o jogador contrata a missão com um NPC, para o caminhão, planta um explosivo na porta traseira e saqueia a carga.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Fluxo da missão](#fluxo-da-missão)
5. [Estados do caminhão](#estados-do-caminhão)
6. [Recompensas](#recompensas)
7. [Integrações](#integrações)
8. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
9. [Localização](#localização)
10. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | Framework base: `GetPlayer`, `Notify`, `GetDutyCountType`, `qbx.spawnVehicle`, statebags |
| `ox_lib` | Sim | Locale, callbacks, `progressBar`, `lib.points`, `lib.waitFor` |
| `ox_target` | Sim | Todas as interações (NPC, plantar bomba, saquear) são opções de target |
| `ox_inventory` | Sim | Consome o explosivo e entrega as recompensas (`RemoveItem`, `AddItem`, `CanCarryItem`, `CustomDrop`) |
| `qb-phone` | Não | Recebe o e-mail de briefing e o alerta policial. Se não existir, basta reescrever `emailNotification` e `alertPolice` no config |
| Recurso de polícia | Não | O alerta padrão dispara `police:client:policeAlert` nos policiais em serviço |

---

## Instalação

1. Copie a pasta `qbx_truckrobbery` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_truckrobbery
   ```
3. Garanta que os itens usados existem no `ox_inventory`:
   - `WEAPON_STICKYBOMB` — o explosivo consumido para abrir o caminhão (`bombItem`).
   - `black_money` e `security_card_01` — recompensas padrão em `config/server.lua`.
4. Ajuste `numRequiredPolice` conforme o número mínimo de policiais em serviço que o servidor exige.

---

## Configuração

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `bombItem` | string | Sim | Item do `ox_inventory` consumido para plantar a bomba. Padrão: `WEAPON_STICKYBOMB` |

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `dealerCoords` | vec4 | Sim | Onde o NPC contratante fica. O ped é criado a 400 m de distância do jogador |
| `dealerModel` | hash | Sim | Modelo do NPC. Padrão: `s_m_y_dealer_01` |
| `routeColor` | number | Sim | Cor da rota GPS até a área e até o caminhão |
| `emailNotification` | função | Sim | Chamada ao iniciar a missão. Por padrão dispara `qb-phone:server:sendNewMail`. Substitua para usar outro sistema de telefone |
| `guardAccuracy` | number | Sim | Precisão dos 4 guardas (0-100). Padrão: `50` |
| `lootDuration` | number (ms) | Sim | Duração da barra de progresso do saque. Padrão: `5000` |

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `numRequiredPolice` | number | Sim | Mínimo de policiais em serviço (tipo de job `leo`) para a missão ser aceita. Padrão: `2` |
| `activationCost` | number | Sim | Valor debitado do banco ao aceitar a missão. Padrão: `500` |
| `missionCooldown` | number (ms) | Sim | Tempo até uma nova missão poder ser iniciada no servidor. Padrão: `2700000` (45 min) |
| `rewards` | array de `Reward` | Sim | Tabela de loot. Cada entrada aceita `item`, `minAmount`, `maxAmount` e `probability` (0.0-1.0, independente por item) |
| `timeToDetonation` | number (s) | Sim | Segundos entre plantar a bomba e a explosão. Padrão: `30` |
| `driverWeapon` | hash | Sim | Arma do motorista. Padrão: `WEAPON_COMBATPISTOL` |
| `passengerWeapon` | hash | Sim | Arma do passageiro da frente. Padrão: `WEAPON_COMBATSHOTGUN` |
| `backPassengerWeapon` | hash | Sim | Arma dos dois guardas de trás. Padrão: `WEAPON_TACTICALRIFLE` |
| `truckModel` | hash | Sim | Modelo do carro-forte. Padrão: `Stockade` |
| `guardModel` | hash | Sim | Modelo dos guardas. Padrão: `s_m_m_security_01` |
| `truckSpawns` | array de vec4 | Sim | Locais possíveis de spawn do caminhão. Padrão: 8 pontos (Union Depository, Legion Square, Rockford Hills, etc.) |
| `alertPolice` | função | Sim | Chamada quando um jogador chega a menos de 5 m do caminhão. Notifica todos os policiais em serviço |

---

## Fluxo da missão

1. O jogador fala com o NPC (`dealerCoords`) e escolhe "pedir missão". A opção não aparece para jobs do tipo `leo`.
2. O servidor valida: nenhuma missão ativa, saldo bancário >= `activationCost`, e policiais em serviço >= `numRequiredPolice`. O custo é debitado do banco.
3. Um dos `truckSpawns` é sorteado. O jogador recebe o e-mail de briefing e um blip de área de 250 m de raio, com rota.
4. Ao entrar na área, o caminhão é spawnado com 4 guardas armados. O blip vira um marcador piscante preso ao veículo, e o caminhão dirige erraticamente pela cidade.
5. Quando qualquer jogador chega a menos de 5 m do caminhão (e a bomba ainda não foi plantada), a polícia é alertada.
6. Com o caminhão parado e fora d'água, o jogador planta a bomba (barra de 5 s). O item é consumido no servidor.
7. Após `timeToDetonation` segundos, as portas traseiras explodem e o caminhão fica saqueável.
8. O saque leva `lootDuration` ms e afastar-se mais de 6 m cancela a ação. As recompensas são sorteadas e entregues.
9. A missão só volta a ficar disponível depois de `missionCooldown`.

**Falha** — se ninguém estiver perto do caminhão (ele deixa de ter dono de rede) e a bomba ainda não tiver sido plantada, o caminhão é deletado e a missão falha.

Qualquer jogador pode interagir com o caminhão, não só quem pagou a missão. Isso é intencional: permite que gangues rivais disputem a carga.

---

## Estados do caminhão

O caminhão carrega o statebag `truckstate`, replicado para todos os clientes. Outros recursos podem ler `Entity(truck).state.truckstate`:

| Valor | Significado |
|---|---|
| `plantable` | Caminhão spawnado, aceita a bomba |
| `planted` | Bomba plantada, contagem regressiva rodando |
| `lootable` | Portas explodidas, carga acessível |
| `looted` | Carga já retirada |

Os guardas usam o statebag `qbx_truckrobbery:initGuard` para que o cliente dono da entidade aplique os atributos de combate.

---

## Recompensas

Definidas em `config.rewards`. Cada item é avaliado de forma independente:

```lua
rewards = {
    {
        item = 'black_money',
        minAmount = 250,
        maxAmount = 450,
    },
    {
        item = 'security_card_01',
        probability = 0.05
    }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `item` | string | Sim | Nome do item no `ox_inventory` |
| `minAmount` | number | Não | Quantidade mínima. Padrão: `1` |
| `maxAmount` | number | Não | Quantidade máxima. Padrão: `1` |
| `probability` | number | Não | Chance de o item aparecer (0.0 a 1.0). Padrão: `1.0` (sempre) |

Se o jogador não tiver espaço no inventário, os itens restantes viram um drop no chão ("Loot") na posição dele.

---

## Integrações

### ox_inventory

O explosivo é validado no cliente (`Search`) e consumido no servidor (`RemoveItem`). Ao plantar, o recurso dispara `ox_inventory:disarm` para guardar a arma em mãos antes da animação. As recompensas usam `AddItem`, `CanCarryItem` e `CustomDrop`.

### qb-phone

Duas funções do config assumem o `qb-phone`:

- `emailNotification` (client) — envia o e-mail de briefing via `qb-phone:server:sendNewMail`.
- `alertPolice` (server) — envia `qb-phone:client:addPoliceAlert` para os policiais em serviço.

Ambas são funções no arquivo de config; se o servidor usa outro telefone/dispatch, basta reescrever o corpo delas.

### Polícia (`leo`)

Jobs com `type = 'leo'` não conseguem contratar a missão, plantar a bomba nem saquear o caminhão. A contagem de policiais em serviço usa `exports.qbx_core:GetDutyCountType('leo')`, tanto para o mínimo exigido quanto para o alerta.

---

## Entrypoints para outros recursos

Não há exports. Os pontos de entrada são eventos e callbacks:

```lua
-- Servidor: inicia a missão para o jogador (mesma validação do NPC)
TriggerServerEvent('qbx_truckrobbery:server:startMission')

-- Cliente: avisado quando a missão começa (recebe as coords do spawn)
RegisterNetEvent('qbx_truckrobbery:client:missionStarted', function(vehicleSpawnCoords) end)

-- Cliente: avisado quando o cooldown termina e a missão é encerrada
RegisterNetEvent('qbx_truckrobbery:client:missionEnded', function() end)
```

Callbacks registrados no servidor:

- `qbx_truckrobbery:server:spawnVehicle` — spawna o caminhão e os guardas, devolve o `netId`.
- `qbx_truckrobbery:server:giveReward` — valida o estado `lootable` e entrega o loot.

---

## Localização

Strings via locale do `ox_lib`. Idiomas em `locales/`: `da`, `de`, `en`, `fr`, `pt`, `pt-br`.

```
setr ox:locale "pt-br"
```

---

## Estrutura de arquivos

```
qbx_truckrobbery/
├── client/
│   └── main.lua          — NPC contratante, blips, targets de plantar/saquear, handlers de statebag
├── server/
│   └── main.lua          — validação e cooldown da missão, spawn do caminhão e guardas, detonação, loot
├── shared/
│   └── types.lua         — enum TruckState (plantable, planted, lootable, looted)
├── config/
│   ├── client.lua        — NPC, blips, precisão dos guardas, duração do saque, e-mail de briefing
│   └── server.lua        — polícia mínima, custo, cooldown, loot, armas, modelos, spawns, alerta
├── locales/              — traduções (da, de, en, fr, pt, pt-br)
└── fxmanifest.lua
```
