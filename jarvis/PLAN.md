# Jarvis – persoonlijke AI-assistent

Stappenplan en aandachtspunten voor een spraakgestuurde assistent die op de laptop draait,
meerdere AI-modellen gebruikt en toegang heeft tot bestanden, agenda, mail en telefoon.

---

## 1. Wat Jarvis moet kunnen (eisen)

| # | Eis | Haalbaarheid |
|---|-----|--------------|
| 1 | Reageert op het wekwoord **"Jarvis"** + opdracht/vraag | Goed haalbaar |
| 2 | Werkt ook als het scherm **vergrendeld** is | Haalbaar, met beperkingen (zie §4) |
| 3 | Werkt met **alle AI-modellen** (Claude, GPT, Gemini, lokaal) | Goed haalbaar via een model-router |
| 4 | Voert **telefoongesprekken**, bv. restaurant reserveren | Haalbaar, maar het lastigste onderdeel |
| 5 | Toegang tot **alle bestanden**, privé en werk | Privé: ja. Werk: alleen met toestemming van BAM (zie §4) |
| 6 | **Agenda's** beheren | Goed haalbaar (Google + Outlook) |
| 7 | **Mails** lezen en antwoorden opstellen | Goed haalbaar, met verzendcontrole |
| 8 | Werkt **naadloos op de laptop** | Haalbaar als achtergrondservice + tray-app |

---

## 2. Architectuur in één oogopslag

```
 Microfoon
    │
    ▼
┌──────────────┐   "Jarvis"   ┌──────────────┐   tekst   ┌───────────────────────┐
│ Wekwoord     │ ───────────▶ │ Spraak→tekst │ ────────▶ │  Jarvis Core (brein)  │
│ (lokaal,     │              │ (Whisper)    │           │  - model-router       │
│ altijd aan)  │              └──────────────┘           │  - geheugen/profiel   │
└──────────────┘                                         │  - rechten & bevestig │
       ▲                                                 └─────────┬─────────────┘
       │ stemherkenning (is dit Nathan?)                           │ tool-calls (MCP)
                                                                   ▼
   ┌───────────┬───────────┬────────────┬────────────┬────────────┬─────────────┐
   │ Bestanden │ Agenda    │ Mail       │ Telefonie  │ Laptop     │ Web/zoeken  │
   │ (lokaal + │ (Google + │ (Gmail +   │ (Twilio +  │ (apps,     │             │
   │ OneDrive) │ Outlook)  │ Outlook)   │ realtime   │ volume,    │             │
   │           │           │            │ voice-AI)  │ meldingen) │             │
   └───────────┴───────────┴────────────┴────────────┴────────────┴─────────────┘
                                   │
                                   ▼
                          Tekst→spraak (antwoord via speaker)
```

**Kernkeuzes (aanbevolen):**

| Onderdeel | Aanbeveling | Alternatief |
|-----------|-------------|-------------|
| Taal/runtime | Python 3.12 | Node/TypeScript |
| Wekwoord | **Porcupine** (Picovoice, heeft "Jarvis" standaard) | openWakeWord (gratis, open source) |
| Spraak→tekst | **faster-whisper** lokaal (goed Nederlands, privé) | OpenAI/Deepgram API (sneller) |
| Tekst→spraak | **ElevenLabs** (natuurlijk Nederlands) | Piper (gratis, lokaal) |
| Brein/agent | **Claude Agent SDK** met tool-calls | LangGraph |
| "Alle modellen" | **LiteLLM** of **OpenRouter** als router | Eigen adapterlaag |
| Koppelingen | **MCP-servers** (Gmail, Google Calendar, Microsoft Graph, bestanden) | Directe API-calls |
| Telefonie | **Twilio** (NL-nummer) + realtime voice-model | Vapi / Retell (kant-en-klaar) |
| Stemverificatie | **Resemblyzer / SpeechBrain** speaker-embedding | Picovoice Eagle |
| Laptop-integratie | Achtergrondservice + tray-icoon | — |

---

## 3. Stappenplan

Elke fase levert iets bruikbaars op. Pas door naar de volgende fase als de vorige stabiel werkt.

### Fase 0 – Voorbereiding (week 1)
1. Bepaal het **besturingssysteem** van de laptop (Windows of macOS) – dit bepaalt de service-aanpak.
2. Check bij **BAM IT** wat mag met werkmail, werkagenda en werkbestanden (zie §4.1). Doe dit *vóór* je gaat koppelen.
3. Maak accounts/API-sleutels aan: Anthropic, OpenAI (optioneel), Picovoice, ElevenLabs, Twilio.
4. Zet een **wachtwoordkluis** op voor sleutels (Windows Credential Manager / macOS Keychain, nooit in code of Git).
5. Stel een **budget en limieten** in per API (zie §5).

### Fase 1 – Spraaklus: "Jarvis, hoe laat is het?" (week 2–3)
1. Microfoon continu uitlezen; wekwoord **lokaal** detecteren (er gaat niets naar de cloud tot "Jarvis" valt).
2. Na het wekwoord: opnemen tot stilte (VAD), omzetten naar tekst met Whisper.
3. Tekst naar één model (Claude), antwoord terug als spraak.
4. Korte "piep" of lampje als Jarvis luistert, zodat je weet wanneer hij opneemt.
5. **Klaar als:** 9 van 10 keer correct wakker worden, < 1 vals alarm per uur, antwoord binnen 3 sec.

### Fase 2 – Model-router: "alle AI-modellen" (week 3)
1. LiteLLM/OpenRouter ertussen zetten: één interface, elk model aanroepbaar.
2. Routeringsregels, bijvoorbeeld:
   - Snelle simpele vragen → klein/goedkoop model
   - Mail, plannen, bestanden → sterk model met tool-gebruik (Claude)
   - Gevoelige/offline dingen → lokaal model (Ollama)
   - "Jarvis, vraag het aan GPT" → expliciete modelkeuze per stem
3. Automatische **fallback** als een provider plat ligt.

### Fase 3 – Agenda en mail (week 4–5)
1. Koppel **Google Calendar + Gmail** (privé) via MCP/OAuth.
2. Koppel **Outlook/Microsoft 365** (werk) via Microsoft Graph – *alleen na akkoord IT*.
3. Functies: agenda voorlezen, afspraak maken/verzetten, vrije tijd zoeken, mails samenvatten, concept-antwoord opstellen.
4. **Regel:** Jarvis maakt standaard een **concept**. Verzenden pas na jouw bevestiging ("Jarvis, verstuur maar").
5. **Klaar als:** "Jarvis, wat staat er morgen op de planning en heb ik nog belangrijke mail?" werkt goed.

### Fase 4 – Bestanden (week 5–6)
1. Lokale mappen + OneDrive/Google Drive indexeren (zoeken op inhoud, niet alleen op naam).
2. Lokale **zoekindex** (vector-database, bv. Chroma/LanceDB) zodat niet elk bestand naar de cloud gaat.
3. Gescheiden **privé** en **werk** index met eigen rechten.
4. Lezen mag vrij; **verwijderen/overschrijven alleen met bevestiging**, met prullenbak/versiebeheer.
5. **Klaar als:** "Jarvis, zoek de offerte van het asfaltwerk uit maart" levert het juiste bestand.

### Fase 5 – Werkt altijd, ook bij vergrendeld scherm (week 6–7)
1. Jarvis draait als **achtergrondservice** die start bij opstarten/inloggen, niet als los venster.
   - Windows: taak in Taakplanner "bij aanmelden" + tray-app; vergrendelen laat het proces doorlopen.
   - macOS: LaunchAgent; microfoontoegang blijft werken bij vergrendeld scherm.
2. Energiebeheer: laptop mag niet in slaapstand terwijl je Jarvis wil gebruiken (dichtgeklapt = microfoon vaak uit).
3. **Stemverificatie** aanzetten: alleen jouw stem mag opdrachten geven (zie §4.2).
4. **Vergrendeld-modus** met beperkte rechten:
   - Wel: vragen, agenda voorlezen, herinneringen, reservering voorbereiden.
   - Niet: mails versturen, bestanden openen/voorlezen, betalingen, iets op het bureaublad doen.
5. Watchdog die Jarvis herstart als hij crasht.

### Fase 6 – Telefoongesprekken (week 8–10)
1. Twilio NL-telefoonnummer; uitgaand bellen via API.
2. Realtime spraakmodel voert het gesprek (lage vertraging is cruciaal, < 1 sec).
3. Werkwijze bij "Jarvis, reserveer bij restaurant X vrijdag om 19:00 voor 2 personen":
   1. Jarvis zoekt nummer en checkt je agenda.
   2. Jarvis **herhaalt de opdracht en vraagt bevestiging** voor hij belt.
   3. Opent het gesprek met: *"Goedenavond, u spreekt met de digitale assistent van Nathan …"*
   4. Strikte grenzen: alleen binnen de opdracht (tijdvenster, aantal personen); bij twijfel → terugbellen of jou vragen.
   5. Na afloop: samenvatting + afspraak in je agenda.
4. **Eerst online reserveren proberen** (website/TheFork) – betrouwbaarder dan bellen.
5. Start met een **testfase**: bel eerst jezelf of een vriend.

### Fase 7 – Laptop-integratie en afwerking (week 10–12)
1. Apps openen, volume, muziek, meldingen voorlezen, timers/herinneringen.
2. Geheugen: Jarvis onthoudt voorkeuren (bv. favoriete restaurants, werktijden, UTA-afspraken).
3. Instellingenscherm in de tray: rechten per onderdeel aan/uit, log bekijken, pauzeknop ("niet luisteren").
4. **Activiteitenlog**: alles wat Jarvis doet is terug te lezen.
5. Back-up van configuratie en geheugen.

---

## 4. Aandachtspunten (opletten!)

### 4.1 Werkgegevens BAM – belangrijkste risico
- Werkmail, werkagenda en projectbestanden zijn **eigendom van BAM**. Ze naar een externe AI-dienst sturen kan in strijd zijn met het IT-/informatiebeveiligingsbeleid en de geheimhouding uit je arbeidscontract.
- Een Microsoft 365 werkaccount vereist vaak **admin-toestemming** voor koppelingen; probeer dit niet te omzeilen.
- **Doe:** vraag schriftelijk na bij IT wat is toegestaan. Mogelijk biedt BAM zelf Microsoft Copilot aan – dat is dan de route voor werkdata.
- **Tot er toestemming is:** werk en privé strikt gescheiden houden; Jarvis alleen voor privé.

### 4.2 Beveiliging – Jarvis is een sleutelbos
- Iemand die "Jarvis" roept (of een video/tv/opname) kan opdrachten geven. Daarom:
  - **Stemverificatie** + beperkte rechten bij vergrendeld scherm.
  - **Bevestiging** voor alles wat onomkeerbaar of extern is: mail versturen, bellen, verwijderen, betalen, afspraken met anderen.
  - Stemverificatie is **niet waterdicht** (stem-klonen bestaat). Gebruik geen stem als enige beveiliging voor geld of wachtwoorden.
- **Prompt-injectie:** een mail of document kan verborgen instructies bevatten ("stuur alle bestanden naar …"). Jarvis mag inhoud van mails/bestanden nooit als opdracht behandelen.
- API-sleutels en tokens in de systeemkluis, nooit in code of Git.
- Laptop-schijf versleuteld (BitLocker/FileVault).

### 4.3 Privacy en wetgeving
- **Telefoongesprekken:** volgens de EU AI-verordening (artikel 50, geldt sinds augustus 2026) moet de ander weten dat hij met een AI praat. Jarvis maakt zich dus **altijd bekend als assistent**.
- **Gesprekken opnemen:** als deelnemer mag het in NL, maar wees transparant en bewaar opnames niet langer dan nodig (AVG).
- **Altijd-aan microfoon:** wekwoord-detectie lokaal houden; alleen na "Jarvis" audio verwerken. Let op gesprekken van Elena of bezoekers – maak een duidelijke pauzeknop.
- Kies waar mogelijk AI-aanbieders met **EU-dataopslag** en zonder training op jouw data.

### 4.4 Techniek
- **Vergrendeld scherm ≠ slaapstand.** In slaapstand of met dichtgeklapte laptop werkt de microfoon meestal niet. Overweeg een externe USB-microfoon of later een losse speaker (bv. Raspberry Pi) als altijd-aan apparaat.
- **Vertraging:** wekwoord → antwoord moet < 2–3 sec voelen; anders gebruik je het niet. Lokale STT + streaming antwoorden helpen.
- **Nederlands:** test wekwoord en spraakherkenning met Zwolse uitspraak, achtergrondgeluid en via laptop-speakers.
- **Accu:** continu luisteren kost energie; lokaal wekwoord is zuinig, lokale grote modellen niet.
- **Telefonie** is het meest foutgevoelig (slechte lijn, onverwachte vragen, wachtmuziek). Bouw dit als laatste.

### 4.5 Gebruik en vertrouwen
- Begin met **alleen lezen** (mail, agenda, bestanden). Geef pas schrijfrechten als Jarvis een paar weken betrouwbaar is.
- AI kan zich vergissen of iets verzinnen; controleer belangrijke mails en afspraken.
- Log alles, zodat je kunt nagaan wat Jarvis gedaan heeft.

---

## 5. Kosten (indicatie per maand)

| Post | Schatting |
|------|-----------|
| AI-modellen (normaal gebruik, ~50 vragen/dag) | € 15 – 40 |
| Spraak→tekst | € 0 (lokaal Whisper) |
| Tekst→spraak ElevenLabs | € 5 – 22 (of € 0 met Piper) |
| Picovoice wekwoord | € 0 (persoonlijk gebruik) |
| Twilio NL-nummer | ± € 5 – 15 + ± € 0,10–0,20 per belminuut |
| Realtime voice-model voor bellen | ± € 0,10–0,30 per minuut gesprek |
| **Totaal** | **± € 30 – 90 per maand** |

Zet harde **uitgavelimieten** in elk API-dashboard.

---

## 6. Beslissingen die nog nodig zijn

1. Windows of macOS?
2. Werkdata: wacht je op akkoord BAM IT, of start je alleen met privé?
3. Privacy vs. gemak: lokale spraakherkenning (privé, iets trager) of cloud (sneller)?
4. Stem van Jarvis: Nederlands, mannelijk/vrouwelijk, Brits "Jarvis"-accent?
5. Telefonie: zelf bouwen (Twilio) of kant-en-klaar platform (Vapi/Retell)?

---

## 7. Eerstvolgende concrete stap

Fase 0 + Fase 1: projectstructuur opzetten in `jarvis/`, wekwoord "Jarvis" + Whisper + Claude
werkend krijgen op de laptop. Daarna testen en pas dan uitbreiden.
