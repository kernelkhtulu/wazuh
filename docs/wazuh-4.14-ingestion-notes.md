# Wazuh 4.14 Ingestion Mechanics

This document summarizes how decoders and rules should be packaged and deployed for Wazuh 4.14 as part of the "Full Samba Compliance" logging solution.  The guidance here is based on the current Wazuh 4.14 branch and should be used to inform decoder and rule development in subsequent phases.

## Decoder packaging and deployment

* **File locations**: Custom decoders should be placed in a dedicated directory (e.g. `decoders/`) at the root of this repository.  When deploying to a Wazuh manager, the resulting XML files must be copied into the manager’s `etc/decoders.d` directory.  Keep each logical decoder set in its own file; for example, all Samba‑related decoders live in `samba_decoders.xml`.
* **Prematch patterns**: Define a tight `<prematch>` for each decoder to reduce performance overhead.  Use static strings (e.g. program names like `smbd` or JSON keys) to quickly discard non‑matching logs.
* **JSON vs plaintext**: Where Samba emits structured JSON audit logs, leverage Wazuh’s `<decoder_parent>json</decoder_parent>` capability to parse JSON fields directly.  For legacy plaintext logs, build regex patterns that capture key attributes (username, source IP, status).  All decoders should populate a consistent set of fields (`data.user`, `data.src_ip`, `data.action`, `data.status`, etc.) where possible.
* **Robustness**: Decoders must tolerate missing or extra fields.  Optional fields should be defined with non‑greedy regex captures, and decoders should avoid failing if a log line is truncated or contains unexpected whitespace.

## Rule layering strategy

Wazuh evaluates rules in numeric order.  To produce maintainable content, organise rules into layers:

1. **Base rules**: These fire on simple event types and assign a low severity with generic groups such as `samba` and `audit`.  Base rules set a baseline for more specific rules to build upon.
2. **Enrichment rules**: These reference the `if_sid` of base rules and add context (e.g., map Samba error codes to `auth_failure` vs `access_denied`).  They may populate additional fields or override the `description` to be auditor‑friendly.
3. **Severity rules**: Higher‑level rules that aggregate multiple events or apply thresholds (e.g., multiple failed logons within a window) should reference earlier SIDs and set appropriate severities (`3=low`, `5=medium`, `7=high`).
4. **Compliance tags**: Final rules should include MITRE ATT&CK, NIS2, or ISO27001 tags where relevant.  Use Wazuh’s `<tag>` element to attach these.  For example, authentication failures map to `T1078` (Valid Accounts) and password changes to `T1098` (Account Manipulation).

Each rule file (e.g., `100-samba-auth.xml`, `110-samba-password.xml`) should include a comment header explaining its purpose and listing the decoder fields it relies on.

## Performance considerations

* **Avoid heavy regex**: Use simple patterns and anchored expressions where possible.  Avoid using `.*` greedily, as it can degrade performance.  Instead, match up to known delimiters (e.g., whitespace, commas).
* **Prematch ordering**: Place the most selective decoders and rules earlier in the file to minimise unnecessary processing.  For example, JSON logs can be matched quickly by looking for a starting `{` and a `"samba"` identifier.
* **Field normalisation**: Normalise usernames and IP addresses to lower case in decoders where appropriate.  This reduces the number of unique values and improves correlation.

## Rule deployment notes

* All custom rules should be stored under the `rules/` directory in this repository.  When deploying to a Wazuh manager, copy the XML files into `/var/ossec/etc/rules.d/` or include them via `local_rules.xml` with `<include>` tags.
* After adding or modifying decoders and rules, restart the Wazuh manager (`systemctl restart wazuh-manager`) to reload the configuration.  Use the `wazuh-logtest` tool to verify that sample logs match the expected decoders and trigger the correct rule IDs.

Generated on 2025-12-31.
