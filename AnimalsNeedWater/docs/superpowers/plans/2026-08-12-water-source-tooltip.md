# Tooltip de status de água ao passar o mouse — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ao passar o mouse sobre um bebedouro/sistema de água fixo (dentro de um `AnimalHouse`) ou sobre um objeto de fonte de água (Water Bowl etc.), mostrar uma tooltip dizendo se precisa ser reabastecido — traduzível via i18n e configurável (on/off) via GMCM.

**Architecture:** Lógica pura em `WaterSourceTooltip.TryGetTooltipKey(...)` decide qual chave i18n mostrar (ou nenhuma); um handler fino em `WaterSourceTooltip.OnRenderedHud` (hookado em `Display.RenderedHud`) resolve a tradução e desenha com `IClickableMenu.drawHoverText`. Segue o padrão do mod de separar lógica pura de hooks de evento (ver `HarmonyPatches.cs` / `HarmonyPatchExecutors.cs`).

**Tech Stack:** C# / .NET 6, SMAPI (via `Pathoschild.Stardew.ModBuildConfig`), Harmony (já usado no mod, não é necessário para esta feature), Generic Mod Config Menu (GMCM) via `IGenericModConfigMenuApi`.

## Global Constraints

- Todo texto novo visível ao jogador (tooltip + label/descrição da opção no GMCM) DEVE vir de `ModEntry.ModHelper.Translation.Get(key)` — nunca string literal hardcoded. (spec: "Traduzível via i18n")
- Escopo da tooltip: só bebedouro/sistema de água fixo (por construção) e objetos `WaterSource` (ex. Water Bowl, por objeto). Fontes de água externas (rios/lagos) ficam fora de escopo.
- Chaves i18n novas vão em `i18n/default.json` (inglês) e `i18n/pt-BR.json` (português). Outros idiomas não são tocados — ficam no fallback em inglês.
- Repositório não tem projeto de testes automatizado para C#; verificação é manual, no jogo, conforme documentado na spec.
- Projeto alvo: `C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater\AnimalsNeedWater\AnimalsNeedWater.csproj`. Mod instalado (deploy de teste) em `C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater`.

---

## Task 1: Config field + opção no GMCM (i18n-driven)

**Files:**
- Modify: `AnimalsNeedWater/Core/Models/ModConfig.cs`

**Interfaces:**
- Produces: `ModConfig.ShowWaterSourceTooltip` (bool, default `true`) — consumido pela Task 3 (`WaterSourceTooltip.OnRenderedHud`).

- [ ] **Step 1: Adicionar o campo de config**

Em `AnimalsNeedWater/Core/Models/ModConfig.cs`, logo depois da propriedade `ThirstyEmoteStartTime` (linha 113, antes de `SprinklersFillWaterBowls`), adicionar:

```csharp
    /// <summary>
    /// Whether to show a tooltip when hovering over a water trough, watering system, or water source object,
    /// indicating whether it needs refilling.
    /// </summary>
    public bool ShowWaterSourceTooltip { get; set; } = true;
```

- [ ] **Step 2: Registrar a opção no GMCM usando i18n**

Na mesma classe, dentro de `SetUpModConfigMenu`, logo depois da linha:

```csharp
        api.AddNumberOption(manifest,
            () => config.ThirstyEmoteStartTime,
            val => config.ThirstyEmoteStartTime = (int)val,
            () => "Thirsty Emote Start Time",
            () => "Animals may start showing a thirsty emote after this time. Only applies if Show Thirsty Emotes is on.",
            min: 600f, max: 2600f, interval: 100f,
            formatValue: val => Game1.getTimeOfDayString((int)val));
```

adicionar (ainda dentro da seção "Notifications", antes de `api.AddSectionTitle(manifest, () => "Water Sources");`):

```csharp
        api.AddBoolOption(manifest,
            () => config.ShowWaterSourceTooltip,
            val => config.ShowWaterSourceTooltip = val,
            () => ModEntry.ModHelper.Translation.Get("Config.ShowWaterSourceTooltip.name"),
            () => ModEntry.ModHelper.Translation.Get("Config.ShowWaterSourceTooltip.tooltip"));
```

- [ ] **Step 3: Verificar que compila**

Run: `dotnet build "C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater\AnimalsNeedWater\AnimalsNeedWater.csproj"`
Expected: `Build succeeded.` (vai reclamar que as chaves i18n usadas ainda não existem? Não — `Translation.Get` resolve em runtime, não compile-time, então isso compila mesmo antes da Task 2.)

- [ ] **Step 4: Commit**

```bash
cd "C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater"
git add AnimalsNeedWater/Core/Models/ModConfig.cs
git commit -m "feat(animals-need-water): add ShowWaterSourceTooltip config option"
```

---

## Task 2: Chaves i18n (inglês + português)

**Files:**
- Modify: `AnimalsNeedWater/i18n/default.json`
- Modify: `AnimalsNeedWater/i18n/pt-BR.json`

**Interfaces:**
- Produces: chaves `Tooltip.Building.Watered`, `Tooltip.Building.NotWatered`, `Tooltip.WaterSource.Full`, `Tooltip.WaterSource.Empty`, `Config.ShowWaterSourceTooltip.name`, `Config.ShowWaterSourceTooltip.tooltip` — consumidas pela Task 1 (Step 2) e Task 3.

Nota: `pt-BR.json` hoje só tem as chaves `AnimalsLeftWithoutWaterYesterday.*` — está faltando a tradução de `Items.Water_Bowl.*` e `Animal.Status.*` que já existem no `default.json` (por isso essas mensagens aparecem em inglês no jogo mesmo com o idioma em português). Esta task também completa essa lacuna, já que vamos mexer no arquivo mesmo.

- [ ] **Step 1: Adicionar as chaves novas em `default.json`**

Em `AnimalsNeedWater/i18n/default.json`, substituir a última linha (`"Animal.Status.WasLeftThirsty": "Was left without water yesterday."` seguida de `}`) por:

```json
  "Animal.Status.WasLeftThirsty": "Was left without water yesterday.",

  "Tooltip.Building.Watered": "This building's water source is full — animals can drink.",
  "Tooltip.Building.NotWatered": "This building's water source is empty — refill it so animals can drink.",
  "Tooltip.WaterSource.Full": "Full — animals can drink.",
  "Tooltip.WaterSource.Empty": "Empty — needs refilling.",

  "Config.ShowWaterSourceTooltip.name": "Show Water Source Tooltips",
  "Config.ShowWaterSourceTooltip.tooltip": "Whether to show a tooltip when hovering over water troughs, watering systems, or water bowls, indicating whether they need refilling."
}
```

- [ ] **Step 2: Completar e adicionar as chaves em `pt-BR.json`**

Substituir todo o conteúdo de `AnimalsNeedWater/i18n/pt-BR.json` por:

```json
{
    "AnimalsLeftWithoutWaterYesterday.globalMessage.oneAnimal_Male": "{{firstAnimalName}} ficou sem água ontem.",
    "AnimalsLeftWithoutWaterYesterday.globalMessage.oneAnimal_Female": "{{firstAnimalName}} ficou sem água ontem.",
    "AnimalsLeftWithoutWaterYesterday.globalMessage.oneAnimal_UnknownGender": "{{firstAnimalName}} ficou sem água ontem.",
    "AnimalsLeftWithoutWaterYesterday.globalMessage.twoAnimals": "{{firstAnimalName}} e {{secondAnimalName}} ficaram sem água ontem.",
    "AnimalsLeftWithoutWaterYesterday.globalMessage.threeAnimals": "{{firstAnimalName}}, {{secondAnimalName}} e {{thirdAnimalName}} ficaram sem água ontem.",
    "AnimalsLeftWithoutWaterYesterday.globalMessage.multipleAnimals": "{{firstAnimalName}}, {{secondAnimalName}}, {{thirdAnimalName}} e mais {{totalAmountExcludingFirstThree}} ficaram sem água ontem.",

    "Items.Water_Bowl.name": "Bebedouro",
    "Items.Water_Bowl.description": "Pode ser colocado dentro de instalações de animais para que eles possam beber água mesmo que a construção não venha com um bebedouro.",

    "Animal.Status.Thirsty.today": "Ainda não bebeu água hoje.",
    "Animal.Status.Thirsty.multipleDays": "Está sem água há {{days}} dias.",
    "Animal.Status.NoWaterButNotThirstyYet": "Sem água disponível, mas ainda não está com sede.",
    "Animal.Status.WasLeftThirsty": "Ficou sem água ontem.",

    "Tooltip.Building.Watered": "A fonte de água desta construção está cheia — os animais podem beber.",
    "Tooltip.Building.NotWatered": "A fonte de água desta construção está vazia — reabasteça para os animais poderem beber.",
    "Tooltip.WaterSource.Full": "Cheio — os animais podem beber.",
    "Tooltip.WaterSource.Empty": "Vazio — precisa ser reabastecido.",

    "Config.ShowWaterSourceTooltip.name": "Mostrar Tooltip de Fontes de Água",
    "Config.ShowWaterSourceTooltip.tooltip": "Se deve mostrar uma tooltip ao passar o mouse sobre bebedouros, sistemas de água ou Water Bowls, indicando se precisam ser reabastecidos."
}
```

- [ ] **Step 3: Validar que os JSONs são válidos**

Run:
```bash
cd "C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater\AnimalsNeedWater"
node -e "JSON.parse(require('fs').readFileSync('i18n/default.json','utf8')); JSON.parse(require('fs').readFileSync('i18n/pt-BR.json','utf8')); console.log('ok')"
```
Expected: `ok` (se `node` não estiver disponível, usar `python -c "import json; json.load(open('i18n/default.json', encoding='utf-8')); json.load(open('i18n/pt-BR.json', encoding='utf-8')); print('ok')"`)

- [ ] **Step 4: Commit**

```bash
git add AnimalsNeedWater/i18n/default.json AnimalsNeedWater/i18n/pt-BR.json
git commit -m "feat(animals-need-water): add water source tooltip translations, complete pt-BR"
```

---

## Task 3: Lógica da tooltip (`WaterSourceTooltip`)

**Files:**
- Create: `AnimalsNeedWater/Core/UI/WaterSourceTooltip.cs`

**Interfaces:**
- Consumes:
  - `ModEntry.PlacementRegistry.TryGetPlacement(string buildingType, out TroughPlacement? placement)` (existing)
  - `TroughPlacement.TroughTiles` (`List<TroughTile>`), `TroughPlacement.WateringSystemTiles` (`List<WateringSystemTile>`) (existing)
  - `TroughTile.TileX`/`TileY` (int), `WateringSystemTile.TileX`/`TileY` (int) (existing)
  - `ModEntry.BuildingWaterManager.IsWatered(Building building)` → `bool` (existing)
  - `ModEntry.WaterSources.TryGet(StardewValley.Object obj, out WaterSource? source)` → `bool` (existing)
  - `ModEntry.WaterSources.IsFull(StardewValley.Object obj, WaterSource source)` → `bool` (existing)
  - `ModEntry.Config.ShowWaterSourceTooltip` (bool, from Task 1)
  - `ModEntry.Config.UseWateringSystems` (existing bool)
  - `ModEntry.ModHelper.Translation.Get(string key)` (existing)
- Produces:
  - `WaterSourceTooltip.TryGetTooltipKey(GameLocation location, int tileX, int tileY)` → `string?` — usado pela Task 4 indiretamente (é chamado de dentro de `OnRenderedHud`, definido nesta mesma task).
  - `WaterSourceTooltip.OnRenderedHud(object? sender, RenderedHudEventArgs e)` — consumido pela Task 4 (registro em `ModEntry.cs`).

- [ ] **Step 1: Criar o arquivo com a lógica pura e o hook de renderização**

Criar `AnimalsNeedWater/Core/UI/WaterSourceTooltip.cs`:

```csharp
using System.Collections.Generic;
using AnimalsNeedWater.Core.Content.Models;
using Microsoft.Xna.Framework;
using StardewModdingAPI.Events;
using StardewValley;
using StardewValley.Buildings;
using StardewValley.Menus;
using Object = StardewValley.Object;

namespace AnimalsNeedWater.Core.UI;

public static class WaterSourceTooltip
{
    /// <summary>
    /// Decides which i18n key (if any) should be shown as a tooltip for the given world tile.
    /// Pure logic, no drawing — safe to call outside of a render context.
    /// </summary>
    public static string? TryGetTooltipKey(GameLocation location, int tileX, int tileY)
    {
        if (location is AnimalHouse && location.ParentBuilding is Building building)
        {
            var buildingType = building.buildingType.Value;
            if (ModEntry.PlacementRegistry.TryGetPlacement(buildingType, out var placement))
            {
                bool matchesTrough = MatchesAnyTroughTile(placement.TroughTiles, tileX, tileY);
                bool matchesWateringSystem = ModEntry.Config.UseWateringSystems
                    && MatchesAnyWateringSystemTile(placement.WateringSystemTiles, tileX, tileY);

                if (matchesTrough || matchesWateringSystem)
                {
                    bool watered = ModEntry.BuildingWaterManager.IsWatered(building);
                    return watered ? "Tooltip.Building.Watered" : "Tooltip.Building.NotWatered";
                }
            }
        }

        if (location.Objects.TryGetValue(new Vector2(tileX, tileY), out Object? obj)
            && ModEntry.WaterSources.TryGet(obj, out var source))
        {
            bool full = ModEntry.WaterSources.IsFull(obj, source);
            return full ? "Tooltip.WaterSource.Full" : "Tooltip.WaterSource.Empty";
        }

        return null;
    }

    private static bool MatchesAnyTroughTile(List<TroughTile> tiles, int x, int y)
    {
        foreach (var t in tiles)
        {
            if (t.TileX == x && t.TileY == y) return true;
        }
        return false;
    }

    private static bool MatchesAnyWateringSystemTile(List<WateringSystemTile> tiles, int x, int y)
    {
        foreach (var t in tiles)
        {
            if (t.TileX == x && t.TileY == y) return true;
        }
        return false;
    }

    /// <summary> Draws the tooltip for the tile under the cursor, if any. Hooked to Display.RenderedHud. </summary>
    public static void OnRenderedHud(object? sender, RenderedHudEventArgs e)
    {
        if (!ModEntry.Config.ShowWaterSourceTooltip) return;
        if (!Context.IsWorldReady) return;
        if (Game1.activeClickableMenu != null) return;
        if (Game1.currentLocation == null) return;

        var cursorTile = Game1.currentCursorTile;
        var key = TryGetTooltipKey(Game1.currentLocation, (int)cursorTile.X, (int)cursorTile.Y);
        if (key == null) return;

        var text = ModEntry.ModHelper.Translation.Get(key);
        IClickableMenu.drawHoverText(e.SpriteBatch, text, Game1.smallFont);
    }
}
```

- [ ] **Step 2: Verificar que compila**

Run: `dotnet build "C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater\AnimalsNeedWater\AnimalsNeedWater.csproj"`
Expected: `Build succeeded.`

- [ ] **Step 3: Commit**

```bash
git add AnimalsNeedWater/Core/UI/WaterSourceTooltip.cs
git commit -m "feat(animals-need-water): add water source hover tooltip logic"
```

---

## Task 4: Registrar o evento em `ModEntry`

**Files:**
- Modify: `AnimalsNeedWater/ModEntry.cs`

**Interfaces:**
- Consumes: `AnimalsNeedWater.Core.UI.WaterSourceTooltip.OnRenderedHud` (Task 3)

- [ ] **Step 1: Adicionar o using**

Em `AnimalsNeedWater/ModEntry.cs`, logo abaixo de `using AnimalsNeedWater.Core.Patching;` (linha 9), adicionar:

```csharp
using AnimalsNeedWater.Core.UI;
```

- [ ] **Step 2: Registrar o handler**

Logo depois de:

```csharp
        helper.Events.Display.RenderedWorld += PlacementDiagnostics.OnRenderedWorld;
        helper.Events.Display.MenuChanged += OutsideDrinkingSpots.OnMenuChanged;
```

adicionar:

```csharp
        helper.Events.Display.RenderedHud += WaterSourceTooltip.OnRenderedHud;
```

- [ ] **Step 3: Verificar que compila**

Run: `dotnet build "C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater\AnimalsNeedWater\AnimalsNeedWater.csproj"`
Expected: `Build succeeded.`

- [ ] **Step 4: Commit**

```bash
git add AnimalsNeedWater/ModEntry.cs
git commit -m "feat(animals-need-water): wire up water source tooltip rendering"
```

---

## Task 5: Deploy local e verificação manual no jogo

**Files:**
- N/A (build + deploy; nenhum arquivo de código novo)

**Interfaces:**
- Consumes: build output do `AnimalsNeedWater.csproj` (Tasks 1–4)

- [ ] **Step 1: Backup do mod instalado atualmente**

O `Pathoschild.Stardew.ModBuildConfig` copia automaticamente o build para a pasta `Mods/AnimalsNeedWater` do jogo (detecta o caminho do jogo sozinho). Antes de deixar isso sobrescrever a instalação atual, faça backup:

```bash
cp -r "C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater" "C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater.bak"
```

- [ ] **Step 2: Build com o jogo fechado**

Feche o Stardew Valley (o SMAPI trava a DLL enquanto o jogo roda). Depois:

```bash
dotnet build "C:\Users\Mario Henrique\www-pessoal\AnimalsNeedWater\AnimalsNeedWater\AnimalsNeedWater.csproj" -c Release
```

Expected: `Build succeeded.`, e a pasta `Mods/AnimalsNeedWater` deve ter uma `AnimalsNeedWater.dll` com data/hora recente (confirma que o deploy automático rodou).

- [ ] **Step 3: Checklist manual no jogo**

Abra o jogo e verifique, em português:

1. Entre no celeiro/galinheiro com um bebedouro fixo vazio → passe o mouse sobre o tile do bebedouro → deve aparecer "A fonte de água desta construção está vazia — reabasteça para os animais poderem beber."
2. Reabasteça (regador ou pela mecânica normal do mod) → passe o mouse de novo → deve mudar para "...está cheia — os animais podem beber."
3. Coloque um Water Bowl, esvazie e passe o mouse → "Vazio — precisa ser reabastecido." Reabasteça → "Cheio — os animais podem beber."
4. Se tiver uma construção com sistema de água (Deluxe Barn/Coop), repita 1–2 no tile do sistema de água.
5. Abra "Configurar Mods" → GMCM → seção "Notifications" → confirme que existe a opção "Mostrar Tooltip de Fontes de Água" com a descrição em português, e que desligá-la faz a tooltip parar de aparecer (religue antes de continuar).
6. Passe o mouse sobre um tile qualquer sem fonte de água (grama, chão) → nenhuma tooltip deve aparecer.
7. Abra qualquer menu (inventário, mapa) e confirme que a tooltip não aparece por baixo/por cima do menu.

- [ ] **Step 4: Remover o backup se tudo estiver certo**

Se os passos 3 passarem, o backup não é mais necessário (o CLAUDE.md do usuário exige confirmação antes de apagar arquivos que não foram criados nesta sessão — mas `AnimalsNeedWater.bak` FOI criado nesta sessão, no Step 1 desta mesma task, então pode ser removido livremente aqui):

```bash
rm -rf "C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater.bak"
```

Se algo falhar, restaure o backup em vez de remover:

```bash
rm -rf "C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater"
mv "C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater.bak" "C:\Program Files (x86)\Steam\steamapps\common\Stardew Valley\Mods\AnimalsNeedWater"
```

---

## Depois do plano

Depois que a Task 5 passar, o usuário abre o PR manualmente no `gzhynko/stardew-mods` (branch, título e descrição do PR serão orientados à parte, fora deste plano).
