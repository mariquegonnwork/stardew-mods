# Tooltip de status de água ao passar o mouse

Data: 2026-08-12

## Problema

Hoje o jogador precisa clicar/interagir com um animal (ou olhar a textura da
construção) para saber se um bebedouro/sistema de água precisa ser
reabastecido. Não há feedback ao simplesmente passar o mouse sobre a
instalação.

## Objetivo

Ao passar o mouse sobre uma instalação de água que pode ficar vazia, mostrar
uma tooltip indicando se ela precisa ou não de reabastecimento. Duas fontes
de água têm esse estado (cheio/vazio):

1. **Bebedouro/sistema de água fixo** (tiles de mapa dentro de um
   `AnimalHouse`, definidos no `TroughPlacement` da construção). O estado é
   por construção inteira (`BuildingWaterManager.IsWatered`).
2. **Water Bowl / outras fontes de água registradas** (objetos
   `StardewValley.Object` reconhecidos por `WaterSourceRegistry`). O estado
   é por objeto (`WaterSourceRegistry.IsFull`).

Fontes de água externas (rios, lagos etc.) ficam fora de escopo — não têm
conceito de vazio/cheio, então não há o que sinalizar ali.

A feature deve ser:
- **Traduzível via i18n**, seguindo o padrão já usado pelo mod
  (`ModHelper.Translation.Get(key)`), para que outros modders/tradutores da
  comunidade possam adicionar outros idiomas do mesmo jeito que já fazem
  para o resto do mod.
- **Configurável via GMCM**, com um toggle on/off, cujo próprio texto
  (nome e descrição da opção) também vem do i18n — diferente das opções
  existentes no `ModConfig.SetUpModConfigMenu`, que hoje têm texto fixo em
  inglês.

## Arquitetura

Novo arquivo `Core/UI/WaterSourceTooltip.cs`, seguindo o padrão de
separação já usado no mod entre lógica pura e o hook do jogo/Harmony (ex.:
`HarmonyPatches.cs` fino + `HarmonyPatchExecutors.cs` com a lógica):

```csharp
public static class WaterSourceTooltip
{
    // lógica pura, sem desenhar nada — testável isoladamente
    public static string? TryGetTooltipKey(GameLocation location, int tileX, int tileY, out bool isFull)

    // hook fino chamado pelo evento Display.RenderedHud
    public static void OnRenderedHud(object? sender, RenderedHudEventArgs e)
}
```

### `TryGetTooltipKey`

1. Se `location` é um `AnimalHouse` com `ParentBuilding` não nulo:
   - Pega o `buildingType` e busca o perfil via
     `ModEntry.PlacementRegistry.TryGetPlacement`.
   - Se `(tileX, tileY)` bate com algum `TroughTile` ou (quando
     `Config.UseWateringSystems`) `WateringSystemTile` do perfil, retorna a
     chave `Tooltip.Building.Watered` ou `Tooltip.Building.NotWatered`
     conforme `BuildingWaterManager.IsWatered(building)`.
2. Senão, se há um `StardewValley.Object` na tile
   (`location.Objects.TryGetValue`) e `WaterSourceRegistry.TryGet(obj, out
   source)` for verdadeiro, retorna `Tooltip.WaterSource.Full` ou
   `Tooltip.WaterSource.Empty` conforme `WaterSourceRegistry.IsFull(obj,
   source)`.
3. Senão, retorna `null` (nenhuma tooltip).

### `OnRenderedHud`

- Não faz nada se `!Config.ShowWaterSourceTooltip`, `!Context.IsWorldReady`,
  ou `Game1.activeClickableMenu != null` (evita desenhar por baixo/por cima
  de menus abertos).
- Calcula a tile sob o cursor (`Game1.currentCursorTile`), chama
  `TryGetTooltipKey` para `Game1.currentLocation`.
- Se retornou uma chave, resolve o texto via
  `ModEntry.ModHelper.Translation.Get(key)` e desenha com
  `IClickableMenu.drawHoverText(e.SpriteBatch, text, Game1.smallFont)`.

Registro em `ModEntry.Entry`:
```csharp
helper.Events.Display.RenderedHud += WaterSourceTooltip.OnRenderedHud;
```

## Config (GMCM)

Novo campo em `ModConfig`:
```csharp
public bool ShowWaterSourceTooltip { get; set; } = true;
```

Registrado na seção "Notifications" do `SetUpModConfigMenu`, usando i18n
(diferente do padrão hardcoded das opções vizinhas):
```csharp
api.AddBoolOption(manifest,
    () => config.ShowWaterSourceTooltip,
    val => config.ShowWaterSourceTooltip = val,
    () => ModEntry.ModHelper.Translation.Get("Config.ShowWaterSourceTooltip.name"),
    () => ModEntry.ModHelper.Translation.Get("Config.ShowWaterSourceTooltip.tooltip"));
```

## i18n

Novas chaves em `i18n/default.json` (inglês, fonte da verdade) e
`i18n/pt-BR.json` (traduzida por nós agora). Outros idiomas ficam sem essas
chaves — caem no fallback em inglês automaticamente (comportamento padrão
do SMAPI), e a comunidade pode completá-las depois como já faz para o
resto do mod.

```json
"Tooltip.Building.Watered": "This building's water source is full — animals can drink.",
"Tooltip.Building.NotWatered": "This building's water source is empty — refill it so animals can drink.",
"Tooltip.WaterSource.Full": "Full — animals can drink.",
"Tooltip.WaterSource.Empty": "Empty — needs refilling.",
"Config.ShowWaterSourceTooltip.name": "Show Water Source Tooltips",
"Config.ShowWaterSourceTooltip.tooltip": "Whether to show a tooltip when hovering over water troughs, watering systems, or water bowls, indicating whether they need refilling."
```

pt-BR:
```json
"Tooltip.Building.Watered": "A fonte de água desta construção está cheia — os animais podem beber.",
"Tooltip.Building.NotWatered": "A fonte de água desta construção está vazia — reabasteça para os animais poderem beber.",
"Tooltip.WaterSource.Full": "Cheio — os animais podem beber.",
"Tooltip.WaterSource.Empty": "Vazio — precisa ser reabastecido.",
"Config.ShowWaterSourceTooltip.name": "Mostrar Tooltip de Fontes de Água",
"Config.ShowWaterSourceTooltip.tooltip": "Se deve mostrar uma tooltip ao passar o mouse sobre bebedouros, sistemas de água ou Water Bowls, indicando se precisam ser reabastecidos."
```

## Fora de escopo (explicitamente)

- Fontes de água externas (rios/lagos/oceano) — sem estado vazio/cheio.
- Tradução das opções de config *já existentes* no GMCM (não pedida nesta
  rodada).
- Qualquer alteração visual além da tooltip (não mexe em texturas,
  overlays etc. — isso já existe).

## Testes / verificação

O repositório upstream não tem projeto de testes automatizados para este
mod. Verificação será manual:

1. `dotnet build` do projeto `AnimalsNeedWater.csproj`.
2. Backup da DLL atual em
   `Mods/AnimalsNeedWater/AnimalsNeedWater.dll` e substituição pela nova
   build.
3. No jogo: hover sobre bebedouro fixo cheio e vazio (dentro do celeiro e
   do galinheiro), hover sobre Water Bowl cheio e vazio, hover sobre
   sistema de água (Deluxe Barn/Coop), e teste do toggle
   "Show Water Source Tooltips" ligando/desligando no menu GMCM.
4. Confirmar que os textos aparecem em português (idioma padrão do jogo do
   usuário) e que nada quebra com o mouse sobre tiles sem fonte de água.

## PR para o mod original

Depois de implementado e verificado localmente, o usuário abre o PR
manualmente no `gzhynko/stardew-mods` — orientação (branch, título,
descrição) será dada à parte, fora desta spec.
