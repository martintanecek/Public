# 🔌 Chytré nabíjení Hyundai Ioniq Electric (28 kWh) — návod pro úplné začátečníky

Tento balíček do Home Assistantu udělá z obyčejné chytré zásuvky **chytrou nabíječku**:

- spočítá, kolik energie a času potřebuješ na dobití,
- **„Quick Charge"** = nabij hned na 80 %,
- **„Na 100 % k odjezdu"** = nabij hned na 80 % a zbytek do 100 % načasuj přesně na čas odjezdu (šetří baterii),
- **„Manuální režim"** = tupé zapnutí/vypnutí; HA nezasahuje, jen zobrazuje chytré výpočty (SOC, ETA na 100 %, spotřeba),
- sám vypne zásuvku po dosažení cíle (v režimech Quick/Plánovaně),
- při **ručním přerušení** uloží reálný dosažený SOC (nespadne zpět na start),
- ukáže cenu, měsíční/roční spotřebu a odhad dojezdu v km,
- hlásí stav do telefonu a (volitelně) nahlas přes reproduktor.

> ⏱️ Časová náročnost: **15–20 minut.** Nepotřebuješ umět programovat, jen kopírovat a přepisovat text.

---

## Co potřebuješ mít

1. **Home Assistant** (jakákoli běžná instalace – HAOS, Supervised…).
2. **Chytrou zásuvku**, do které zapojíš nabíjecí kabel auta. Musí jít:
   - **zapínat/vypínat** z HA (entita typu `switch.…`),
   - **měřit okamžitý výkon ve wattech** (entita typu `sensor.… power`).
   - Ověřené levné varianty: Shelly Plug S, Tapo P110, Athom/Tasmota plug apod.
3. **Doplněk „Studio Code Server"** nebo **„File editor"** (na úpravu souborů). Najdeš v *Nastavení → Doplňky → Obchod s doplňky*.

> ⚠️ **Pozor na proudové zatížení!** Ioniq z běžné zásuvky tahá ~9–10 A (cca 2,1 kW) trvale mnoho hodin. Použij kvalitní zásuvku a okruh. Levné plug-iny se mohou přehřívat.

---

## Krok 1 – Zjisti názvy svých entit

V HA jdi do **Nastavení → Zařízení a služby → Entity** a najdi svou zásuvku. Potřebuješ dvě:

| Co hledáš | Vypadá nějak takto | Poznamenej si |
|---|---|---|
| Spínač zásuvky | `switch.nabijecka` | _____________ |
| Výkon ve W | `sensor.nabijecka_power` | _____________ |

A pro notifikace svůj telefon (musíš mít appku **Home Assistant Companion** přihlášenou):

| Co hledáš | Příklad |
|---|---|
| Notifikace do mobilu | `notify.mobile_app_TVUJTELEFON` |

> 💡 Jak najít notifikaci: *Vývojářské nástroje → Akce (Actions)* → do vyhledávání napiš `notify.` a uvidíš seznam.

(Volitelně, pokud máš chytrý reproduktor pro hlasové hlášky: `media_player.…` a `tts.…`.)

---

## Krok 2 – Zapni „balíčky" (packages)

1. Otevři soubor **`configuration.yaml`** (přes Studio Code Server / File editor).
2. Zkontroluj, jestli někde na začátku máš tyto dva řádky. Pokud ne, **přidej je úplně nahoru**:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

3. Vytvoř složku **`packages`** ve stejném místě, kde je `configuration.yaml` (vedle něj).

---

## Krok 3 – Nahraj šablonu a přepiš názvy

1. Soubor **`ioniq_ev_charging_template.yaml`** zkopíruj do složky `packages/`.
2. Otevři ho a udělej **„Najít a nahradit"** (ve Studio Code Server: `Ctrl+H`). Nahraď tyto texty za své entity z Kroku 1:

   | Najít (token) | Nahradit za |
   |---|---|
   | `switch.ZASUVKA` | tvůj spínač, např. `switch.nabijecka` |
   | `sensor.ZASUVKA_vykon` | tvůj senzor výkonu, např. `sensor.nabijecka_power` |
   | `notify.MUJ_TELEFON` | tvá notifikace, např. `notify.mobile_app_pixel` |
   | `media_player.MUJ_REPRO` | (volitelné) tvůj reproduktor |
   | `tts.MUJ_TTS` | (volitelné) tvá TTS služba |

   > **Nemáš reproduktor/TTS?** Najdi v souboru každý blok začínající `- action: tts.speak` a smaž ho i s odsazenými řádky pod ním. Notifikace do telefonu fungují i bez toho.

3. **Konstanty auta měnit nemusíš** – jsou nastavené pro Ioniq 28 kWh. Pokud nabíjíš jiným výkonem (např. wallbox), uprav v souboru blok `sensor: "EV config"`:
   - `charger_kw` – nabíjecí výkon v kW (běžná zásuvka ≈ `2.14`),
   - ostatní (`battery_kwh`, dojezd, ztráty) nech.

4. **Ulož** a restartuj: *Nastavení → Systém → (vpravo nahoře) → Restartovat*.

> ✅ Po restartu zkontroluj *Nastavení → Systém → Protokoly*, jestli tam není červená chyba o tomto souboru. Když je čisto, jsi za vodou.

---

## Krok 4 – Nastav cenu elektřiny

V *Nastavení → Zařízení a služby → Pomocníci* najdi **„EV cena za kWh"** a nastav svou cenu (např. 6,50 Kč). Používá se pro výpočet nákladů.

---

## Krok 5 – Přidej ovládací panel (dashboard)

1. Otevři libovolný dashboard → vpravo nahoře **tužka (Upravit)** → **„+ Přidat zobrazení"**.
2. U nového zobrazení klikni na **tři tečky → Upravit v YAML** (Raw editor).
3. Vlož tento obsah:

```yaml
title: EV nabíjení
path: ev
icon: mdi:ev-station
type: sections
max_columns: 3
sections:
  - type: grid
    cards:
      - type: heading
        heading: Stav baterie
        icon: mdi:battery-charging-high
      - type: gauge
        entity: sensor.ev_estimated_soc
        name: Odhadovaný SOC
        min: 0
        max: 100
        needle: true
        segments:
          - { from: 0, color: "#db4437" }
          - { from: 20, color: "#0f9d58" }
          - { from: 80, color: "#f4b400" }
          - { from: 100, color: "#0f9d58" }
      # aktuální režim – zobrazí se jen když nabíjení běží
      - type: tile
        entity: input_select.ev_mode
        name: Aktuální režim
        icon: mdi:ev-station
        visibility:
          - condition: state
            entity: binary_sensor.ev_charging
            state: "on"
      - type: entities
        entities:
          - entity: sensor.ev_dojezd_km
            name: Dojezd
            icon: mdi:map-marker-distance
          - entity: input_number.ev_soc_current
            name: Aktuální SOC (zadej)
      # ⚡ Rychlé nabití na 80 %
      - type: tile
        entity: script.ev_quick_charge
        name: ⚡ Nabij hned na 80 %
        icon: mdi:flash
        hide_state: true
        vertical: true
        tap_action:
          action: perform-action
          perform_action: script.ev_quick_charge
      - type: entities
        entities:
          - entity: input_datetime.ev_ready_time
            name: Čas odjezdu
      # 🕐 Na 100 % k odjezdu (dvoufázově)
      - type: tile
        entity: script.ev_charge_100_by_departure
        name: 🕐 Na 100 % k odjezdu
        icon: mdi:calendar-clock
        hide_state: true
        vertical: true
        tap_action:
          action: perform-action
          perform_action: script.ev_charge_100_by_departure
      # 🔧 Manuální režim (tupé zapnutí)
      - type: tile
        entity: input_select.ev_mode
        name: 🔧 Manuální nabíjení
        icon: mdi:hand-back-right
        hide_state: true
        vertical: true
        tap_action:
          action: perform-action
          perform_action: input_select.select_option
          target:
            entity_id: input_select.ev_mode
          data:
            option: Manual
      - type: entities
        title: Detail
        entities:
          - entity: sensor.ZASUVKA_vykon
            name: Aktuální výkon
            icon: mdi:flash-outline
          - entity: sensor.ev_session_delivered_kwh
            name: Dodáno
          - entity: sensor.ev_kwh_needed
            name: Cíl (potřeba)
          - entity: sensor.ev_session_cost_czk
            name: Aktuální cena
          - entity: sensor.ev_remaining_text
            name: Zbývá
          - entity: sensor.ev_eta
            name: ETA (na cíl)
          - entity: sensor.ev_eta_100
            name: Hotovo na 100 %
          - entity: binary_sensor.ev_charging
            name: Nabíjení běží
  - type: grid
    cards:
      - type: heading
        heading: Plán a historie
        icon: mdi:calendar-clock
      - type: entities
        title: Vstupy
        entities:
          - entity: input_number.ev_soc_target
            name: Cíl
          - entity: input_datetime.ev_ready_time
            name: Hotovo v (odjezd)
          - entity: input_select.ev_mode
            name: Režim
          - entity: input_number.ev_price_czk
            name: Cena za kWh
      - type: entities
        title: Výpočet
        entities:
          - entity: sensor.ev_planned_start
            name: Začne nabíjet v
          - entity: sensor.ev_duration_text
            name: Bude trvat
      - type: entities
        title: Souhrn
        entities:
          - entity: sensor.ev_monthly_kwh_display
            name: Tento měsíc kWh
          - entity: sensor.ev_monthly_cost_czk
            name: Tento měsíc Kč
          - entity: sensor.ev_yearly_cost_czk
            name: Tento rok Kč
      - type: history-graph
        title: Výkon nabíjení (24 h)
        hours_to_show: 24
        entities:
          - entity: sensor.ZASUVKA_vykon
            name: Výkon W
```

> ❗ I tady nahraď `sensor.ZASUVKA_vykon` za svůj senzor výkonu (poslední graf).

4. Ulož a hotovo.

---

## Jak to používat

### A) Rychlé nabití na 80 %
1. Posuň slider **„Aktuální SOC"** na hodnotu, kterou ukazuje auto (např. 45 %).
2. Zmáčkni **„Quick Charge → 80 %"**.
3. Nabíjení se spustí a samo se vypne na 80 %.

### B) Plné nabití k odjezdu (šetrné k baterii)
1. Posuň slider **„Aktuální SOC"** na reálný stav.
2. Vedle nastav **čas odjezdu**.
3. Zmáčkni **„Na 100 % k odjezdu"**.
   - Auto se **hned** dobije na 80 % (kdybys musel/a vyrazit dřív).
   - Zbytek do 100 % se spustí tak, aby byl hotový **přesně v čas odjezdu**.

> 🔋 **Proč dvě fáze?** Baterie nerada stojí dlouho nabitá na 100 %. Tímto je na 100 % až těsně před jízdou.

### C) Manuální nabíjení (tupé zap/vyp)
1. (Volitelně) posuň slider **„Aktuální SOC"** na reálný stav, ať sedí výpočty.
2. Zmáčkni **„Manuální nabíjení"** → zásuvka se **tupě zapne** a HA do toho **nezasahuje**.
3. Během nabíjení vidíš živě SOC, dojezd, **ETA na 100 %**, dodané kWh i cenu.
4. **Vypneš ručně** (dlaždicí zásuvky / vypínačem) → automaticky se **uloží reálný dosažený SOC** a režim se vrátí na „Off".

> 🔧 Manuál je pro případ, kdy chceš mít plnou kontrolu (např. nabít na přesně tolik, kolik zrovna chceš, bez cíle) – chytré výpočty ti přitom pořád běží jako přehled.

---

## Důležité vědět

- **Vždy nastav reálný „Aktuální SOC"** před spuštěním. Z toho se počítá, kdy přestat. Když ho necháš špatně, nabití nebude přesné.
- Systém **neví**, kolik je v autě doopravdy (Ioniq to po kabelu neposílá). Pracuje s odhadem podle dodané energie. Po pár nabíjeních si můžeš doladit přesnost (viz níže).
- Po dosažení cíle se zásuvka **sama vypne** a režim se vrátí na „Off" (v režimech Quick / Plánovaně).
- **Přerušíš-li nabíjení ručně** (třeba když přijde bouřka), uloží se skutečný dosažený SOC – hodnota nespadne zpět na startovní. Platí pro všechny režimy.

---

## Doladění přesnosti (po prvním nabíjení)

Nabij např. z 50 na 80 a podívej se, na kolik % se auto reálně nabilo:

- Nabilo se na **víc** než 80 (přebití) → v `sensor: EV config` lehce **zvyš** `eff_below_80` (např. 0.92 → 0.94).
- Nabilo se na **míň** než 80 (nedobití) → lehce **sniž** `eff_below_80` (např. 0.92 → 0.90).

Obdobně `eff_above_80` pro pásmo 80–100 %. Po úpravě restartuj nebo *Vývojářské nástroje → YAML → Znovu načíst šablony*.

---

## Když něco nefunguje

| Problém | Řešení |
|---|---|
| Po restartu je v Protokolech červená chyba o souboru | Nejčastěji špatné odsazení po mazání TTS řádků, nebo nepřepsaný token. Zkontroluj. |
| `sensor.ev_estimated_soc` je „neznámé" | Šablony se ještě nenačetly – restartuj, nebo *Vývojářské nástroje → YAML → Znovu načíst šablony*. |
| Tlačítko nic nedělá | Ověř, že existují `script.ev_quick_charge` a `script.ev_charge_100_by_departure` (*Nastavení → Automatizace a scény → Skripty*). |
| Nabíjí přes cíl | Měl/a jsi „Aktuální SOC" stejný nebo vyšší než cíl → nebylo co nabíjet. Nastav reálnou (nižší) hodnotu. |
| Graf výkonu / aktuální výkon je prázdný | V dashboardu jsi nepřepsal/a `sensor.ZASUVKA_vykon` za svůj senzor. |
| Chybí režim „Manual" ve výběru | Po úpravě jsi nereloadoval/a. Udělej *Vývojářské nástroje → YAML → Input select*, nebo restartuj. |
| „Aktuální režim" na dashboardu se neukazuje | To je záměr – zobrazí se **jen když nabíjení běží**. |

---

## Seznam entit, které balíček vytvoří

**Ovládání:** `input_number.ev_soc_current`, `input_number.ev_soc_target`, `input_number.ev_price_czk`, `input_datetime.ev_ready_time`, `input_select.ev_mode`, `input_button.ev_reset_session`, `input_boolean.ev_finish_to_100`

**Skripty:** `script.ev_quick_charge`, `script.ev_charge_100_by_departure`

**Senzory:** `sensor.ev_estimated_soc`, `sensor.ev_dojezd_km`, `sensor.ev_kwh_needed`, `sensor.ev_duration_text`, `sensor.ev_planned_start`, `sensor.ev_session_delivered_kwh`, `sensor.ev_session_cost_czk`, `sensor.ev_remaining_text`, `sensor.ev_eta`, `sensor.ev_eta_100`, `sensor.ev_monthly_kwh_display`, `sensor.ev_monthly_cost_czk`, `sensor.ev_yearly_cost_czk`, `binary_sensor.ev_charging`

**Režimy** (`input_select.ev_mode`): Off, Charge now, Scheduled, **Manual**.

**Automatizace:** start hned, start naplánovaně, vypnutí na cíli (Manual ignoruje), hlášky 80/100 %, failsafe, **manuální start**, **uložení SOC při přerušení**.

---

*Vytvořeno pro Hyundai Ioniq Electric 28 kWh. Volně k použití a úpravám.*
