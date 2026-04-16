# Modifiche da effettuare — Il Gigante delle Stelle

File: index.html

---

## 1. Spezzare bossAI() in strategie per-boss

**Dove:** righe 4122-5859 (738 righe)

**Come:**
- Creare oggetto `BOSS_STRATEGIES` con chiavi 1-10
- Ogni voce contiene funzioni: `movement(boss, frame)`, `attacks(boss, frame)`
- bossAI() diventa dispatch: `BOSS_STRATEGIES[boss.type].movement(boss, frame)` ecc.
- Spostare ogni blocco if/else del movimento (righe 4152-4386) nella rispettiva funzione
- Spostare ogni blocco if/else degli attacchi (righe 4388-5858) nella rispettiva funzione
- Mantenere la logica phase 2/3 (HP<60%, HP<30%) dentro ogni strategia

**Attento a:**
- Boss 10 (righe 5495-5858) combina attacchi di tutti i boss precedenti
- Gli attacchi usano setTimeout — verificare che i closure funzionino dopo lo spostamento
- I boss.type vanno da 1 a 10, verificare che il dispatch copra tutti

---

## 2. Aggiungere delta time al game loop

**Dove:** game loop riga 7204, update() riga ~5949, tutti i movimenti/timer

**Come:**
- Modificare `gameLoop()` per accettare timestamp da requestAnimationFrame:
  ```
  function gameLoop(timestamp) {
    const dt = Math.min(Math.max((timestamp - lastTimestamp) / 16.667, 0.5), 3.0);
    lastTimestamp = timestamp;
    update(dt);
    render();  // render resta frame-based
    requestAnimationFrame(gameLoop);
  }
  ```
- Moltiplicare tutte le velocità per dt:
  - `player.x += dx * speed * dt` (riga 5969)
  - `bullets[i].y += bullets[i].vy * dt` (riga 5997)
  - `b.x += b.vx * dt; b.y += b.vy * dt` (riga 6309)
  - `m.x += m.vx * dt; m.y += m.vy * dt` (riga 6331)
  - `p.x += p.vx * dt; p.y += p.vy * dt` (riga 6610)
  - Tutti i `boss.x/y` assignments nel movement
- Moltiplicare tutti i timer per dt:
  - `boss.attackTimer += dt` (era `++`)
  - `player.iFrames -= dt` (era `--`)
  - Tutti i `timer--` / `timer++` nei proiettili speciali (homingTimer, explodeTimer, zoneTimer, novaTimer, convergeTimer, pillarTimer, fadeTimer, ghostTimer, flickerTimer, implodeTimer, duplicateTimer, homingDelay)
  - I contatori `frame % N` rimangono frame-based (sono visivi, non fisici)

**Attento a:**
- I setTimeout degli attacchi boss sono in millisecondi reali, non hanno bisogno di dt
- I frame-based check (`frame % 3 === 0` per particelle) vanno tenuti come sono
- Clamare dt tra 0.5 e 3.0 per evitare salti improvvisi
- Verificare che le collisioni funzionino con dt > 1 (proiettili possono saltare oggetti)

---

## 3. Separare mutazione stato dal rendering

**Dove:** drawBossHead() righe 1185-2493, 5 punti di particles.push()

**Punti da spostare:**
1. Riga 1582 — Boss 5 (Nebulosa): particelle energia ogni 3 frame
2. Riga 1733 — Boss 6 (Arcano): particelle arcane ogni 4 frame
3. Riga 2023 — Boss 8 (Stella Morente): particelle decadimento ogni 5 frame
4. Riga 2220 — Boss 9 (Colosso): particelle polvere ogni 8 frame
5. Riga 2417 — Boss 10 (Entità Finale): particelle cosmiche ogni 3 frame

**Come:**
- Creare `updateBossParticles(boss)` chiamata da update() prima di render()
- Spostare i 5 blocchi particles.push() in questa funzione
- drawBossHead() non tocca più l'array particles
- Se drawBossHead è già diviso in funzioni per-boss (già fatto), le particelle vanno nella rispettiva funzione update di ogni boss

**Attento a:**
- Le particelle usano `boss.x` e `boss.y` — devono essere generate DOPO che il boss si è mosso ma PRIMA del render
- I check `frame % N` restano identici
- Variabili locali come `dmgRatio` nel boss 8 servono per decidere se generare particelle — servirà esporle o ricalcolarle

---

## Ordine consigliato

1. Separare stato da rendering (il più isolato, meno rischi)
2. Delta time (sistematico, tocca molto codice ma meccanico)
3. Spezzare bossAI (il più complesso, farlo per ultimo quando il resto è stabile)