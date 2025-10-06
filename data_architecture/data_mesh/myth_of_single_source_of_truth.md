## 🧩 1️⃣ Source of Truth

**Definition:**
The **system or domain that generates and owns the authoritative version** of a specific data entity or attribute — the *origin of business facts*.

**In your example:**

* The *Facilities Management* system is the **source of truth** for *physical attributes* (area, GPS coordinates, layout).
* The *Finance/Administration* system is the **source of truth** for *legal and ownership details* (lease, cost center).
* The *Access Control* system is the **source of truth** for *security permissions and logs*.

Each of these systems is the **authoritative producer** of its respective type of “site” data.

✅ **Key point:**
There’s no single, universal “Site” source of truth — rather, each *aspect* of the Site has its own authoritative source domain.

---

## 🪞 2️⃣ Multiple Valid Truths

**Definition:**
Different domains have **legitimate, context-specific representations** of the same real-world entity — each *truthful within its own purpose*.

**Example:**

* The **Operations** domain sees “Site A” as *active* because production is ongoing.
* The **Finance** domain sees “Site A” as *inactive* because the lease expired last week.
* The **Security** domain sees “Site A” as *restricted* because access control is disabled for renovation.

All three statements are true — just from *different perspectives*.

✅ **Key point:**
Data Mesh **embraces multiple truths** — as long as each truth is well-defined, owned, and discoverable.
The goal isn’t to *eliminate* multiple truths, but to make them *consistent within context* and *traceable to their sources*.

---

## 📦 3️⃣ Data Copy (Conscious, Consistent Replication)

**Definition:**
A **deliberate, controlled replication** of data from a source domain to another domain or storage system — for analytical, performance, or integration reasons.

**Example:**

* The *Operations Analytics* team copies *site layout data* (from the Facilities domain) into their analytical store to calculate movement efficiency.
* The copy is **sourced from the authoritative system**, with lineage, timestamps, and governance intact.

✅ **Key point:**
A **data copy** is *not duplication* if:

* It’s sourced from the right domain,
* Versioned and timestamped,
* Kept consistent through sync or refresh policies, and
* Clearly attributed to the original source of truth.

This is **normal and necessary** in distributed data systems and Data Mesh — analytics systems often need local or transformed copies.

---

## ❌ 4️⃣ Duplicated or Inconsistent Data

**Definition:**
When **multiple uncontrolled copies** of the same data exist — often:

* Out-of-date,
* Unsynchronized,
* Lacking lineage or ownership, and
* Used as if they were the truth.

**Example:**

* Different teams maintain their own “Site Master” spreadsheets or tables.
* One lists *Site A* as “Active,” another as “Closed.”
* No one knows which is correct or when it was last refreshed.

❌ **This is duplicated/inconsistent data** — the root cause of mistrust and rework in analytics.

---

## 🔁 Summary — All Four Together

| Concept                            | Definition                                    | “Building Site” Example                                         | Acceptable in Data Mesh? |
| ---------------------------------- | --------------------------------------------- | --------------------------------------------------------------- | ------------------------ |
| **Source of Truth**                | The authoritative system for specific facts   | Facilities system owns physical site data                       | ✅ Yes                    |
| **Multiple Valid Truths**          | Different domains’ contextual truths          | Finance: leased site; Operations: active site                   | ✅ Yes                    |
| **Data Copy**                      | Controlled replication with lineage           | Analytics team copies Facilities data for performance dashboard | ✅ Yes                    |
| **Duplicated / Inconsistent Data** | Uncontrolled, outdated, or conflicting copies | Multiple spreadsheets with conflicting “Site A” statuses        | ❌ No                     |

---

### 🧠 Takeaway

* **Data Mesh doesn’t eliminate copies or multiple truths** — it **manages them intentionally**.
* It ensures every data element has:

  * A *clearly defined source of truth*,
  * *Well-governed data copies*,
  * *Recognized context-specific versions (multiple valid truths)*, and
  * *No shadow duplicates* causing inconsistency.
