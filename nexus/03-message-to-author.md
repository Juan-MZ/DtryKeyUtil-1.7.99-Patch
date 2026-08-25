# Courtesy message to Dtry (optional — not a blocker)

**You do not need to send this before publishing.** Dtry's Modification permission already allows
releasing fixes *"without permission from or credit to me"*, so nothing is gated on a reply. This is
a heads-up and an offer, not a request.

Send it as a Nexus DM, or as a comment on the mod page if DMs are closed. It's short on purpose —
mod authors get a lot of messages, and this one is only asking for their attention, not their time.

Replace `[your name]` before sending.

---

**Subject:** 1.7.99 compatibility patch for DtryKeyUtil — yours if you want it

Hi Dtry,

Quick heads-up rather than a request. The 20 August 2026 Skyrim update (1.7.99) broke DtryKeyUtil,
so I put together a compatibility patch and published it on Nexus. Your permissions allow releasing
fixes without asking, so I didn't — but it felt wrong not to tell you.

Source and diff: **https://github.com/Juan-MZ/DtryKeyUtil-1.7.99-Patch**

It's only the recompiled DLL — no features touched, still reports itself as 0.0.2. What actually
needed fixing:

- **Address Library format 5.** 1.7.99 changed the on-disk format, and the released DLL was built
  against a CommonLibSSE-NG that only reads formats 1 and 2. Rebuilt against 6.7.0.
- **A logging bug in `utils.h`.** `logger::info()` was being passed a value read from the INI as its
  *format* string. Newer fmt rejects it at compile time, and it was a real bug either way — an INI
  value with braces in it would have been parsed as a format specifier. Now `logger::info("{}", …)`.
- **The plugin declaration.** CommonLib's `add_commonlibsse_plugin()` helper can't set the new
  Address Library v5 flag — its `StructCompatibility` field sits on the same offset as
  `versionIndependenceEx`, so it only ever writes 0 or 1. Declared by hand with
  `PluginVersionData` instead. `PluginDeclaration::GetSingleton()` still works, since it just reads
  the export and the layouts agree on the fields you use.

I also checked the vtable hook against a real 1.7.99 binary: `VTABLE_MovementHandler` still
resolves, and slots `0x2` / `0x4` still line up with `ProcessThumbstick` / `ProcessButton`.

**The diff is yours — take any of it, no conditions, no credit needed.** If you'd rather push the
fix yourself, that's the better outcome and I'll pull my patch the same day. Same if you'd just
prefer it gone, no questions.

I want nothing out of it. Thanks for the plugin — plenty of mods lean on it.

[your name]

---

## Spanish gloss (so you know what you're sending)

1. Aviso, no petición: 1.7.99 rompió el mod y ya publicaste un parche.
2. Aclara que sus permisos lo autorizaban sin preguntar, pero te pareció correcto avisarle.
3. Le pasa el repo.
4. Resume los tres arreglos reales — incluido el bug del INI, que es un fallo suyo de verdad y le
   sirve aunque no use tu parche.
5. Menciona la verificación del hook de vtable contra el binario real de 1.7.99.
6. Le ofrece el código sin condiciones y se compromete a retirarlo si actualiza o si lo pide.

## If they reply

- **They update the mod:** pull the patch, and add a line to the description pointing people at
  their release.
- **They ask you to take it down:** take it down, and the GitHub repo too if they ask.
- **They merge the diff:** great — retire the patch and say so on the page.
