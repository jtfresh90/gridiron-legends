# GitHub Pages packaging note

This tree is the same modded Gridiron Legends build as the canonical copy
(Drive folder "Gridiron Legends" / local `modded/`), repackaged only for
publishing: the hosting upload channel rejects single requests over ~4 MB,
and `models/character.js` is 14.3 MB as one file.

So the character model's data is split across `models/character_p1.js` ..
`models/character_p4.js`, which `index.html` loads first; each appends its
chunk to `window.GLM["character"]`, reassembling the byte-identical string.
`models/character.js` itself is a tiny stub that satisfies the game's
loader, which then reads the reassembled string exactly as in the
canonical build. No game code or model data differs.
