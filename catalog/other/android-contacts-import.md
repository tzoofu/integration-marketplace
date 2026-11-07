# Android contacts import (address book → app records, phone-based dedup)

- **category**: other
- **provider**: Android `ContactsContract` content provider (`READ_CONTACTS`)
- **reusable**: yes — one query helper plus three pure functions (normalize, distinct, filter-existing); only the target record type changes.
- **docs**: https://developer.android.com/identity/providers/contacts-provider/retrieve-names

Lets the user bulk-import people from the phone's address book into the app's own customer/contact list: read name, phone and email, collapse duplicates by normalized phone number, mark the ones that already exist, and create records for the picked ones.

## Playbook

### Prerequisites
- `<uses-permission android:name="android.permission.READ_CONTACTS"/>` and a runtime request.
- A phone normalizer that maps local and international formats to one key (e.g. `054-1234567` and `+972541234567` → the same string).
- For Play: a prominent in-app disclosure before the permission prompt, and a matching Data safety declaration.

### Setup steps
1. Show a short disclosure (what is read, that it stays in the app/account, nothing is uploaded elsewhere), then request `READ_CONTACTS` with `rememberLauncherForActivityResult(RequestPermission())`.
2. Query emails first into a `contactId → email` map, then phones sorted by display name, joining the email by contact id.
3. Run the pure dedup (drop entries without a usable number or name, `distinctBy` normalized key).
4. Show a multi-select list with search; disable/mark rows whose key matches an existing record.
5. On confirm, build records from the selection, again filtering keys already present (the data may have changed while the sheet was open), and write them in one batch.

### Core pattern
```kotlin
// Android side (IO)
suspend fun Context.readContacts(): List<ContactCandidate> = withContext(Dispatchers.IO) {
    val emails = HashMap<Long, String>()
    contentResolver.query(Email.CONTENT_URI, arrayOf(Email.CONTACT_ID, Email.ADDRESS), null, null, null)?.use { c ->
        while (c.moveToNext()) c.getString(1)?.takeIf { it.isNotBlank() }?.let { emails.putIfAbsent(c.getLong(0), it) }
    }
    val out = ArrayList<ContactCandidate>()
    contentResolver.query(Phone.CONTENT_URI, arrayOf(Phone.DISPLAY_NAME, Phone.NUMBER, Phone.CONTACT_ID), null, null,
        Phone.DISPLAY_NAME + " COLLATE LOCALIZED ASC")?.use { c ->
        while (c.moveToNext()) out += ContactCandidate(c.getString(0).orEmpty(), c.getString(1).orEmpty(), emails[c.getLong(2)].orEmpty())
    }
    distinctCandidates(out)
}

// Pure, unit-testable side
data class ContactCandidate(val name: String, val phone: String, val email: String = "") {
    val key: String get() = normalizePhone(phone)          // "" when not a usable number
}
fun distinctCandidates(list: List<ContactCandidate>) =
    list.filter { it.key.isNotEmpty() && it.name.isNotBlank() }.distinctBy { it.key }
fun existingKeys(records: List<Person>) =
    records.flatMap { listOf(normalizePhone(it.phone), normalizePhone(it.phone2)) }.filter { it.isNotEmpty() }.toSet()
fun recordsFromContacts(selected: List<ContactCandidate>, existing: List<Person>): List<Person> {
    val taken = existingKeys(existing)
    return distinctCandidates(selected).filter { it.key !in taken }
        .map { Person(name = it.name.trim(), phone = it.phone.trim(), email = it.email.trim()) }
}
```

### Env vars
none.

### Gotchas
- **A contact has several rows in `Phone.CONTENT_URI`** (one per number) and the same number often appears under two contacts; dedup on the normalized number, not on contact id or raw string.
- Compare against **all** phone fields of existing records (primary and secondary), and re-check at save time, not only when the list is shown.
- Query on `Dispatchers.IO`; large address books take noticeable time. Show a loading state.
- `READ_CONTACTS` is a sensitive permission for Play review: a prominent disclosure screen before the system prompt, a privacy-policy section, and a Data safety entry ("contacts, collected, not shared, used for app functionality") are expected. Only request it when the user taps import, never at startup.
- Handle "denied forever": `shouldShowRequestPermissionRationale` false after a denial → offer a button to app settings instead of re-requesting.
- Keep the mapping logic pure (no Android imports) so dedup and normalization are covered by JVM unit tests.
- Names may be empty or emoji-only for business numbers; drop or label them rather than creating blank records.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace: a native Android app importing address-book entries as customers, with a disclosure dialog, a searchable multi-select sheet and normalization of local vs international phone formats.
