# PresetKombi — Show only radio presets in the Macan cluster (MIB2 High)

Make the instrument cluster (Kombi) open **only your saved radio presets** instead of
the full station list — on **Porsche Macan PCM (MIB2 High / MHI2)**.

This is something the forums have long said is **impossible without a factory software
feature that never shipped**. It turns out it *is* possible, by patching one HMI class.

> **Status: working, with known issues (WIP).** Core works — the cluster shows presets.
> Still rough: autostore ("best signal") entries can leak in and selection can be jittery.
> See *Known issues*. Contributions welcome.

---

## What it does

On MIB2, the cluster and the steering‑wheel list normally open the **reception list**
(all received stations). The PCM already sends the cluster **two** lists — the reception
list *and* a preset list — plus a `PreferredList` BAP hint. The hint alone does **not**
change which list the cluster opens.

The real lever is in **`de.audi.tuner.app.combi.CombiBAPServiceHandler`**:

- `updatedActiveList()` / `setCombiService()` → `combiService.updatePreferredList(…)`
  (BAP enum `PreferredList`: `1 = reception list`, `2 = preset list`).
- `sendReceptionList(band)` → builds the list the cluster actually opens, from
  `getActiveTunerGuiHandler().getStationList()` (the full list).

The patch:
1. Force `updatePreferredList(2)` (prefer the preset list), **and**
2. Feed `sendReceptionList` from the **memory/preset list** instead of the full station
   list, filtered to the current band.

---

## How it works (method, not redistribution)

> ⚠️ The patched class is **derived from proprietary Porsche/Audi HMI firmware**. Do **not**
> redistribute the compiled `lsd.jxe`/class. This guide shares the **method** — you apply it
> to **your own** extracted firmware. Use at your own risk; you are modifying your own car.

### Requirements
- A MIB2 toolbox with root shell access (MHI2 "M.I.B." / SSH).
- [`jxe2jar`](https://github.com/luka-dev/jxe2jar) — converts the IBM J9 `lsd.jxe` → `.jar`.
- [CFR](https://www.benf.org/other/cfr/) decompiler.
- An **old JDK** (Java 8 works) to compile to **target 1.4 (classfile v48)** — newer
  class versions (v49+) are **rejected by the car's J9** and hang the tuner.

### Steps

1. **Get `lsd.jxe`** from your unit (`/ifs/lsd.jxe`) and convert it:
   ```
   python3 jxe2jar/src/jxe2jar.py lsd.jxe lsd.jar
   ```

2. **Decompile the target class** (needs the jar on the classpath for context):
   ```
   java -jar cfr.jar cls/de/audi/tuner/app/combi/CombiBAPServiceHandler.class \
        --extraclasspath lsd.jar --outputdir src
   ```

3. **Apply the changes** (conceptual diff — exact lines vary by firmware):

   ```java
   // (a) Always prefer the preset list — both call sites:
   //   this.combiService.updatePreferredList(this.models.getActiveList() == 12 ? 2 : 1);
   //   this.combiService.updatePreferredList(n == 12 ? 2 : 1);
   // becomes:
   this.combiService.updatePreferredList(2);

   // (b) In sendReceptionList(int n): feed the list from the PRESETS, filtered by band,
   //     with a safe fallback to the full station list.
   //   TunerObjectContainer[] arr = iTunerGUIHandler.getStationList();
   // becomes:
   TunerObjectContainer[] src = this.memory.getList();
   int want = -1;
   switch (n) { case 1: case 4: case 10: want = 3; break;   // AM/FM
                case 5: want = 6; break;                     // DAB
                case 7: want = 5; break;                     // SDARS
                case 11: want = 7; break; }                  // Uni
   java.util.ArrayList list = new java.util.ArrayList();
   if (src != null && want != -1)
       for (int i = 0; i < src.length; i++)
           if (src[i] != null && src[i].getType() == want) list.add(src[i]);
   while (list.size() > 15) list.remove(list.size() - 1);    // keep manual presets only
   TunerObjectContainer[] arr;
   if (list.size() > 0) arr = (TunerObjectContainer[]) list.toArray(new TunerObjectContainer[list.size()]);
   else                 arr = iTunerGUIHandler.getStationList();
   ```

   > **Why the band filter is required:** `createAMFMList()` assumes every entry is AMFM
   > (`getAMFMService().getUniqueId()`), so feeding mixed‑band presets throws an NPE and the
   > list disappears. Filter by band type first.

4. **Compile to v48 and jar it:**
   ```
   javac -source 1.4 -target 1.4 -cp lsd.jar -d out \
         src/de/audi/tuner/app/combi/CombiBAPServiceHandler.java
   (cd out && jar cf ../PresetKombi.jar de/audi/tuner/app/combi/CombiBAPServiceHandler*.class)
   ```

5. **Install (class override, no ROM edit):**
   - Copy `PresetKombi.jar` → `/mnt/app/eso/hmi/lsd/jars/`
     (the factory `lsd.sh` auto‑adds jars there to the bootclasspath).
   - To guarantee it overrides the ROM class, prepend it ahead of the `-jxe:` image.
     **Back up `lsd.sh` first**, then edit both `$J9 … -Xbootclasspath:$BOOTCLASSPATH …`
     launch lines:
     ```
     $J9 -Xbootclasspath/p:$BASE_DIR/lsd/jars/PresetKombi.jar $VMOPTIONS -Xbootclasspath:$BOOTCLASSPATH …
     ```
   - Reboot the head unit.

### Reverting
Restore the `lsd.sh` backup and delete `/mnt/app/eso/hmi/lsd/jars/PresetKombi.jar`, reboot.
`lsd.sh` is never otherwise touched, so recovery is clean.

---

## Gotchas learned the hard way

- **Classfile version matters.** The car's J9 accepts up to **v48**. A `jxe2jar`‑reconstructed
  **v49** class loads far enough to **hang the tuner** — recompile from source at `-target 1.4`.
- **Don't hex‑patch `lsd.jxe`.** It's a ZIP wrapping an XIP `rom.classes` image; editing bytes
  breaks the ZIP CRC and XIP alignment (brick territory). Use the class‑override method above.
- The cluster's `PreferredList` hint does **not** by itself change which list opens — you must
  change the **reception list source** (`sendReceptionList`).

## Known issues / TODO

- **Autostore leakage:** the memory list can include auto‑stored "best signal" stations after
  your manual presets; the `>15` cap is a crude trim. A proper manual‑vs‑autostore flag would be better.
- **Selection instability:** the list rebuilds on background station‑list updates; needs
  throttling and/or a cleaner ID mapping for the cluster selection.
- AM vs FM ordering when both have presets isn't fully handled.

PRs / ideas welcome.

## Credits
- [`jxe2jar`](https://github.com/luka-dev/jxe2jar) — J9 JXE → JAR.
- [`grajen3/mib2-lsd-patching`](https://github.com/grajen3/mib2-lsd-patching) — class‑override install method.
- The MIB2 community ([mib-helper.com](https://mib-helper.com)).

## Disclaimer
Modifying your head unit can brick it. This is provided as‑is, for your **own** vehicle and
**own** extracted firmware. Not affiliated with Porsche/Audi/VW.

---

### ☕ Support
If this helped you, you can support the work:
**Ko‑fi:** https://ko-fi.com/trd_10
