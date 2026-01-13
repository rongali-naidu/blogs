# **Why We Should Use IANA Time Zones, Not Just Offsets**

## **What Is a Time Zone?**

Looking at a timestamp like `2023-01-14 01:23:34`, we cannot tell whether it represents **Seattle/USA** time or **London/UK** time.

Since each country or region follows its own local clock, we need a **globally accepted reference clock** to translate time from one location to another. This reference is called **UTC (Coordinated Universal Time)**.

Each local clock can also be given a name—these are called **time zone names**.

---

## **How Is a Time Zone Represented?**

Time zones can be represented in several ways:

1. **Long Names (IANA)**: These include geography/city names.

   * Example: `America/Los_Angeles`, `Europe/London`, `Asia/Kolkata`

2. **Short Names / Abbreviations**:

   * Example: `PST` for Pacific Standard Time, `IST` for India Standard Time

3. **Offset from UTC**: Indicates how far the local time is ahead of or behind UTC.

   * Format: `+/-HH:MM`
   * Example: PST offset is `-08:00` (standard) or `-07:00` (daylight, during DST)

4. **DST (Daylight Saving Time)**: Some countries adjust clocks seasonally.

   * Example: `America/Los_Angeles` is `-8:00` in winter (PST) and `-7:00` in summer (PDT).

---

## **Why Timezone Representation Is Tricky**

Although [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) is widely recommended for storing timestamps, its **timezone support is limited**. Handling timezones correctly is challenging, especially when working with **data across multiple countries with different DST rules**.

Common approaches:

1. **ISO 8601 offsets**: `-08:00`

   * Provides a fixed UTC offset but does **not include regional context or DST rules**.

2. **Abbreviations in databases**: `PST` / `PDT`

   * Some databases accept these, but they are **ambiguous**. Multiple regions may share the same abbreviation with different DST rules.
   * PST/PDT distinction:

     * PT = Pacific Time
     * S = Standard (PST) → early November to mid-March
     * D = Daylight (PDT) → mid-March to early November

3. **Vendor-specific names**: `Pacific Standard Time` (SQL Server)

   * Works within that platform but may **not be portable** or fully DST-aware across other systems.

4. **IANA timezone names**: `America/Los_Angeles`

   * Standardized, region-based identifiers that include **DST rules, historical changes, and future transitions**, making conversions **unambiguous and reliable**.

---

## **IANA Time Zones: Accurate, Predictive, and Unambiguous**

The **IANA Time Zone Database (tzdata, also known as Olson database)** is the global standard for representing time zones in software.

* **Rule-Based**: Stores exact DST start/end rules, historical changes, and offsets for named regions like `"America/Los_Angeles"`.
* **Predictive**: Libraries and databases that rely on IANA data can calculate **future times** accurately, applying DST automatically.
* **Unambiguous**: Each IANA name uniquely identifies a location, avoiding the ambiguities of offsets or abbreviations.

**Example:**

| IANA Zone           | Offset Jan | Offset Jul | Notes                          |
| ------------------- | ---------- | ---------- | ------------------------------ |
| America/Los_Angeles | UTC−8      | UTC−7      | Automatically switches PST/PDT |
| America/Phoenix     | UTC−7      | UTC−7      | No DST, offset stays constant  |

You can check the [list of IANA time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) or explore the [official IANA tzdata portal](https://www.iana.org/time-zones) for historical and future timezone information.

**Key point:** Unlike static offsets or abbreviations, IANA zones are **future-proof**, accounting for new laws, DST changes, and regional adjustments.

---

## **How Databases Handle Timezones (with SQL Examples)**

| Database        | Storage                              | Conversion Example                                                                   | DST                                               |
| --------------- | ------------------------------------ | ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| PostgreSQL      | `TIMESTAMPTZ` stores UTC internally  | `SELECT timestamp_utc AT TIME ZONE 'America/Los_Angeles' AS local_time FROM events;` | ✅ DST handled automatically                       |
| Redshift        | `TIMESTAMPTZ` stores UTC internally  | `SELECT timestamp_utc AT TIME ZONE 'America/Los_Angeles' AS local_time FROM events;` | ✅ Works, but abbreviations like PST/PDT are risky |
| Oracle          | `TIMESTAMP WITH TIME ZONE`           | `FROM_TZ(ts, 'America/Los_Angeles')`                                                 | ✅ DST applied automatically                       |
| MySQL / MariaDB | Uses tz tables; IANA names supported | `CONVERT_TZ(timestamp_utc, 'UTC', 'America/Los_Angeles')`                            | ✅ Must keep tz tables updated                     |

---

## **How Programming Languages Handle Timezones**

| Language                | Library/Type                                           | Conversion Method       | DST |
| ----------------------- | ------------------------------------------------------ | ----------------------- | --- |
| Python                  | `zoneinfo` or `pytz` (IANA names)                      | UTC ↔ local conversion  | ✅   |
| Java / Scala            | `ZonedDateTime` + `ZoneId`                             | IANA tzdb internally    | ✅   |
| TypeScript / JavaScript | `dayjs.tz()`, `moment-timezone`, `Intl.DateTimeFormat` | IANA rules              | ✅   |
| C#/.NET                 | `DateTimeOffset` + `TimeZoneInfo`                      | IANA or Windows mapping | ✅   |

**Pattern:** Modern systems rely on **UTC internally + IANA rules**, ensuring **DST correctness** and accurate historical/future conversions.
