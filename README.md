# Unlockurr — Face ID hotel door lock

A touchscreen hotel door lock (Face ID + keypad) and a separate Lock Manager panel.
Face matching runs on the device with face-api.js. **Drex** makes the unlock decision.

```
public/lock.html      Hotel door + Unlockurr touchscreen lock (Face ID, PIN keypad, welcome animation)
public/manager.html   Lock Manager: guests, profile photos, 4–8 digit PINs, face enrollment, activity log
public/shared.js      Storage, PIN hashing, face models, avatars
server.py             Static server + POST /api/identify → Drex (key stays server-side)
```

## Run

```bash
# Put the key in .env (gitignored, chmod 600). It overrides any DREX_API_KEY in your shell.
echo 'DREX_API_KEY=nace_sk_...' > .env
python3 server.py              # http://127.0.0.1:8080
```

The key never reaches the browser; only `server.py` talks to Drex.

Open **/manager.html** and **/lock.html** side by side in the same browser. They share data through
`localStorage` and update live.

1. In the Lock Manager, click **+ Add guest**. Enter a first and last name and upload an optional photo.
2. Add one or more PINs (4–8 digits, unique per lock, stored as SHA-256 hashes).
3. Under Face ID, capture 3–5 samples with the camera, or upload a face photo.
4. **Save guest**. Then look at the lock's camera or tap **Use keypad**.

## How Drex decides

Drex only reads text, so images never leave the browser. Each unlock attempt works like this:

1. The lock computes 128‑d face embeddings for 4 consecutive camera frames.
2. It compares them with every enrolled sample and sends the server only numbers: mean, best and worst
   distance per guest, plus detection confidence.
3. `server.py` asks `drex-v2.0` two questions about that readout:
   - `identity` (`choice`): which guest it is, or `no_match`
   - `reliable` (`noul`): whether the capture is trustworthy enough to act on
4. The door unlocks only when **all** of these hold (constants at the top of `server.py`):
   - identity probability ≥ 0.80
   - reliable ≥ 0.70
   - the chosen guest's mean distance ≤ 0.55 (a hard limit enforced in code)

Face unlock **fails closed**. If Drex is unreachable, the key is missing or the answer is malformed, the
door stays locked and the screen shows "Face ID unavailable · use keypad". PIN entry always works.
After 5 wrong PINs the keypad locks for 30 s.

## Demo limitations

- Guest data lives in the browser's `localStorage`. In a real deployment, store enrolments and do the
  matching server-side, so a client can't fake a readout.
- Tune the thresholds on your own enrolment data before relying on them.
